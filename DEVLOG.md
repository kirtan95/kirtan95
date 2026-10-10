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

## 2026-10-07

**asyncpg pool sizing: size the pool like a budget.** I was tuning a FastAPI service against Postgres and kept hitting `too many clients already` under load. The fix was creating one `asyncpg` pool in the app lifespan (`min_size=5, max_size=20`) and sharing it everywhere, instead of opening a connection per request. My rule of thumb now: pool max should be roughly uvicorn workers times connections each can hold, staying well under Postgres `max_connections` with headroom left for admin work and migrations. I also set `command_timeout` on the pool so a stuck query gives its connection back instead of starving everyone. The lesson that stuck: a connection pool is a backpressure device, not a performance feature. Sizing it is deciding how many concurrent writers you are willing to fund.

## 2026-10-08

**MCP tool servers: an agent's tools are just a JSON-RPC API.** I have been packaging some of my agent workflow tools (search, file read, database queries) as an MCP server so any MCP-capable client can call them over stdio. The protocol is plain JSON-RPC 2.0: the client lists available tools, then sends `tools/call` with the tool name and arguments, and the server responds with result content. The habit that paid off was giving every tool a strict input schema and a hard timeout - one slow tool without a timeout stalls the whole agent loop, and MCP clients trust the schema you declare. The second lesson: return structured data (JSON strings) instead of prose, because the model parses JSON far more reliably than it summarizes a paragraph. It is the same discipline as my function-calling validation work, just moved to the transport layer - and it means the same tools work in Claude Desktop, my own agent, or anything else that speaks MCP.

## 2026-10-09

**Slack bots: ack first, think later.** While wiring up a slash command for a small ops bot with slack-bolt (Python, async app), I learned Slack re-delivers the request if it does not get an `ack()` within 3 seconds, so slow handlers end up firing twice. The handler now acknowledges immediately and pushes the slow work - the LLM call, the database lookup - into a background task, then updates the message later with `response_url`. I also verify the signing secret on every request so the endpoint cannot be hit by anyone who guesses the URL. The pattern generalizes to any webhook: accept fast, do the real work after you have already answered. Kept the bot single-purpose instead of a grab bag of commands, which also kept the scope honest.

## 2026-10-10

**Hybrid search: keyword precision plus vector recall, fused with RRF.** In my RAG service (Postgres + pgvector) I kept seeing pure vector search miss exact terms like product codes and error names, while plain full-text search missed paraphrases of the same idea. So I now run both for every query: `to_tsvector`/`tsquery` for keyword matching alongside the HNSW index for semantic recall, then fuse the two rankings with Reciprocal Rank Fusion so neither score scale can dominate. The fusion is just a SQL expression over two ranked subqueries, so it lives entirely in Postgres with no extra service to run. On my golden-question eval, recall improved without touching a single embedding, because the two retrievers fail on different questions. Lesson: hybrid search is not two systems bolted together, it is admitting one signal is never enough.
