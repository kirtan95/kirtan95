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

## 2026-10-02

**Slack bots: ack fast, verify signatures, dedupe events.** Been wiring up a Slack bot and the two things that bite hardest are the 3-second ack window and duplicate event delivery. Slack retries a webhook if you don't ack in time, so every handler should ack first and hand off the real work to a background task, never do it inside the request. Every request also needs signature verification against the signing secret using the timestamp header, with a short expiry check to block replay attacks. And since retries can still deliver the same event twice, I keep a table of processed event_ids with a unique constraint so a duplicate insert just no-ops. Pattern: verify, ack, dedupe, then act. Boring plumbing, but it's what separates a demo bot from one that survives a busy channel.

## 2026-10-03

**Tool-calling safety: validate the arguments before the tool ever runs.** I have been adding function-calling to my agent workflow and the habit that has saved me the most grief is parsing the model's JSON arguments through a Pydantic model before dispatching. LLMs will happily return `{"count": "five"}` or invent parameters the tool doesn't accept, and a loose `**kwargs` pass-through turns those hallucinations into real bugs. So each tool now declares a strict schema, the dispatcher validates against it, and anything that fails gets re-asked to the model with the validation error included in the message. The bonus is that the same schema renders the tool descriptions for the API and the docs, so they can't drift apart. Small layer, but it is the difference between an agent that occasionally breaks and one that degrades gracefully.

## 2026-10-04

**WebSocket backpressure: stop buffering, start pushing back.** I hit a nasty failure mode in a Node.js WebSocket server under load: when a slow client cannot keep up, the server's per-socket send queue grows unbounded and eventually OOMs the process. The fix is `ws.bufferedAmount` - before each send, check whether buffered bytes exceed a threshold and, if so, either drop the message (for ephemeral game-state updates that will be superseded) or throttle that client. Better still, don't let clients subscribe to firehose updates they can't consume; batch state diffs on the server tick and send one compact snapshot per tick instead. Lesson: backpressure is a protocol design problem, not a bandwidth one - the server must decide what to do when the pipe is full, because silently buffering is just choosing to crash later.
## 2026-10-05

**React dashboards: downsample before you render, not after.** Been building a metrics dashboard in React with Chart.js and hit the wall where 50k time-series points turn the UI to molasses. Pagination helped the table but not the chart, so I switched to downsampling server-side with the Largest-Triangle-Three-Buckets algorithm before the payload ever reaches the client. The chart keeps its visual shape, payloads shrink by 95%+, and the raw points stay in Postgres for the drill-down view. I also moved the chart into a memoized component keyed on the data hash so unrelated state updates stop triggering full re-renders. Lesson: the browser is a display device, not a compute device — do the heavy lifting in Python, ship the pixels to React.

## 2026-10-06

**Idempotent pipelines: make every write safe to retry.** I have been hardening a Python ingestion pipeline that pulls records from an API and writes them to Postgres, and the failure I kept hitting was the duplicate-write-on-retry: a worker times out, the orchestrator retries, and suddenly there are two rows for one event. The fix that actually held was giving every record a stable natural key and writing with `INSERT ... ON CONFLICT (source, external_id) DO UPDATE SET ...`, so a retry is just a no-op instead of a data corruption event. On top of that I wrapped each batch in an explicit transaction and record a checkpoint cursor only after the commit succeeds, so a crash replays from the last good cursor rather than from somewhere in the middle. The lesson that stuck: retries are inevitable in distributed pipelines, so idempotency has to be a property of the write path, not something you bolt on later.
