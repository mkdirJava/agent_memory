# Agent Memory Service — Architecture

> **What this is:** a **microservice** — a single FastAPI application that is
> built, tested, and deployed as one unit. It exposes `POST /memory` and
> `GET /memory/related`, and talks to storage (Postgres, Neo4j) and an LLM
> provider directly.
>
> What people casually call "the Memory Creation Service" and "the Memory
> Retrieval Service" are **modules inside this one microservice**.
>
> The **Memory Linker**, however, is a **separate microservice**. It does
> not run inside the Agent Memory Service: it **listens to a messaging
> system**, and the `memory-creation` module inside the Agent Memory
> Service **pushes events to that messaging system** whenever a memory is
> created (`memory.created`) or updated (`memory.updated`). The two talk
> to each other only through the broker.

## Component Model

| Component | Type | Responsibility |
|---|---|---|
| **Agent Memory Service** | microservice (FastAPI) | The main deployable unit; exposes `POST /memory` and `GET /memory/related` |
| **memory-creation** | module (inside Agent Memory Service) | Summarizes the scenario, creates entities, decides same-scenario vs. new (small model), updates or creates the memory record |
| **memory-retrieval** | module (inside Agent Memory Service) | Accepts queries, surfaces candidates via vector search (Postgres), then merges and ranks them with graph context (Neo4j) |
| **Messaging System** | shared broker | Event bus; the only channel between the Agent Memory Service and the Memory Linker |
| **Memory Linker** | microservice (separate deployment) | Consumes `memory.created` and `memory.updated` events; creates or updates the co-reference edges accordingly |
| **LLM** | external HTTP API (OpenAI-compatible) | Used by the memory-creation module for summarization, entity extraction, and the small-model scenario check |
| **Storage** | shared data store | Postgres (vector search + metadata) + Neo4j (entity/edge graph) |

## High-Level Diagram

```
                        ┌─────────────────────────┐
                        │   Client / Agent App    │
                        └────────────┬────────────┘
            POST /memory    GET /memory/related
                                 │
                                 ▼
    ┌─────────────── Agent Memory Service (FastAPI) ───────────────┐
    │          ┌──────────────┐          ┌──────────────┐          │
    │          │ memory-      │          │ memory-      │          │
    │          │ creation     │          │ retrieval    │          │
    │          └──┬────────┬──┘          └──────┬───────┘          │
    └─────────────┼────────┼────────────────────┼──────────────────┘
                  │        │                    │
      persisted (create)   push created   read: vector
                   │        │             search → fetch
                   │        │             entity/edge
                   │        │             details
                   │        │                    │
       ┌──────────┘  ┌─────▼───────┐            │
       │             │  Messaging  │            │
       │             │   System    │            │
       │             └─────┬───────┘            │
       │                   │ consume             │
       │              ┌────▼─────┐               │
       │              │  Memory  │               │
       │              │  Linker  │               │
       │              └────┬─────┘               │
       │                   │ create links         │
       │                   │                      │
       ▼                   ▼                      ▼
    ┌────────────────────────────────────────────────────┐
    │                     Storage                         │
    │                 Postgres + Neo4j                     │
    └────────────────────────────────────────────────────┘
```

The diagram deliberately does not show the LLM client (used only by the
**memory-creation** module inside the service).

## Data Flow

### 1. `POST /memory` — creating a memory

**Flow at a glance**

`utterance → summarize → entities → same-scenario? → UPDATE or CREATE → persist → one event → response`

**How it works**

1. **Summarize the scenario** — distill the utterance into a short scenario description.
2. **Extract / create entities** — identify the entities that will live in the graph.
3. **Small-model same-scenario check** — ask a lightweight model: "is this the same scenario as the most recently created memory?" (A small model, not the larger model used for summarization/entity extraction; it reads only the last created memory.)
4. **Decide: merge or create** —
   - *Same scenario* → **UPDATE** the existing memory (merge the new details in).
   - *New scenario* → **CREATE** a new memory record.
5. **Persist** — store vectors + metadata in Postgres, entities in Neo4j (the **persist** arrow in the diagram).
6. **Publish exactly one event** — the branch from step 4 determines which (see the table below).

**Branch → event (exclusive — the two never cross)**

| Branch taken (step 4) | Event published (step 6) |
|---|---|
| Same scenario → **UPDATE** | `memory.updated` |
| New scenario → **CREATE** | `memory.created` |

The Memory Linker microservice consumes the event over the messaging system and performs the entity/edge writes (the **create links** arrow).

**Notes**

- Events are **exclusive**: `memory.created` fires only on the **CREATE** branch; `memory.updated` only on the **UPDATE** branch.
- Steps 5 and 6 are **not atomic**: the memory is stored before the linker finishes; a retrieval during that window simply sees fewer edges — no correctness harm.
- The memory-creation module does **not** wait for the Memory Linker microservice: it publishes the event and returns the response immediately.

### 2. `GET /memory/related?query=...&k=N` — retrieving a memory

**Flow at a glance**

`query → vector search (Postgres) → entities + edges (Neo4j) → merge & rank → top-k`

**How it works**

1. **Surface candidates** — embed the raw query and run a vector search in Postgres to get candidate memory IDs + scores.
2. **Pull graph details** — for the top candidates, fetch their entities and the entity/memory edges that connect them (Neo4j).
3. **Merge & rank** — combine vector scores with entity context from the graph, then return the memories whose graph-backed details match best (top-k).

**Notes**

- In short: text in → vector search surfaces candidates → pull matching details (entities and edges) from the graph (Neo4j) → rank and return.
- Retrieval reads **only from storage**; edge writes are the exclusive job of the Memory Linker microservice.
- No LLM call on the read path: every stored memory is already the LLM summary written at CREATE time, so the query is matched directly against those summaries.

## Process and Deployment Model

- **One process = one microservice.** The Agent Memory Service is a single
  FastAPI process — `memory-creation` and `memory-retrieval` share the same
  event loop, same Postgres/Neo4j connection pools, same LLM client. There
  is no IPC between them; they are function calls.
- The **Memory Linker** is its own process and its own deployment. It talks
  to the Agent Memory Service **only through the messaging system** — it
  listens for `memory.created` and `memory.updated` events and makes no HTTP
  requests to the Agent Memory Service.
- Because creation and retrieval live in the same process, one `uvicorn`
  invocation serves both endpoints; that process is the unit you deploy,
  scale, and restart.
- The Messaging System is shared infrastructure — the sole coupling point
  between the two services.

## Storage Model

| Store | Purpose |
|---|---|
| **Postgres** | Vector embeddings for memory + `memory_meta` (entities, source, confidence) |
| **Neo4j** | `entity` nodes (text, category) + `memory_has_entity`/`entity_in` edges |

Both are **shared backing data** for the microservice; each process (Agent
Memory Service, Memory Linker) gets its own connection pool. There is no
separate storage service.
