# Enterprise Knowledge Assistant

LangGraph + RAG + RAGAS + MCP assistant that answers questions over
internal policy documents, scores its own answer, and writes an audit
report to disk through an MCP server. Everything runs on free tiers only.

## Architecture Overview

```
Phase 1 - ingestion (run once)
  data/*.txt -> load -> chunk -> embed (Ollama) -> store in Chroma

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
ollama pull nomic-embed-text
copy .env.example .env      # then edit .env, add GROQ_API_KEY
```
patch_ragas.py is a one-time fix. ragas 0.3.9 imports a Google Vertex AI class that no longer exists in current langchain-community versions — this project doesn't use Vertex AI, so the script just makes that import optional instead of crashing the app. Safe to re-run any time.

## Execution Steps

```powershell
uv run python main.py
uv run python main.py "What is the meal expense limit for domestic travel?"

# or the browser UI:
uv run streamlit run app.py
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
| Generation | Groq, `llama-3.3-70b-versatile`, answers only from retrieved context |

**Why Groq + Ollama instead of one provider:** Groq is free and very fast
for chat/generation but has no embeddings endpoint at all, so embeddings
run locally via Ollama at zero cost instead.

## LangGraph Design

| Node | Responsibility |
|---|---|
| `retriever` | Similarity search in Chroma, adds `context` to state |
| `responder` | Prompts Groq with context + question, adds `answer` |
| `evaluator` | Runs RAGAS, writes report via MCP, adds `scores` + `report_path` |

Flow is strictly linear: `retriever -> responder -> evaluator -> END`,
one shared `AssistantState` dict passed through all three.

## MCP Integration

- **Server:** official MCP Filesystem server (`@modelcontextprotocol/server-filesystem`), spawned via `npx`. Chosen because the brief lists it as the recommended starting point and it needs no auth/account setup.
- **Tool used:** `write_file`.
- **Where it's used:** the Evaluator node calls it after RAGAS scoring to save a Markdown report (question, context, answer, all 4 scores) to `reports/`. Same call pattern would work for a Jira or GitHub MCP server, just a different server + tool name.

## RAGAS Evaluation

All 4 metrics implemented:

| Metric | Type | Needs reference? |
|---|---|---|
| Faithfulness | Mandatory | No |
| Answer Relevancy | Mandatory | No |
| Context Precision | Bonus | No (`LLMContextPrecisionWithoutReference`) |
| Context Recall | Bonus | Yes |

Context Recall always needs a ground-truth reference answer to check
against - that's what it measures. Since there's no curated answer key
for arbitrary questions here, the generated answer is reused as its own
proxy reference. This is a simplification, flagged in the report output;
a production setup would use a curated question/reference-answer test set.

Judge model: Groq (same model used for generation) via `LangchainLLMWrapper`.
Interpretation: score >= 0.8 Good, 0.5-0.8 Moderate, < 0.5 Poor.

## Observability (requirement 5)

No LangSmith needed. Every node prints a bordered `[NODE] -> / <-` block
showing its input and output before/after running. One console capture
of a run shows all three agents firing in order with real data - that's
the node-by-node execution trace deliverable.
