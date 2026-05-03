# Day 3 — Punith (Agentic AI)

Notes, code, and experiments for Day 3.

Picking up from the [Day 3 topics](./README.md) — focusing on the retrieval and ranking pieces:

- [x] Embeddings 101 — what they are, picking a model (Titan, Cohere Embed, OpenAI)
- [x] Chunking strategies — fixed-size vs recursive vs semantic
- [x] Retrieval + reranking — top-k, hybrid (BM25 + vector), cross-encoder rerank

## Notes

# RAG Retrieval & Ranking — Complete Notes

A RAG system is only as good as the chunks it retrieves. The model can't reason about what it never sees, and it can't ignore garbage you stuff into the context window. These notes start from first principles — what an embedding *actually is* — and build up to a production-grade retrieval pipeline.

The mental model to hold throughout:

```
RAG  =  retrieval (find the right text)  +  generation (LLM writes the answer)

If retrieval is wrong, no model on earth can fix it.
If retrieval is right, even a small model can answer well.
```

So 80% of the work is in retrieval. That's what these notes are about.

---

# CONCEPT 1 — Embeddings 101

---

## The One-Sentence Intuition

> An embedding is **GPS coordinates for meaning**. Each piece of text gets mapped to a point in a high-dimensional space, and texts with similar meaning end up close together.

That's the whole idea. Everything else is mechanics.

---

## What an Embedding Actually Is

An embedding is a fixed-length vector of floats — usually 256, 1024, or 1536 numbers — that represents the *meaning* of a piece of text.

```python
embed("dog")   → [0.12, -0.44, 0.89, ...]   # 1024 floats
embed("puppy") → [0.14, -0.40, 0.91, ...]   # close to "dog"
embed("loan")  → [-0.71, 0.22, -0.03, ...]  # far from "dog"
```

You don't read the numbers — they're meaningless to humans. What matters is the **distance** between two vectors.

### How "close" is measured: cosine similarity

The standard metric is **cosine similarity** — the angle between two vectors, ignoring length.

```
cosine_sim = 1.0   →  same direction  →  same meaning
cosine_sim = 0.0   →  perpendicular   →  unrelated
cosine_sim = -1.0  →  opposite        →  opposite meaning (rare in practice)
```

```python
import numpy as np

def cosine(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

cosine(embed("dog"),   embed("puppy"))  # ~0.92  → very close
cosine(embed("dog"),   embed("canine")) # ~0.81  → close
cosine(embed("dog"),   embed("loan"))   # ~0.05  → unrelated
cosine(embed("dog"),   embed("not dog")) # ~0.65 → still pretty close (!)
```

> **Gotcha:** "dog" and "not dog" are *close* in embedding space, because embeddings encode topic more than logic. This is why pure vector search can't handle negation — you need BM25 alongside it. We'll come back to this in Concept 3.

### Vocabulary cheat sheet

| Term            | What it means                                                  |
|-----------------|----------------------------------------------------------------|
| Dimension       | Length of the vector (e.g. 1024, 1536, 3072)                   |
| Embedding model | Neural net that maps text → vector                             |
| Cosine sim      | 1 = identical meaning, 0 = unrelated, -1 = opposite            |
| Distance        | `1 - cosine_sim` (most vector DBs index distance, not sim)     |
| Normalize       | Scale vector to length 1 — makes dot product = cosine sim      |

> The vector is meaningless on its own. It only has value relative to other vectors produced by the **same model**. Mixing vectors from different models in one index is like mixing GPS coordinates with street addresses — the numbers are nonsense together.

---

## Why High-Dimensional? (the part nobody explains)

Why 1024 dimensions and not 3? Because language has way more than 3 axes of meaning.

Picture each dimension as a hidden "concept slider":

```
dim 12 →  "is it animal-like?"      (dog: high,   loan: low)
dim 47 →  "is it about money?"      (dog: low,    loan: high)
dim 88 →  "is it formal?"           (dog: low,    loan: medium)
... 1024 of these
```

The model isn't told what each dimension means — it learns them from billions of training examples. Most dimensions encode subtle gradients that don't have clean English labels. That's fine; you only ever use them to compute distance.

> You don't need to understand any individual dimension. You only need similar texts to land near each other.

---

## Picking a Model — Titan vs Cohere vs OpenAI

These are the three you'll see most often. They're not interchangeable — pick on the axis that matters for *your* workload.

| Model                          | Dim  | Max tokens | Languages | Notable trait                          |
|--------------------------------|------|------------|-----------|----------------------------------------|
| **Amazon Titan Text v2**       | 1024 / 512 / 256 | 8192 | 100+   | Native to Bedrock, IAM-bound           |
| **Cohere Embed v3 (English)**  | 1024 | 512        | English   | Best-in-class retrieval quality        |
| **Cohere Embed v3 Multilingual** | 1024 | 512      | 100+      | Strong cross-lingual retrieval         |
| **OpenAI text-embedding-3-small** | 1536 | 8191    | 100+      | Cheap, fast, good baseline             |
| **OpenAI text-embedding-3-large** | 3072 | 8191    | 100+      | Highest quality, highest cost          |

### Decision rule (the one you'll actually use)

```
On AWS, IAM-bound, no data egress     →  Titan v2
Best retrieval quality, English-only  →  Cohere Embed v3
Multilingual corpus                   →  Cohere v3 Multilingual or OpenAI 3-large
Cheap + fast baseline                 →  OpenAI 3-small
Strict data residency / Bedrock-only  →  Titan v2
```

### Things people get wrong

| Mistake                                          | Why it bites                                |
|--------------------------------------------------|---------------------------------------------|
| Mixing models in the same index                  | Vectors aren't comparable across models     |
| Choosing 3072-dim "because bigger is better"     | 4× storage + slower ANN, marginal quality   |
| Embedding 8000-token chunks with a 512-tok model | Silent truncation — half the doc is lost    |
| Re-embedding only new docs after switching model | Old vectors no longer comparable; rebuild   |

> Benchmark on **your data**, not on MTEB leaderboards. A model that crushes generic retrieval can flop on your domain (medical, legal, code). 30 hand-labeled queries beats every public leaderboard for *your* use case.

---

## Calling Each Provider

**Titan via Bedrock:**

```python
import boto3, json

bedrock = boto3.client("bedrock-runtime")

def titan_embed(text: str) -> list[float]:
    resp = bedrock.invoke_model(
        modelId="amazon.titan-embed-text-v2:0",
        body=json.dumps({
            "inputText": text,
            "dimensions": 1024,    # 256 / 512 / 1024
            "normalize": True,     # makes cosine sim = dot product
        }),
    )
    return json.loads(resp["body"].read())["embedding"]
```

**Cohere via Bedrock:**

```python
def cohere_embed(texts: list[str], input_type="search_document") -> list[list[float]]:
    resp = bedrock.invoke_model(
        modelId="cohere.embed-english-v3",
        body=json.dumps({"texts": texts, "input_type": input_type}),
    )
    return json.loads(resp["body"].read())["embeddings"]
```

**OpenAI:**

```python
from openai import OpenAI
client = OpenAI()

def openai_embed(texts: list[str]) -> list[list[float]]:
    resp = client.embeddings.create(model="text-embedding-3-small", input=texts)
    return [d.embedding for d in resp.data]
```

> **Cohere quirk worth memorizing:** `search_document` is for indexing, `search_query` is for retrieval. Using the wrong one tanks recall by ~10–20% with no error message. Titan and OpenAI don't have this knob — they treat queries and docs the same way.

---

## Symmetric vs Asymmetric Embeddings (and why it matters)

Two flavors of how a query and a document get encoded:

- **Symmetric** — query and document use the *same* encoding. Good for "find similar questions" or clustering.
- **Asymmetric** — query and document use *different* encodings (or different `input_type`). Better for retrieval, where queries are short ("what's the warranty?") and docs are long.

```
Symmetric  → embed("warranty info?") and embed("our 2-year warranty covers...") use same path
             Quality OK, but the query/doc shape mismatch hurts.

Asymmetric → query gets encoded as "this is a question"
             doc gets encoded as "this is a passage to be searched"
             Better recall on real RAG workloads.
```

| Model                | Mode         |
|----------------------|--------------|
| Cohere v3            | Asymmetric   |
| Titan v2             | Symmetric    |
| OpenAI 3-small/large | Symmetric    |

For pure RAG, asymmetric usually wins by a few points of recall. If you're using Cohere, **always pass the right `input_type`**. If you're using Titan or OpenAI, you don't need to think about it.

---

## Cost & Latency Reality Check

```
Embedding 1M tokens (rough as of late 2025):
  OpenAI 3-small      ~$0.02
  OpenAI 3-large      ~$0.13
  Titan v2            ~$0.02
  Cohere v3 (Bedrock) ~$0.10

Embedding latency per call:
  Single text         ~50–150 ms
  Batch of 96 texts   ~200–400 ms  (always batch when indexing)
```

Two paths in your system have very different cost shapes:

```
Indexing path  → runs once per doc (or per update). Latency doesn't matter much.
                 You can throw a slow, expensive model at it if quality is worth it.

Query path     → runs on every user question. Every ms is user-visible.
                 This is where you optimize.
```

> Indexing is a one-time cost; query embeddings are per-request. Don't waste days shaving milliseconds off the indexing pipeline — your users never see it.

---

# CONCEPT 2 — Chunking Strategies

---

## The One-Sentence Intuition

> Chunking is the art of slicing documents into pieces that are **small enough to be specific** and **large enough to be self-contained**. Get this wrong and no embedding model can save you.

---

## Why You Have to Chunk At All

Two reasons you can't just embed whole documents:

**1. Token limits.** Embedding models have a max input size — 512 tokens for Cohere, 8K for Titan and OpenAI. A 200-page PDF is hundreds of thousands of tokens. It doesn't fit.

**2. Vector smearing.** Even if it fit, embedding a 200-page doc as one vector averages every concept in it into mush. The vector ends up meaning "manual-ish" — close to every query and useful for none.

```
Bad chunk (too small):
  "It was discontinued in 2019."          ← what was discontinued?

Bad chunk (too big):
  [50 pages of product manual]            ← vector means "manual-ish"

Good chunk:
  "The Acme X1 router was discontinued in 2019.
   It was replaced by the X2 line, which uses
   the same firmware..."                   ← specific AND self-contained
```

The art is finding the middle.

---

## Strategy 1 — Fixed-Size Chunking

The simplest possible thing: split on a token or character count. This is the baseline everything else gets compared to.

```python
def fixed_chunks(text: str, size=512, overlap=64):
    chunks, i = [], 0
    while i < len(text):
        chunks.append(text[i : i + size])
        i += size - overlap
    return chunks
```

### Walkthrough — what overlap actually does

Imagine the answer to "when was the X1 discontinued?" lives at character position 510, right at a chunk boundary:

```
size=512, overlap=0:
  chunk 1: "...the Acme X1 router was disco"   ← ends here
  chunk 2: "ntinued in 2019. The X2 replaces..."
  → query "when was X1 discontinued?" matches neither chunk well

size=512, overlap=64:
  chunk 1: "...the Acme X1 router was disco"
  chunk 2: "...router was discontinued in 2019. The X2..."
  → query matches chunk 2 cleanly
```

| Pros                          | Cons                                              |
|-------------------------------|---------------------------------------------------|
| Trivial to implement          | Splits mid-sentence, mid-table, mid-code-block    |
| Predictable chunk count       | Loses logical structure                           |
| Works for any input           | Often the lowest retrieval quality                |

> Always include **overlap** (10–20% of chunk size). Without it, the answer to a query lives across two chunks and neither one matches well.

---

## Strategy 2 — Recursive Chunking

Smarter: split on a *hierarchy* of separators — paragraphs first, then sentences, then words — only descending when a chunk is still too big. This is what LangChain's `RecursiveCharacterTextSplitter` does.

The intuition: try to break at the biggest natural seam first. Only break at smaller seams if you have to.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=64,
    separators=["\n\n", "\n", ". ", " ", ""],
    #            ↑       ↑     ↑     ↑    ↑
    #         para   line  sent  word  char (last resort)
)
chunks = splitter.split_text(doc)
```

The separators list is the key. The default is reasonable for prose. For other content, swap it:

```python
# Code
separators=["\nclass ", "\ndef ", "\n\n", "\n", " "]

# Markdown — but prefer a Markdown-aware splitter that understands headings
separators=["\n## ", "\n### ", "\n\n", "\n", ". ", " "]
```

| Pros                                    | Cons                                       |
|-----------------------------------------|--------------------------------------------|
| Respects natural boundaries             | Tuning separators is per-corpus            |
| Good default for most prose             | Still chunk-size-driven, not meaning-driven|
| LangChain / LlamaIndex have it built-in | No semantic awareness                      |

> Recursive chunking is the right default for ~80% of real RAG projects. Start here. Only move to semantic chunking if your retrieval evals say you need to.

---

## Strategy 3 — Semantic Chunking

The smartest: split where the **meaning shifts**, not where the character count hits a limit.

The trick: embed each sentence, then look at how similar adjacent sentences are. Where similarity drops sharply, you've found a topic boundary.

```
sent embeddings:  s1  s2  s3  s4 │ s5  s6  s7
cosine sim drops here ───────────┘
                        (topic shift detected)
```

```python
def semantic_chunks(sentences, threshold=0.75):
    embs = embed(sentences)
    chunks, current = [], [sentences[0]]
    for i in range(1, len(sentences)):
        if cosine(embs[i-1], embs[i]) < threshold:
            chunks.append(" ".join(current))
            current = []
        current.append(sentences[i])
    if current:
        chunks.append(" ".join(current))
    return chunks
```

| Pros                                    | Cons                                            |
|-----------------------------------------|-------------------------------------------------|
| Chunks correspond to coherent ideas     | Embedding cost upfront                          |
| Best retrieval quality on dense prose   | Threshold needs tuning per corpus               |
| Handles unstructured text well          | Slow; not suitable for huge real-time ingestion |

> Semantic chunking shines on unstructured prose (blog posts, transcripts, interviews) and *hurts* on structured docs (tables, code, FAQs). For structured content, respect the structure first; semantic only helps for free-form text.

---

## Picking a Strategy — A Decision Table

| Content type                | Best strategy                                          |
|-----------------------------|--------------------------------------------------------|
| Generic prose / mixed docs  | Recursive (LangChain default)                          |
| Code / config files         | Language-aware splitter (AST or symbol-based)          |
| Markdown / structured docs  | Markdown header splitter + recursive within sections   |
| Long unstructured prose     | Semantic                                               |
| Tabular / row-oriented data | Don't chunk — index rows directly                      |
| FAQs / Q&A pairs            | One chunk per Q&A pair, never split a question         |

### Numbers that work as defaults

```
chunk_size      →  256–512 tokens
overlap         →  10–20% of chunk_size  (so 32–64 for a 256 chunk)
embed model max →  always check before tuning chunk_size
```

> If retrieval is bad, your **chunking** is almost always at fault before your embedding model is. Inspect the actual retrieved chunks before swapping models. 9 times out of 10 the answer was split across a chunk boundary.

---

## Metadata: the cheap quality multiplier

Every chunk should carry metadata you can filter or rerank on. This isn't optional — it's how you avoid retrieving stale or wrong-version content.

```python
{
    "text": "<chunk>",
    "embedding": [...],
    "doc_id": "manual-x1",
    "section": "Specifications",
    "page": 17,
    "updated_at": "2024-08-01",
    "tags": ["router", "x1"],
    "lang": "en",
}
```

This lets you do **hybrid filters at query time** — "search only the latest version of doc X, in the Specs section, in English." Pure vector search alone can't.

```python
# Pseudocode — filter before vector search
results = vector_db.search(
    query_embedding,
    filter={"doc_id": "manual-x1", "updated_at__gte": "2024-01-01"},
    top_k=10,
)
```

> Metadata filters are the difference between "the model answered with a 2019 fact" and "the model answered with current data." Cheap to add, expensive to live without.

---

# CONCEPT 3 — Retrieval & Reranking

---

## The One-Sentence Intuition

> Retrieval is a **funnel**: cast a wide net, then filter aggressively. Recall first (don't miss the right chunk), precision later (rank it to the top).

---

## Top-K Retrieval Basics

The flow at query time:

```
1. User asks a question
2. Embed the question with the same model used for indexing
3. Find the K nearest chunks in the vector index (ANN search)
4. Pass those K chunks to the LLM as context
5. LLM writes the answer using the chunks
```

Choosing K is a tradeoff between **missing the answer** and **drowning the model**:

| K     | Effect                                                           |
|-------|------------------------------------------------------------------|
| 1–3   | Tight context, fast, but misses if best chunk isn't actually #1  |
| 5–10  | Sweet spot for most apps — covers near-misses                    |
| 20+   | Hurts model quality (lost-in-the-middle), eats token budget      |

### "Lost in the middle" — the reason K isn't free

LLMs pay most attention to the **start** and **end** of the context window, and underweight the middle. Stuff 30 chunks in there and the answer in chunk #15 gets ignored even though it's literally there.

```
Attention to context across position:
  start  ████████████████
  ...    ██████
  middle ██               ← lost-in-the-middle
  ...    ██████
  end    ███████████
```

> Recall@K matters more than precision@1 *before* reranking — rerank can fix ordering, but it can't recover chunks you never retrieved. So: retrieve wide (K=50), rerank tight (top 5).

---

## ANN Indexes (HNSW, IVF) — how vector search is fast

Doing exact nearest-neighbor over millions of vectors is too slow for online use (every query would compare against every vector). Vector DBs use **Approximate Nearest Neighbor (ANN)** indexes — small accuracy tradeoff for huge speed gains.

The two you'll meet:

- **HNSW** (Hierarchical Navigable Small World) — graph-based; fast queries, high recall, more RAM. Default in OpenSearch, pgvector, Pinecone, Qdrant.
- **IVF** (Inverted File) — partition-based; lower memory, slightly worse recall. Good for billion-scale corpora.
- **Flat** — exact, brute force. Use only for < 100K vectors or for ground-truth comparisons.

### Tunables you'll touch

```
HNSW:  M (graph connectivity at build time)
       ef_construction (build-time effort, set once)
       ef_search (query-time effort — higher = better recall, slower)

IVF:   nlist (number of clusters at build time)
       nprobe (clusters scanned at query — higher = better recall, slower)
```

The pattern is the same for both: **at query time, you trade latency for recall**. Tune by measuring `recall@K` vs `query_latency` on a held-out set, and pick the point where recall plateaus.

> If you don't know which to pick, use HNSW. It's the default in every major vector DB for good reason.

---

## Why Pure Vector Search Isn't Enough

Vector search is great at semantic match but bad at:

- **Exact identifiers** — SKUs, error codes, version numbers (`X1-2019-A`, `ERR_403`)
- **Rare terms** — anything the embedding model didn't see enough of in pretraining (proprietary names, novel jargon)
- **Negation / boolean intent** — "docs without `deprecated`"

These are exactly what **BM25** (lexical / keyword search) is good at. BM25 is the same algorithm Elasticsearch and old-school search engines use — it scores documents by how often the query terms appear, weighted by how rare they are globally.

```
Query:  "ERR_403 in router X1"

Vector search   → returns chunks about "authentication errors", "router setups"
                  but might miss the chunk that literally contains "ERR_403"

BM25 search     → returns the chunk with "ERR_403" instantly, exact match
                  but misses chunks that say "permission denied error"
                  (which is semantically the same)

Hybrid          → gets both. Always wins in practice.
```

That's the whole reason hybrid search exists.

---

## Hybrid Search (BM25 + Vector)

Run both retrieval methods in parallel, then **fuse** the two ranked lists into one:

```
       ┌─→ BM25 retrieval ────→ top 50 by lexical score
query ─┤                                              ╲
       └─→ Vector retrieval ──→ top 50 by cosine sim   ──→ fuse → top 20 → rerank
```

### Reciprocal Rank Fusion (RRF)

Simple, hyperparameter-free, works embarrassingly well. The idea: score each doc by `1 / (k + its rank)`, summed across both lists. Docs that show up high in *either* list bubble to the top; docs that show up high in *both* bubble even higher.

```python
def rrf(lists, k=60):
    """lists: list of ranked result lists. each item is a doc_id."""
    scores = {}
    for ranked in lists:
        for rank, doc_id in enumerate(ranked):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: -x[1])

# Worked example
bm25_top  = ["A", "B", "C", "D"]
vector_top = ["B", "E", "A", "F"]

rrf([bm25_top, vector_top], k=60)
# → B is #2 in BM25 + #1 in vector → highest combined
# → A is #1 in BM25 + #3 in vector → close second
# → E, C, D, F follow
```

> RRF beats most weighted-score-blending schemes in practice because it's scale-free — it doesn't care that BM25 scores are 0–30 and cosine sims are 0–1. Start with RRF; only tune weights if you have a measured reason to.

OpenSearch supports hybrid natively via the `hybrid` query type. For pgvector, run BM25 (Postgres `tsvector`) and vector queries separately and fuse in app code.

---

## Cross-Encoder Reranking — the highest-ROI add-on

Embedding models are **bi-encoders** — they encode query and doc *independently*, then compare vectors. That's fast, but the comparison is approximate; the model never gets to look at the query and the doc together.

**Cross-encoders** take `(query, doc)` together as a single input and score them jointly. Much better quality, but you can't run them on millions of docs — you can only run them on a shortlist.

```
                    BI-ENCODER (embedding)
   query  ─→ encoder ─→ vec_q  ─┐
                                 ├─→ cosine  →  score
   doc    ─→ encoder ─→ vec_d  ─┘
   (fast, approximate, runs on millions of docs)


                    CROSS-ENCODER (reranker)
   (query, doc)  ─→ encoder  ─→ score
   (slow, accurate, runs only on shortlist of 20–100)
```

So the two-stage pattern emerges naturally:

```
Stage 1 — fast (bi-encoder + BM25):  shortlist of 50 candidates
Stage 2 — slow (cross-encoder):      rerank to top 5 final
```

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank(query, candidates, top_n=5):
    pairs = [(query, c["text"]) for c in candidates]
    scores = reranker.predict(pairs)
    ranked = sorted(zip(candidates, scores), key=lambda x: -x[1])
    return [c for c, _ in ranked[:top_n]]
```

| Reranker option                          | Where it runs       | Notable                          |
|------------------------------------------|---------------------|----------------------------------|
| Cohere Rerank v3 (via Bedrock or API)    | Managed             | State of the art, multilingual   |
| `cross-encoder/ms-marco-MiniLM-L-6-v2`   | CPU / small GPU     | Cheap, fast, English             |
| `BAAI/bge-reranker-large`                | GPU                 | Strong open-weight option        |
| LLM-as-reranker (Claude/GPT scoring)     | API call per doc    | Highest quality, highest cost    |

> Reranking 50 → 5 typically lifts answer quality more than swapping embedding models. It's the single highest-ROI thing to add once basic retrieval works. If you do nothing else from these notes, add a reranker.

---

## End-to-End Retrieval Pipeline

This is what a production-grade RAG pipeline looks like, in order:

```
user query
   │
   ▼
[ rewrite / expand ]   ← optional: HyDE, multi-query, query decomposition
   │
   ▼
┌──────────────────┬──────────────────┐
│ BM25 (top 50)    │ Vector (top 50)  │
└─────────┬────────┴────────┬─────────┘
          │                 │
          ▼                 ▼
       [ RRF fusion → top 20 ]
                  │
                  ▼
       [ cross-encoder rerank → top 5 ]
                  │
                  ▼
       [ optional metadata filter ]
                  │
                  ▼
            LLM answer
```

Each stage either improves recall (early stages) or precision (late stages). You can skip stages, but skip in this order:

```
rerank > hybrid > query rewrite

(ditch the fanciest thing first; never ditch chunking quality)
```

### Optional: query rewriting / HyDE

A quick mention because it sometimes helps a lot. The idea: real user queries are short and ambiguous ("billing issue?"), but documents are long and detailed. So **transform the query before retrieval** to make it look more like a document:

- **HyDE** (Hypothetical Document Embeddings) — ask the LLM to *write a fake answer* to the query, then embed that and use it for retrieval. Works because the fake answer's vector lives in "answer space," closer to real answers in your index.
- **Multi-query** — generate 3–5 paraphrases of the query, retrieve for each, fuse the results with RRF.
- **Query decomposition** — break complex queries into sub-questions, retrieve for each.

> Worth trying once basic retrieval is solid. Don't reach for these to fix bad chunking — they amplify whatever's underneath, good or bad.

---

## Evaluation: how do you know it's working?

You can't tune what you can't measure. Build a small eval set early — even **30 hand-labeled `(query, ideal_doc_id)` pairs** is enough to start.

| Metric              | What it measures                                         |
|---------------------|----------------------------------------------------------|
| **Recall@K**        | Did the right chunk show up in the top K?                |
| **MRR**             | How high in the ranking was the right chunk?             |
| **nDCG**            | Same as MRR but with graded relevance                    |
| **Faithfulness**    | Does the LLM answer actually use the retrieved chunks?   |
| **Answer correctness** | Is the final answer right? (LLM-as-judge or human)    |

### How they map to fixes

```
Recall@K is low         → fix retrieval: chunking, embeddings, hybrid
Recall@K is OK, MRR low → fix ranking: add a reranker
MRR good, faithfulness low → fix generation: prompt, K, chunk quality
```

> Optimize Recall@K first (retrieval problem), then MRR / nDCG (ranking problem), then faithfulness (generation problem). Mixing them up wastes weeks. Diagnose before treating.

---

## Common mistakes

| Mistake                                                | Symptom                                            |
|--------------------------------------------------------|----------------------------------------------------|
| Mixing embedding models in one index                   | Garbage similarity scores, silent quality drop     |
| Forgetting `input_type=search_query` on Cohere queries | 10–20% recall loss, no error                       |
| K=20+ passed straight to the LLM                       | Lost-in-the-middle, hallucinated answers           |
| No overlap between chunks                              | Answers split across chunks → neither retrieved    |
| Pure vector, no BM25                                   | Misses exact terms (SKUs, codes, names)            |
| No reranker                                            | Top-1 is often not the best; LLM gets noisy context|
| No eval set                                            | Tuning by vibes; regressions go unnoticed          |
| Re-indexing only new docs after model swap             | Old vectors incompatible — must rebuild full index |

---

# Quick Reference — All Three Concepts

```
EMBEDDINGS — GPS COORDINATES FOR MEANING
  Vector of floats representing meaning of a piece of text
  Same model on both sides — never mix models in an index
  Cosine similarity is the standard metric (1 = same, 0 = unrelated)
  Dimensions: 256 / 1024 / 1536 / 3072 — bigger ≠ always better
  Vectors only have meaning relative to other vectors from the same model

PICKING A MODEL
  Titan v2          → AWS-native, IAM-bound, 8k tokens, multilingual
  Cohere Embed v3   → top retrieval quality, 512 tokens, asymmetric
  OpenAI 3-small    → cheap baseline, 8k tokens
  OpenAI 3-large    → highest quality + cost
  Cohere needs      → input_type=search_document for indexing
                      input_type=search_query    for retrieval
  Always benchmark on YOUR data, not MTEB

CHUNKING — SLICING DOCS RIGHT
  Fixed-size      → simple, splits mid-sentence, baseline
  Recursive       → hierarchy of separators, default for prose
  Semantic        → breaks on meaning shifts, best for free-form prose
  Defaults        → 256–512 tokens, 10–20% overlap
  Always attach metadata: doc_id, section, page, updated_at, tags

CHUNKING RULES
  Tabular data → don't chunk, index rows
  Code         → AST / symbol-based splitter
  Markdown     → respect headings first, then recurse
  FAQs         → one chunk per Q&A pair, never split a question
  Bad retrieval is almost always a chunking problem first

TOP-K RETRIEVAL — THE FUNNEL
  Retrieve wide (K=50), trim narrow (top 5 after rerank)
  K=5–10 is the usual final number passed to the LLM
  ANN indexes: HNSW (default), IVF (huge corpora), Flat (<100K)
  Tune ef_search / nprobe by measuring recall@K vs latency
  Lost-in-the-middle: LLMs underweight the middle of long contexts

HYBRID SEARCH — VECTOR + BM25
  BM25 catches exact terms / SKUs / rare tokens / negation
  Vector catches semantic / paraphrase matches
  Combine with Reciprocal Rank Fusion (RRF) — k=60 default
  RRF beats weighted-score blending because it's scale-free
  OpenSearch hybrid query supports this natively

RERANKING — THE HIGHEST-ROI ADD-ON
  Bi-encoder = fast, lossy   → first-pass retrieval
  Cross-encoder = slow, accurate → rerank top 50 → top 5
  Cohere Rerank v3 / bge-reranker / ms-marco MiniLM
  Single biggest quality lift after basic retrieval works

PIPELINE ORDER
  query → (rewrite) → BM25 + vector → RRF → rerank → filter → LLM
  Skip in this order if you must: rerank > hybrid > rewrite
  Never skip chunking quality

EVAL — DIAGNOSE BEFORE YOU TREAT
  30 (query, doc_id) pairs is enough to start
  Recall@K low         → retrieval problem (chunking, embeddings, hybrid)
  Recall@K OK, MRR low → ranking problem (add reranker)
  MRR good, faithfulness low → generation problem (prompt, K, chunks)

COMMON MISTAKES
  Mixing models in one index
  Forgetting Cohere input_type
  K=20+ straight to the LLM (lost-in-the-middle)
  No chunk overlap
  Pure vector, no BM25
  No reranker
  No eval set
  Partial re-index after model swap
```
