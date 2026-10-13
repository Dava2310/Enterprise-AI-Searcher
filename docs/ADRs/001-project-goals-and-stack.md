# ADR-001: Project goals and technology stack

- Status: Accepted
- Date: 2026-10-13

## Context

This project is an enterprise search engine with AI: it connects to sources such as Google Drive,
Notion, Slack and GitHub, indexes their content, and answers questions with citations while
respecting what each user is allowed to see.

It is built for two reasons: to learn deliberately, and to produce a portfolio piece that
demonstrates production-grade backend skills. The hard part is not calling an LLM, but the
backend around it: permissions, incremental sync, tenant isolation and measured search quality.

Constraints:

- One developer, 10 to 15 hours per week.
- The author is experienced with SQL and Node.js (Express, NestJS), but new to Python backends.
- Every improvement must be justified by a metric, so the stack must make evaluation easy.
- Permissions must never leak. This is the only fixed target from day one.

## Options considered

### Language and API framework

1. **Python + FastAPI.** Native async, automatic validation (Pydantic) and OpenAPI docs. Python is
   the main ecosystem for AI work (embeddings, evaluation, agent frameworks). Cost: a new language
   and a new async model for the author.
2. **Node.js + NestJS.** Already familiar, so faster at the start. Cost: a thinner ecosystem for
   ML, embeddings and evaluation, and it would not build the Python skills this project aims for.

### Storage and vector search

1. **PostgreSQL + pgvector.** Documents, permissions, vectors and full-text search in one place,
   so the permission filter runs inside the same query as the similarity search.
2. **Dedicated vector database (Qdrant, Pinecone, ...).** Optimized for vectors, but permissions
   would live in a different system from the vectors, which creates a synchronization problem and
   a new way to leak documents.

### Background processing

1. **Redis + arq.** Async-native and simple.
2. **Celery.** More features and control, but heavier. Kept as a fallback if arq falls short.

### LLM integration

1. **Direct provider SDK, hand-written RAG and agent loop first.** Forces understanding of every
   step.
2. **LangChain / LlamaIndex from the start.** Faster, but hides the mechanics this project is
   meant to teach. LangGraph is adopted later (milestone 6) and compared against the hand-written
   agent.

## Decision

Use **Python with FastAPI**, **PostgreSQL with pgvector** for data, permissions and vectors,
**Redis with arq** for background work, and the **provider SDK directly** for LLM calls, with
frameworks introduced only after the hand-written version exists.

Tooling: uv, ruff, mypy (strict), pytest, pre-commit, Docker Compose and GitHub Actions CI.

Keeping permissions next to chunks in PostgreSQL is the key choice: the access filter is applied
within the retrieval query, not after it.

## Consequences

- **Gain:** a single database to operate and test, and a retrieval path where permission leaks
  can be tested directly.
- **Gain:** the stack matches the skills the project aims to demonstrate.
- **Cost:** a steeper learning curve at the start (Python typing, the async model, a new
  ecosystem) compared with staying on Node.js.
- **Cost:** pgvector may become a bottleneck at large scale. That is acceptable for a corpus of
  this size, and the decision can be revisited with an ADR if measurements show otherwise.
- **Cost:** writing the RAG and the agent without frameworks takes longer, in exchange for
  understanding what a framework would later abstract.
- **Pending:** the project name, and whether to choose arq or Celery will be confirmed when the
  first worker is built.
