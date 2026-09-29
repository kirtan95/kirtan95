# Dev Log

A daily engineering journal — short notes on things I'm learning and building.

## 2026-09-28

**RAG retrieval: chunking matters more than the embedding model.** Been working on a RAG knowledge assistant (FastAPI + pgvector). Swapping embedding models moved hit-rate a few points, but fixing chunk boundaries — keeping tables and lists intact instead of naive fixed-size splits — moved it far more. Lesson: retrieval quality is a data-plumbing problem first, a model problem second.
