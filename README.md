# PortfolioInsight CLI

PortfolioInsight CLI is a production-style Python 3.11+ Retrieval-Augmented Generation (RAG) CLI for a BYLD Wealth take-home assignment.

It now includes a Rich Terminal UI (TUI) for executive scannability, while still keeping strict JSON outputs for automation.

It is designed to answer portfolio and market context questions using:

- deterministic local data,
- retriever + vector index,
- a multi-step LangGraph pipeline,
- structured Pydantic output,
- and a deterministic fallback path when model access is unavailable.

This project is intentionally CLI-only (no chatbot UI) so outputs can be tested and consumed by automation.

---

## CLI Preview

<img src="docs/preview.png" alt="CLI Preview" width="750"/>


---

## Architecture at a glance

The workflow is a **Variant C — Multi-Step Agent** implemented as a LangGraph state machine:

<p align="center">
  <img src="docs/agent_graph.png" alt="Agent Graph Flow" width="180" />
</p>

Each query passes through four sequential nodes before returning a typed response:

| Node | What it does |
|---|---|
| `retrieve` | Similarity search against Chroma vector store — returns top-6 chunks from news + glossary |
| `cross_reference` | Maps retrieved chunk content to portfolio tickers; applies global bypass for full-portfolio and risk queries so safe/defensive assets are never filtered out |
| `rank` | Calculates exposure weights per holding by market value; orders items for the LLM prompt |
| `format` | Calls Ollama via `with_structured_output` — returns a strict `GeneralQA` or `NewsImpact` Pydantic shape; falls back to deterministic heuristic if Ollama is unavailable |

If Ollama is unreachable, the graph short-circuits at `format` and returns a data-aware fallback (real tickers extracted from state) rather than a generic error.

Main package layout:

- `portfolio_ask/indexer.py`: document loading and chunking (`chunk_size=500`, `chunk_overlap=50`)
- `portfolio_ask/vector_store.py`: Chroma + BAAI embedding setup
- `portfolio_ask/llm.py`: Ollama model and deterministic fallback behavior
- `portfolio_ask/agent.py`: LangGraph state machine
- `portfolio_ask/main.py`: Rich TUI entrypoint and output renderer
- `portfolio_ask/schemas.py`: `GeneralQA` and `NewsImpact` response contracts
- `portfolio_ask/__main__.py`: CLI wrapper entrypoint
- `evals/cases.yaml`: fixed test cases
- `evals/run_eval.py`: schema-driven evaluation harness

---

## Requirements

- Python 3.11+
- `uv` installed
- model pulled in Ollama: `llama3.1`

---

## Quick start (one-command workflow)

### 1) Setup environment and dependencies

Run:

`make setup`

### 2) Generate mock data

Run:

`make data`

This creates:

- `data/portfolio.json` (16 holdings — equity, ETF, bond, mutual fund)
- `data/news/*.md` (20 files)
- `data/glossary.md` (wealth-tech terms)

### 3) Run a single query

Run:

`make run QUERY="What is my tech exposure?"`

Example direct CLI usage:

`python3 -m portfolio_ask "What news is likely to affect my portfolio?" --trace`

### 4) Run evaluation suite

Run:

`make eval`

The eval harness validates schema shape, checks fallback behavior, prints trace for flagged cases, and exits non-zero on failures.

---

## Output contract

The system returns one of two strict JSON schemas:

### `GeneralQA`

- `query_type: str`
- `answer: str`
- `ranked_items: list[dict]`
- `sources: List[str]`
- `trace: List[str]`

### `NewsImpact`

- `query_type: str`
- `summary: str`
- `ranked_items: List[{ticker, rationale, exposure_weight, risk_level}]`
- `sources: List[str]`
- `trace: List[str]`

This structure is enforced via Pydantic and is used in evaluation assertions.

---

## Deterministic fallback behavior

If Ollama is unavailable, or if fallback is forced in eval mode, the system uses Data-Aware Graceful Degradation.

Current heuristic:

- for queries containing “risk”, fallback filters portfolio to FMCG or Bond holdings and returns a valid typed shape.
- if the LLM crashes, the system extracts the available tickers from the current state and still shows the user actionable holdings data instead of a generic error.

This guarantees stable behavior in offline or credential-free reviewer environments.

---

## What “hallucination” means in this RAG CLI

In this project, a hallucination means:

> The model outputs a confident statement that is not grounded in retrieved documents, portfolio data, or defined glossary entries.

How this implementation reduces hallucination risk:

- retrieval-first design (context loaded before response generation),
- source file propagation (`sources` field),
- strict structured output with Pydantic validation,
- deterministic fallback path when model behavior is unreliable,
- eval checks that reject malformed or schema-violating responses.

This does not make hallucinations mathematically impossible, but it narrows failure modes and makes them easier to detect.

---

## Evaluation design

`evals/cases.yaml` defines exactly five cases:

1. General portfolio query
2. Risk/fallback query
3. News impact query
4. Glossary query
5. Edge out-of-scope query

At least three cases print execution traces to inspect graph behavior.

`evals/run_eval.py`:

- loads YAML,
- runs agent per case,
- validates against expected Pydantic schema,
- checks forced fallback branch,
- supports surgical testing with `--case` so one failed case can be run by itself,
- prints PASS/FAIL summary,
- raises assertion if any test fails.

---

## Notes

- This repo is designed to be readable first, then extensible.
- The CLI intentionally favors deterministic JSON outputs for auditability.
