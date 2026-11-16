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

## Framework structure (names and purpose only; contents are private)

```
triage-agent/
|-- run_test_and_triage.py     launcher: run tests -> parse -> triage over A2A -> report + exit code
|-- mcp_server.py              MCP front door exposing the parsing / pattern / report tools
|-- conftest.py                test gate: no real provider call without explicit opt-in
|-- pytest.ini                 registers the "live provider" marker
|
|-- a2a_service/               A2A front door and everything that runs around it
|   |-- server.py              A2A service: agent card, request parsing, result + provenance
|   |-- client.py              A2A client, service start/stop, task-state and artifact rules
|   |-- validation.py          caller-side checks on every answer
|   |-- pipeline.py            reports -> parser -> A2A -> validation -> output, outcomes, narration
|   |-- evidence.py            plain-language description of how an answer was reached
|   |-- agent_route.py         optional bounded LLM investigator (budget guard, tool gate)
|   |-- agent_validation.py    deterministic checks on the model's part of an answer
|   |-- agent_plan.py          plan-first approval for the one-agent route
|   |-- agent_export.py        records exactly what a run produced, for later evaluation
|   |-- coordinator.py         two-agent mode: coordinator agent, shared budget, exchange state
|   |-- two_agent_plan.py      plan-first approval for the two-agent route (fingerprints, caps)
|   |-- heal.py                optional heal stage: one bounded action + one isolated rerun
|   |-- evaluate.py            runs the frozen labelled cases through the real route
|   |-- demo.py / demo_two_agent.py   offline demos (the second uses scripted models)
|   `-- README.md, design and safety notes
|
|-- agents/
|   |-- grounded_triage.py     evidence-grounded classification with abstention
|   |-- triage_agent.py        ReAct agent + deterministic local path
|   `-- recovery_investigator.py   optional tool-selecting recovery agent
|
|-- knowledge/
|   |-- corpus/                the internal notes (private)
|   |-- ingestion/             chunker, indexer, integrity (drift) check, fallback embedding
|   |-- retrieval/             hybrid search, rank fusion, reranker
|   `-- access_control.py      role -> clearance filtering applied before ranking
|
|-- tools/                     parser, known-issue matcher, diagnostics, remediation,
|                              report builder, execution trace, grounded-triage tool
|-- adapters/surefire_ingest.py   normalises real test-report failures
|-- config/llm_client.py       provider selection, output caps, one pre-tool outage fallback
|-- evals/                     frozen labelled sets, reconciliation, retrieval metrics, judge checks
`-- tests/                     offline suite, no provider calls
```

## Class reference: purpose and pseudocode

The pseudocode is written for this page to show each idea. It is not the project source.

### Service layer

**`GroundedTriageExecutor`**: A2A executor that answers one triage request, read-only.
```
receive request -> parse {message, trace}
result = grounded_triage(request)
return artifact { result, provenance, cited note text }
```

**`LlmAgentTriageExecutor`**: same protocol, adds an optional bounded LLM investigator.
```
base = deterministic_result(request)
agent_block = run_bounded_agent(request)        # may fail; base answer stays the answer
return base + agent_block
```

**`ServiceProcess`**: starts a local service for one run and stops only that process.
```
port = ask_os_for_free_port()
proc = launch(service, port, no_provider_keys)
wait_until_healthy(proc)
on exit: stop(proc)                             # never touches other processes
```

**`Check` / `validate`**: the caller's independent checks on an answer.
```
checks = [bound_to_request, citations_exist, cited_text_matches_note,
          category_agrees_with_note, no_fix_claimed]
return list of (name, passed)                   # failures reported, never hidden
```

**`BudgetGuard`**: stops an agent loop in code, not in the prompt.
```
before each model call:
    if turns >= limit or past_deadline or cancelled: raise stop
    turns += 1
before each tool call:
    if tools >= limit: raise stop
```

**`Toolbox`**: the only tools an agent can reach.
```
allowed = {read-only tools}
call(name, args):
    if name not in allowed: refuse
    return thin_wrapper(existing_tool)(args)
```

**`SharedBudget` / `RunSpend` / `CoordinatorGuard`**: limits shared across two agents.
```
before each model call by EITHER agent:
    reserve worst-case output tokens
    if calls, tokens, time or estimated spend would exceed the shared cap: stop
after call: record usage, flag unknown usage
```

**`ExchangeState`**: what the harness knows about the two-agent conversation.
```
record every request, answer, delegation and usage
facts = computed_from_record(), never from what a model claims
```

**`Narrator`**: makes a run readable by a non-expert.
```
print "<icon> sender -> receiver: what was said"; keep a plain-text copy
```

**Heal stage**: optional, after a validated answer.
```
if answer accepted and cause is fixable and action really changed something:
    rerun ONE test in isolation, read the fresh report
    label result "passed alone" (never "fixed"); else "human needed" + reason
```

**Plan builders**: approval before any spend.
```
plan = { providers, data sent, call limits, cost cap,
         fingerprints(prompts, tools, knowledge, code, inputs) }
plan_id = hash(plan)
run only if the supplied plan_id matches and used bytes == approved bytes
```

### Knowledge and retrieval

**`Chunk` / `chunk_document`**: sentence-aware, token-budgeted splitting.
```
for sentence in document: add to current chunk until token budget; then start next
```

**`build_index`**: builds and saves both halves of the hybrid index.
```
chunks = chunk(corpus); dense = embed(chunks); sparse = tokenize(chunks)
save(dense, sparse, corpus_fingerprint)
```

**`HybridIndex`**: runs dense and sparse search side by side.
```
dense_list  = nearest(query_vector)
sparse_list = keyword_rank(query_tokens)
return fuse([dense_list, sparse_list])
```

**`reciprocal_rank_fusion`**
```
score[item] = sum over lists of 1 / (k + rank_in_list)
return items sorted by score
```

**`Reranker`**: late-interaction rerank against the real query.
```
for candidate: score = sum over query tokens of best match among candidate tokens
return candidates sorted by score
```

**`TinyTfidf`**: dependency-free embedding fallback.
```
fit term weights on corpus; vector(text) = normalised weights
```

**Integrity check**: catches a stale index.
```
if fingerprint(corpus now) != fingerprint(corpus at index time): report "stale"
```

**Access control**: restricted material never enters ranking.
```
allowed = clearances_for(role)
candidates = [c for c in all if c.clearance in allowed]    # filter BEFORE scoring
```

### Agents, tools and config

**Grounded triage**: classify with evidence, or abstain.
```
if specific keyword rule matches: answer from rule
else: candidates = retrieve(failure)
      if best is clearly better than runner-up: answer with citation
      else: UNCERTAIN (look-alike causes)
```

**ReAct triage agent**: model chooses tools, with a no-model path.
```
if model available and allowed: loop { think -> pick tool -> observe } until report
else: run_scripted_path()                       # same job, no model
```

**`AgentExecutionTrace` / `ToolCallRecord`**: stable record of one run.
```
trace = [ (tool, outcome) ... ] extracted in one place, so evals never touch library internals
```

**LLM client**: provider selection with explicit switching.
```
provider = chosen_by_config                     # never silently switched
on outage BEFORE any tool ran, with a second key present: retry once, say so loudly
```

**Surefire adapter**
```
for each failed or errored test case in the reports: emit normalised {test, message, trace}
```

**MCP server**
```
expose tools: parse_reports, check_known_patterns, reset_state, check_health, rerun, report
```

**Test gate (`conftest.py`)**
```
if test is marked live_provider and no explicit opt-in: skip
a key alone never enables it; real network transports are also blocked
```

### Evaluation

**Route evaluation / reconciliation**
```
for case in frozen_cases: run through real client + service; save answer + validation
account for EVERY case in exactly one bucket; none dropped from a denominator
```

**Judge sensitivity check**
```
feed good and deliberately flawed explanations to the judge; see whether scores separate
treat the metric as advisory, not a gate
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
