# Internship Interview Preparation — Quantjo AI Platform

Grounded in the actual source of `quantjo-saas`, `quantjo-agent-tests`, and `quantjo-ui-saas`.
Every claim below traces to a file you can open in the interview if asked.

---

## READ THIS FIRST — Stack Reality Check

Your original brief listed technologies that **are not in this codebase**. I verified by grepping
both `pyproject.toml` files, the entire `quantjo_api/` package, and `quantjo-agent-tests/src/`.

| Brief claimed | What the repository actually contains |
|---|---|
| LangGraph | **OpenAI Agents SDK** (`openai-agents>=0.9.0`). `Runner.run_streamed()`, `Agent`, `@function_tool`. No graph library. |
| ChromaDB | **Nothing.** No vector database, no `chromadb` dependency, no collection, no persistence dir. |
| RAG / embedding search | **No embeddings.** Retrieval is exact-ID filesystem lookup: `read_sdk_page(page_id)` → `agent_kb/wiki/_tool/topics/{page_id}.md`. Zero similarity scoring. |
| tree-sitter | **Python stdlib `ast`.** `ast.parse()`, `ast.walk()`, `ast.ClassDef`. See `harness/ast_utils.py`. |
| Model Context Protocol / FastMCP | **Not present.** Tools are plain `@function_tool`-decorated async functions. |
| GitHub APIs | **Not present** in application code. |

**Do not claim these in an interview.** If asked "did you use a vector DB?", the correct and much
stronger answer is:

> "No — and that was a deliberate call. Our knowledge base was one 20K-token guide covering fifteen
> well-separated topics. That's a closed, small, human-curatable domain. Embedding it would have added
> an index, an embedding model, a similarity threshold, and a whole class of retrieval failures, to
> solve a routing problem that a 350-token table of contents solves exactly. I got 96% token reduction
> with zero recall risk, because exact-ID lookup can't return the wrong chunk. If the corpus had grown
> to thousands of documents, or become user-generated, vector search becomes the right answer — and I
> can tell you what I'd have built."

That answer demonstrates judgment. "We used ChromaDB" demonstrates you followed a tutorial.

---

## The Three Contributions — Verified

| Contribution | Status | Primary evidence |
|---|---|---|
| **Token cost optimization** (~20K → ~350 tokens) | **Real, merged, running** | `agent_kb/wiki/_tool/_token_report.md` measures 19,659 → 792. `system_prompt.py:_build_sdk_navigator()` emits ~350 tokens of it. Commit `dc3e340`. |
| **Agentic regression harness** (20 cases, AST + LLM judge) | **Real, standalone repo** | `quantjo-agent-tests/`. Exactly 20 case YAMLs. Three assertion layers. Reports in `reports/`. |
| **Real-time agent streaming** | **Real, in production path** | `agent_loop.py:_handle_stream_event()`, `event_emitter.py`, `connection_manager.py`, `routes/websocket.py`. |

One nuance worth being precise about: the token report's headline is **19,659 → 792** for the
always-on floor (that 792 is the raw `INDEX.md` file). The **~350 tokens** figure is the *runtime*
navigator — `_build_sdk_navigator()` regex-extracts only the topic id + "load when" column from
INDEX.md and discards the rest. So both numbers are true; they measure different artifacts. Say
"about 96% off the always-on prompt" and you're safe either way.

---

## Files

| # | File | Covers |
|---|---|---|
| 01 | [company_and_product.md](01_company_and_product.md) | Product, platform, team, your role |
| 02 | [token_cost_optimization.md](02_token_cost_optimization.md) | The navigator architecture — your headline contribution |
| 03 | [rag_pipeline.md](03_rag_pipeline.md) | Retrieval as actually built, vs. vector RAG |
| 04 | [regression_testing.md](04_regression_testing.md) | The 20-case multi-turn harness |
| 05 | [ast_validation.md](05_ast_validation.md) | Layer 1 — stdlib `ast` predicates |
| 06 | [llm_judge.md](06_llm_judge.md) | Layer 2 — LLM-as-judge |
| 07 | [websocket_streaming.md](07_websocket_streaming.md) | Real-time streaming |
| 08 | [fastapi_backend.md](08_fastapi_backend.md) | Routing, DI, async, background tasks |
| 09 | [agent_architecture.md](09_agent_architecture.md) | Single-agent + Codex-as-tool |
| 10 | [system_design.md](10_system_design.md) | End-to-end architecture |
| 11 | [scalability.md](11_scalability.md) | What breaks first, and why |
| 12 | [security.md](12_security.md) | Secrets, auth, prompt injection, rate limits |
| 13 | [deployment.md](13_deployment.md) | Dev, test, deploy, monitor |
| 14 | [my_contributions.md](14_my_contributions.md) | Exactly what you built |
| 15 | [mock_interview.md](15_mock_interview.md) | 100 questions |

---

## How to Use This

1. Read **14** first. It is the ground truth of what you can defend.
2. Memorize the **three interview answers** in 02, 04, 07. Those are your headline stories.
3. Skim **10** and **11** — system design rounds at Amazon/Google will go there.
4. Drill **15** the night before.

**The one rule:** if you did not build it, say so. "That part existed before I joined; what I added
was X" is a *strong* answer. Interviewers at this level are calibrated to detect ownership inflation,
and they probe exactly where you sound least specific.
