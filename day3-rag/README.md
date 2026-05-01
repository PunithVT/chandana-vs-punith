# Day 3 — RAG end-to-end

Topic for today: **Retrieval-Augmented Generation (RAG)**. Building on Day 1's Lambda — putting together a real pipeline so we can both see how the AWS pieces and the retrieval pieces fit.

## Topics

1. Embeddings 101 — what they are, picking a model (Titan, Cohere Embed, OpenAI)
2. Chunking strategies — fixed-size vs recursive vs semantic, and what breaks each
3. Vector storage on AWS — OpenSearch Serverless, pgvector on RDS, Bedrock Knowledge Bases
4. Retrieval + reranking — top-k, hybrid (BM25 + vector), cross-encoder rerank
5. Putting it together — a small Lambda + Bedrock + OpenSearch RAG chatbot

## Notes

- [punith.md](./punith.md) — Agentic AI angle on the topics
- [chandana.md](./chandana.md) — AWS angle on the topics

Tracking issue: [#3](https://github.com/PunithVT/chandana-vs-punith/issues/3)
