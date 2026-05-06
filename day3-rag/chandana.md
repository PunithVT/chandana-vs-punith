# Day 3 — Chandana (AWS)

Notes, labs, and experiments for Day 3.

## Topics
- [x] Vector storage on AWS — OpenSearch Serverless, pgvector on RDS, Bedrock Knowledge Bases
- [ ] Putting it together — a small Lambda + Bedrock + OpenSearch RAG chatbot

---

## 1. Vector Storage on AWS

### What is a vector and why does it need special storage?

An embedding model converts text into a list of floats called a **vector**.  
Example: `"The dog barked"` → `[0.23, -0.87, 0.41, ...]` (1536 numbers)

Semantically similar text → numerically similar vectors.

The query you run is not `WHERE column = value`.  
It is **"find the K vectors closest to this query vector"** — called nearest neighbour search.

Normal databases can store vectors but cannot search them fast.  
Vector databases solve this with **ANN indexes** (e.g. HNSW) — find K nearest vectors in milliseconds without scanning everything.

---

### The three AWS options at a glance

| | pgvector on RDS | OpenSearch Serverless | Bedrock Knowledge Bases |
|---|---|---|---|
| Complexity | Low — SQL extension | Medium — index + mapping config | Lowest — point at S3, done |
| Scale | ~5M vectors | Billions of vectors | AWS manages it |
| Hybrid search | Basic (SQL WHERE + vector) | Best (kNN + BM25 + filters) | Limited (Bedrock API only) |
| Cost | Cheapest — reuse existing RDS | ~$700/mo floor | Highest — managed premium |
| Best for | Existing Postgres infra | Large scale + rich filters | Fast prototyping |

---

### Option 1 — pgvector on RDS

A Postgres extension that adds a `vector` column type and ANN index support inside your existing database. No new infrastructure.

**Use when**
- You already have RDS Postgres running (Rooman does — just run `CREATE EXTENSION`)
- Dataset is small to medium (under ~5M vectors)
- You want SQL joins alongside vector search (`WHERE user_id = 42 ORDER BY similarity`)
- You want the simplest possible setup

**Limits**
- HNSW index lives in RAM — large datasets need bigger RDS instance
- Not designed for pure vector workloads at massive scale

```sql
-- Enable once
CREATE EXTENSION vector;

-- Table with a vector column
CREATE TABLE documents (
    id        SERIAL PRIMARY KEY,
    content   TEXT,
    metadata  JSONB,
    embedding vector(1536)       -- 1536 dims = Bedrock/OpenAI embedding size
);

-- HNSW index for fast approximate search
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- Query: find 5 most similar chunks to a query vector
SELECT content, metadata,
       1 - (embedding <=> '[0.23, -0.87, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.23, -0.87, ...]'::vector
LIMIT 5;

-- <=>  = cosine distance    1 - distance = similarity score
```

---

### Option 2 — OpenSearch Serverless (vector engine)

A full search engine with ANN vector search built in. Serverless = you pay per OCU consumed, no cluster to provision. Combines vector similarity + keyword (BM25) search in one query.

**Use when**
- You need **hybrid search** — semantic + keyword in one query
- Dataset is large (millions of vectors)
- You need rich metadata filtering (`similar to X AND category = 'legal' AND date > 2024`)
- You're already using OpenSearch for app search

**Limits**
- ~$700/month minimum even at zero traffic — overkill for small datasets
- Cold start latency after idle period
- More setup than a Postgres extension

```python
# kNN search — find 5 nearest neighbours
query = {
    "size": 5,
    "query": {
        "knn": {
            "embedding": {
                "vector": [0.11, -0.92, 0.38, ...],   # query vector
                "k": 5
            }
        }
    }
}

# Hybrid: vector + keyword filter in one query
hybrid_query = {
    "size": 5,
    "query": {
        "bool": {
            "must": [{"knn": {"embedding": {"vector": [...], "k": 5}}}],
            "filter": [{"term": {"metadata.category": "legal"}}]
        }
    }
}
```

---

### Option 3 — Bedrock Knowledge Bases

Fully managed RAG pipeline. Point it at an S3 bucket — Bedrock handles chunking, embedding, and storing vectors automatically. You query with natural language via API.

**Use when**
- You want to ship a RAG system fast with minimal code
- Source documents are files in S3 (PDFs, Word docs, text)
- You don't need custom chunking or fine-grained pipeline control
- Team wants to avoid managing embedding infrastructure

**Limits**
- Less control — chunking strategy and embedding model choices are limited
- Data must live in S3 (can't embed from a DB or API directly)
- Costs more than managing pgvector yourself

```python
import boto3

bedrock = boto3.client("bedrock-agent-runtime", region_name="us-west-2")

response = bedrock.retrieve(
    knowledgeBaseId="KB_ID_HERE",
    retrievalQuery={"text": "What is the refund policy?"},
    retrievalConfiguration={
        "vectorSearchConfiguration": {"numberOfResults": 5}
    }
)

for result in response["retrievalResults"]:
    print(result["content"]["text"])   # matched chunk
    print(result["score"])             # relevance score
    print(result["location"])          # source S3 object
```

---

### Which one to pick

```
Already have RDS Postgres?
  └── Yes → start with pgvector (CREATE EXTENSION, done)
        └── Outgrow it (> 5M vectors / need hybrid search)?
              └── Migrate to OpenSearch Serverless

Need to ship fast, source is S3, don't want to own the pipeline?
  └── Bedrock Knowledge Bases
```

> For Rooman context — start with **pgvector**. RDS is already running.
> Cost is near zero to add. Graduate to OpenSearch only if scale demands it.

---

## 2. How RAG works (the full pipeline)

The vector store is one piece. The same flow applies regardless of which option you pick.

```
User query
  ↓
Embedding model (Bedrock Titan / Cohere)
  converts query text → query vector
  ↓
Vector store (pgvector / OpenSearch / Bedrock KB)
  finds K nearest chunks by vector similarity
  ↓
Retrieved chunks (raw text passages)
  ↓
Prompt = system prompt + retrieved chunks + user query
  ↓
LLM (Claude via Bedrock)
  generates a grounded answer using the retrieved context
  ↓
Response to user
```

Only the vector store step changes between options.  
The embedding call before it and the LLM call after it are identical.

---

### RAG in code (pgvector + Bedrock + Claude)

```python
import boto3, psycopg2, json

bedrock = boto3.client("bedrock-runtime", region_name="us-west-2")
conn    = psycopg2.connect(host="...", dbname="rooman", user="...", password="...")

def embed(text: str) -> list[float]:
    resp = bedrock.invoke_model(
        modelId="amazon.titan-embed-text-v1",
        body=json.dumps({"inputText": text})
    )
    return json.loads(resp["body"].read())["embedding"]

def retrieve(query: str, k: int = 5) -> list[str]:
    vec = embed(query)
    cur = conn.cursor()
    cur.execute("""
        SELECT content
        FROM documents
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    """, (vec, k))
    return [row[0] for row in cur.fetchall()]

def rag(query: str) -> str:
    chunks  = retrieve(query)
    context = "\n\n".join(chunks)
    prompt  = f"""Answer using only the context below.

Context:
{context}

Question: {query}"""

    resp = bedrock.invoke_model(
        modelId="anthropic.claude-3-sonnet-20240229-v1:0",
        body=json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 512,
            "messages": [{"role": "user", "content": prompt}]
        })
    )
    return json.loads(resp["body"].read())["content"][0]["text"]

# Usage
print(rag("What is the refund policy?"))
```

---

### Key terms to remember

| Term | What it means |
|---|---|
| **Vector / embedding** | Text converted to a list of floats by an embedding model |
| **ANN index (HNSW)** | Data structure that finds nearest vectors fast without full scan |
| **Cosine similarity** | Distance metric — 1.0 = identical, 0.0 = unrelated |
| **Chunk** | A short passage of text that was embedded and stored |
| **k** | Number of nearest chunks to retrieve (typically 3–10) |
| **Hybrid search** | Vector similarity + keyword search combined in one query |
| **RAG** | Retrieve relevant chunks → stuff into prompt → LLM answers |