# Evidence-Grounded Test-Failure Triage Agent: architecture showcase

> **This repository is documentation only.** The implementation is private. It describes what I
> designed and built, how the parts fit together, and what I measured. The code samples below are
> short **pseudocode written for this page**. They show the pattern, they are not the
> project source, and they contain no data, prompts, thresholds or credentials.
> I'm happy to walk through the real code in an interview.

## What it is

A Python system that reads automated-test failure reports, classifies each failure against a small
internal knowledge base using hybrid retrieval, and **abstains** when the evidence is weak. The
result is served over **A2A (Agent2Agent)** and **MCP**. It also has optional, tightly bounded LLM
agent modes, including a two-agent mode where a coordinator and a specialist collaborate over A2A.
Every LLM-assisted path has a rule-based fallback that works with **no API key**.

**Stack:** Python · LangGraph · LangChain · A2A SDK · Model Context Protocol · BM25 + dense
embeddings · RRF · late-interaction reranking · pytest · Gemini / OpenAI-compatible providers.

## Architecture

```
 Surefire / TestNG XML
        |
        v
 deterministic parser  (no model reads raw XML)
        |
        v
 A2A client  ---------- A2A task ---------->  A2A service (read-only)
        ^                                       |
        |                                       v
        |                          grounded triage
        |                   keyword rule  |  hybrid retrieval  |  abstain
        |                                       |
        |                         (optional) bounded LLM agent(s)
        |                                       |
        +------ answer + citations + provenance-+
        |
        v
 caller-side validation  (re-checks everything before trusting it)
        |
        v
 report + exit code  (classification and agent outcomes kept separate)
```

## Design principles

1. **AI is an enhancer, not a dependency.** The default path calls no model. If a provider is
   missing or down, a rule-based path produces the same shaped answer.
2. **Validate before trust.** The caller re-checks every answer independently of the service and
   of any model.
3. **Bounded agents.** Model turns, tool calls, time, tokens and estimated spend are capped in
   code. Agents get read-only tools.
4. **Plan first, spend second.** A live-provider run requires an explicit, hash-bound approval
   plan and a cost cap. The default test suite cannot reach a provider.
5. **Code decides, models advise.** Final dispositions are computed by deterministic code.
6. **Honest evaluation.** Failures and wrong answers are reported, not hidden.

## Framework structure (names only; contents are private)

```
triage-agent/
|-- a2a_service/          A2A front door: server, client, caller-side validation, pipeline,
|                         bounded LLM agents, two-agent coordinator, plans, heal stage
|-- agents/               grounded triage, ReAct agent, optional recovery investigator
|-- knowledge/
|   |-- corpus/           the internal notes (private)
|   |-- ingestion/        chunker, indexer, integrity (drift) check, fallback embedding
|   `-- retrieval/        hybrid search, rank fusion, reranker
|-- tools/                parser, known-issue matcher, diagnostics, remediation, report, trace
|-- adapters/             test-report ingest (Surefire / TestNG XML)
|-- config/               LLM client: provider selection, caps, outage fallback
|-- evals/                frozen labelled sets, route evaluation, reconciliation, judge checks
|-- tests/                offline test suite (no provider calls)
|-- mcp_server.py         MCP front door
|-- run_test_and_triage.py  launcher: test run -> triage -> report
`-- conftest.py           test gate: no real provider call without explicit opt-in
```

## Components, one summary each

### Service layer (`a2a_service/`)

| Component | What it does |
|---|---|
| `GroundedTriageExecutor` | A2A executor that runs the grounded triage once per request and returns a structured result artifact with provenance. Read-only. |
| `LlmAgentTriageExecutor` | Same protocol, with an optional bounded LLM investigator added behind the same front door. |
| `ServiceProcess` | Starts a local service for one run and stops exactly that process, never anything else. |
| `Check` / `validate` | The caller's independent checks on a result: shape, citations exist, cited text matches the stored note, category agrees with the cited note, no fix claimed, bound to this request. |
| `Narrator` | Prints who said what to whom, so a run can be followed by a non-expert, and keeps a plain-text copy. |
| `BudgetGuard` | Counts model turns and tool calls and stops the loop when a budget, deadline or cancellation is hit. |
| `Toolbox` | The only tools an agent can reach, each a thin gate around an existing implementation. Write-style tools are excluded in code. |
| `SharedBudget` / `RunSpend` / `CoordinatorGuard` | Limits shared across both agents in two-agent mode: calls, tokens, time and money for the whole run. |
| `ExchangeState` | Everything the harness knows about the two-agent conversation, kept apart from what either model says. |
| Heal stage | After a validated answer, may attempt one bounded remediation and one isolated rerun. It labels a pass as "passed alone", never "fixed". |
| Plan builders | Produce the approval plan: providers, data sent, call limits, cost cap, fingerprints of prompts, tools, knowledge and code. No provider call is made while planning. |

### Knowledge and retrieval (`knowledge/`)

| Component | What it does |
|---|---|
| `Chunk` / `chunk_document` | Sentence-boundary, token-budgeted chunking of the notes. |
| `build_index` | Builds both halves of the hybrid index (dense vectors and sparse tokens) and persists them. |
| `HybridIndex` | Loads the index and runs dense and sparse search side by side. |
| `reciprocal_rank_fusion` | Merges the two ranked lists into one order. |
| `Reranker` | Late-interaction (ColBERT-style) reranking against the actual query, using persisted document vectors. |
| `TinyTfidf` | Dependency-free fallback embedding so retrieval still works if the embedding model is unavailable. |
| Integrity check | Detects drift between the notes and the built index, so stale indexes are caught instead of silently used. |
| Access control | Role-to-clearance filtering applied before ranking, so restricted material never enters a candidate list. |

### Agents, tools and config

| Component | What it does |
|---|---|
| ReAct triage agent | LangGraph agent that chooses tools, with a deterministic local path that does the same job without a model. |
| `AgentExecutionTrace` / `ToolCallRecord` | Normalised record of one agent run, so evaluation never depends on library internals. |
| LLM client | Provider selection with explicit (never silent) switching, output-token caps, and a single, loud, pre-tool outage fallback to a second provider. |
| Surefire adapter | Normalises real test-report failures into one shape. |
| MCP server | Exposes the parsing, pattern-check and reporting tools over the Model Context Protocol. |
| Test gate (`conftest.py`) | Blocks any test from calling a real provider unless the person running pytest opts in explicitly. A key alone never enables it. |

### Evaluation (`evals/`)

| Component | What it does |
|---|---|
| Frozen labelled sets | Three small, hand-labelled sets, kept immutable and hash-checked. |
| Route evaluation | Runs every case through the real client and service, not a shortcut. |
| Reconciliation | Accounts for every case, so none can be dropped from a denominator. |
| Retrieval metrics | Non-LLM ranking metrics (first-rank, top-2, MRR) on what the service actually returned. |
| Judge sensitivity check | Tests whether an LLM faithfulness judge can tell good explanations from bad ones. Used as advisory only. |

## Pseudocode (written for this page, not project source)

**1. Merge two ranked lists (rank fusion, in pseudocode)**

```python
def fuse(rankings, k=60):
    scores = {}
    for ranking in rankings:                     # e.g. [dense_order, sparse_order]
        for position, item in enumerate(ranking, start=1):
            scores[item] = scores.get(item, 0) + 1 / (k + position)
    return sorted(scores, key=scores.get, reverse=True)
```

**2. A budget guard that stops an agent loop in code, not in the prompt**

```python
class Budget:
    def __init__(self, max_turns, max_tools, deadline):
        self.turns = self.tools = 0
        self.max_turns, self.max_tools, self.deadline = max_turns, max_tools, deadline

    def before_model_call(self):
        if self.turns >= self.max_turns or time.monotonic() > self.deadline:
            raise BudgetExceeded("model-turn or time limit")
        self.turns += 1
```

**3. Validate before trust: the caller re-checks, never takes the answer on faith**

```python
def validate(answer, request, notes):
    checks = [
        ("bound_to_request", answer["request_hash"] == hash_of(request)),
        ("citations_exist",  all(c in notes for c in answer["citations"])),
        ("text_matches",     all(answer["evidence"][c] == notes[c] for c in answer["citations"])),
        ("no_fix_claimed",   not answer.get("remediation_applied")),
    ]
    return [(name, ok) for name, ok in checks]   # a failed check is reported, never hidden
```

**4. Abstain when evidence is weak**

```python
def decide(candidates):
    best, runner_up = candidates[0], candidates[1]
    if not clearly_better(best, runner_up):
        return {"category": "UNCERTAIN", "reason": "look-alike causes"}
    return {"category": best.category, "citations": [best.note_id]}
```

## Results, reported honestly

These are small, hand-labelled, **uncalibrated** sets. They are not a benchmark.

- **30 frozen cases** run through the real client and service: **22 label matches**, 6 correct
  abstentions, 5 uncertain, and **3 confidently wrong**. All three wrong answers passed structural
  validation, which shows that a valid structure is not a correct diagnosis.
- The development set (9/9) was used while tuning, so it is not a fair test. The least-tuned set
  scored 2 of 5.
- One real two-agent run completed 2 of 3 cases, and the harness rejected the third. I traced the
  cause to a gap between the specialist's check and the coordinator's request, and fixed it.
- 545 offline tests pass. None of them calls a provider.

## My role

I designed the architecture and the validation, budget and approval safeguards, ran the
evaluation, and diagnosed the failures above. I used AI coding assistants for parts of the
implementation and reviewed the results.

## What is not included, on purpose

Source code, the knowledge base, prompts, evaluation cases and labels, thresholds, run output, and
anything environment-specific. This is a showcase of design and results, not a code release.
