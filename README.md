# Enterprise Knowledge Assistant

LangGraph + RAG + RAGAS + MCP assistant that answers questions over
internal policy documents, scores its own answer, and writes an audit
report to disk through an MCP server. 

## Architecture Overview

```
Phase 1 - ingestion (run once)
  data/*.txt -> load -> chunk -> embed (nomic-embed) -> store in Chroma

Phase 2 - query (every question, the LangGraph part)
  question -> [Retriever] -> [Responder] -> [Evaluator] -> END
                                                  |
                                          RAGAS scores + MCP write_file
```

## Setup (using uv)

Requirements: Python 3.11+, Node.js (for `npx`, used by the MCP server),
[Ollama](https://ollama.com) installed and running, a free Groq API key,
[uv](https://docs.astral.sh/uv/) installed.

```powershell
uv venv
uv pip install -r requirements.txt
uv run python patch_ragas.py      # one-time fix, see below
ollama pull nomic-embed-text
copy .env.example .env            # then edit .env, add GROQ_API_KEY
```
## `patch_ragas.py` — what it's for

The pinned `ragas` version imports a Google Vertex AI class that no
longer exists in current `langchain-community`, which crashes the app
on startup even though this project never touches Vertex AI. The script
patches that import inside your installed `ragas` package so it fails
silently instead of crashing.

Run it once, right after installing dependencies and before running
`main.py` / `app.py`:

```powershell
uv run python patch_ragas.py
```

Safe to re-run anytime — it detects if it's already patched and exits
cleanly. Re-run it if you ever delete and recreate `.venv/`.

## Execution Steps

```powershell
# the browser UI:
uv run streamlit run app.py

# or
uv run python main.py
uv run python main.py "What is the meal expense limit for domestic travel?"
```

First run builds the Chroma store automatically. Delete `chroma_db/` to
re-ingest after changing files in `data/`.

## Frontend

`app.py` is a minimal Streamlit UI: a text box, an "Ask" button, the
answer, a table of the 4 RAGAS scores, and an expandable view of the
retrieved chunks. It calls the exact same `build_graph()` / `build_vectorstore()`
functions as `main.py` - the agentic system is identical, this is just a
thin display layer on top. The node-by-node execution log still prints to
the terminal you launched Streamlit from, not the browser.

## RAG Design

| Step | Choice |
|---|---|
| Source | `.txt` files in `data/` (3 sample HR/IT/finance policies) |
| Chunking | `RecursiveCharacterTextSplitter`, 500 chars, 50 overlap |
| Embeddings | Ollama, `nomic-embed-text` (local, free) |
| Vector DB | Chroma, persisted to `chroma_db/` |
| Retrieval | top-3 chunks by similarity |
| Generation | Groq, `openai/gpt-oss-120b`, answers only from retrieved context |

## LangGraph Design

| Node | Responsibility |
|---|---|
| `retriever` | Similarity search in Chroma, adds `context` to state |
| `responder` | Prompts Groq with context + question, adds `answer` |
| `evaluator` | Runs RAGAS, writes report via MCP, adds `scores` + `report_path` |

Flow is linear: `retriever -> responder -> evaluator -> END`,
one shared `AssistantState` dict passed through all three.

## MCP Integration

- **Server:** official MCP Filesystem server (`@modelcontextprotocol/server-filesystem`), spawned via `npx`. Chosen because the brief lists it as the recommended starting point and it needs no auth/account setup.
- **Tool used:** `write_file`.
- **Where it's used:** the Evaluator node calls it after RAGAS scoring to save a Markdown report (question, context, answer, all 4 scores) to `reports/`. Same call pattern would work for a Jira or GitHub MCP server, just a different server + tool name.

## RAGAS Evaluation

All 4 metrics implemented:

| Metric | Type | Needs reference? |
|---|---|---|
| Faithfulness | Mandatory |
| Answer Relevancy | Mandatory |
| Context Precision | Bonus |
| Context Recall | Bonus |

Judge model: Groq (same model used for generation) via `LangchainLLMWrapper`.
Interpretation: score >= 0.8 Good, 0.5-0.8 Moderate, < 0.5 Poor.

## Observability

No LangSmith needed. Every node prints a bordered `[NODE] -> / <-` block
showing its input and output before/after running. One console capture
of a run shows all three agents firing in order with real data that's
the node-by-node execution trace deliverable.
