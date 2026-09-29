# Dev Log

A daily engineering journal — short notes on things I'm learning and building.

## 2026-09-28

**RAG retrieval: chunking matters more than the embedding model.** Been working on a RAG knowledge assistant (FastAPI + pgvector). Swapping embedding models moved hit-rate a few points, but fixing chunk boundaries — keeping tables and lists intact instead of naive fixed-size splits — moved it far more. Lesson: retrieval quality is a data-plumbing problem first, a model problem second.

## 2026-09-29

**Streaming LLM tokens from FastAPI with SSE.** Wired up token streaming for a chat endpoint today using FastAPI's StreamingResponse with Server-Sent Events instead of blocking until the full completion. The trick is an async generator that iterates the model's token stream, yields `data: {chunk}` frames, and a client that consumes them with an EventSource. I learned to disable proxy buffering (`X-Accel-Buffering: no`) and flush headers immediately, otherwise the whole "stream" arrives in one batch. Also made the endpoint accept a `stopped` signal so the client can cancel mid-stream and the generator releases the model call. Latency to first token dropped from seconds to under a second in the UI, which changes how the app feels entirely.
