# Dev Log

A daily engineering journal — short notes on things I'm learning and building.

## 2026-09-28

**RAG retrieval: chunking matters more than the embedding model.** Been working on a RAG knowledge assistant (FastAPI + pgvector). Swapping embedding models moved hit-rate a few points, but fixing chunk boundaries — keeping tables and lists intact instead of naive fixed-size splits — moved it far more. Lesson: retrieval quality is a data-plumbing problem first, a model problem second.

## 2026-09-29

**Streaming LLM tokens from FastAPI with SSE.** Wired up token streaming for a chat endpoint today using FastAPI's StreamingResponse with Server-Sent Events instead of blocking until the full completion. The trick is an async generator that iterates the model's token stream, yields `data: {chunk}` frames, and a client that consumes them with an EventSource. I learned to disable proxy buffering (`X-Accel-Buffering: no`) and flush headers immediately, otherwise the whole "stream" arrives in one batch. Also made the endpoint accept a `stopped` signal so the client can cancel mid-stream and the generator releases the model call. Latency to first token dropped from seconds to under a second in the UI, which changes how the app feels entirely.
## 2026-09-30

**Lean Docker images for FastAPI + ML services.** Reworked a Dockerfile for a FastAPI service that pulls in ML libraries, and the big win came from treating the image like a supply chain instead of a junk drawer. A multi-stage build compiles native dependencies in a builder stage and copies only the installed packages into a python-slim runtime stage, so build tools never ship to production. Ordering the Dockerfile so dependency installs happen before the code copy keeps those layers cached across deploys, which makes rebuilds noticeably faster during iteration. Running the container as a non-root user and pinning the base image digest closes two easy security holes. Lesson learned: most of a "heavy ML image" is build debris, not the app itself.

## 2026-10-01

**pgvector HNSW: ef_search is the dial that matters.** Been tuning vector search in my RAG service (pgvector on Postgres). I already had an HNSW index built with m=16 and ef_construction=64, but recall versus latency still felt off. Turns out the build-time parameters matter far less than ef_search at query time - raising it from 40 to 100 lifted recall noticeably while keeping p95 latency comfortably in budget. The cleaner fix was setting it per-session with `SET hnsw.ef_search` instead of hardcoding it in the query, so I can tune it without redeploying. Lesson: index build params are set-and-forget, but ef_search is a runtime knob worth benchmarking against a golden question set.
