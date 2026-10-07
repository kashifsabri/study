
This is a **suggested** layout. The roadmap doesn't fix file names, so this is a design decision. Log it in `DECISIONS.md` once you accept it.

One rule to remember: **create a file only when its phase arrives.** Don't build the whole tree now. Memory, RAG and the scheduler are deferrable, so those folders don't exist until you decide you need them.

---

## The tree

```
agent_core/
├── DECISIONS.md                  one line per architectural decision, with the reason
├── PROGRESS.md                   one line per phase, plus the failure you  triggered by hand
├── pyproject.toml
│
├── src/agent_core/
│   │
│   ├── contracts/                P1 · pure data. No logic. Imports nothing from agent_core.
│   │   ├── ids.py                tenant_id, principal, run_id, step_id
│   │   ├── run.py                Agent, AgentVersion, Session, Turn, Run, Step
│   │   ├── events.py             AgentEvent envelope (schema_version), all event types
│   │   ├── fragments.py          ContextFragment (instruction_authority, content_trust)
│   │   ├── model.py              ModelRequest, ModelResponse
│   │   └── actions.py            ActionProposal, ToolCall, ToolResult, Observation
│   │
│   ├── model/                    P1, P2, P3 · owns the vendor
│   │   ├── provider.py           ModelProvider interface: generate(), generate_structured()
│   │   ├── groq_provider.py      P2 · the Groq implementation. The only file that imports groq.
│   │   ├── fake_provider.py      scripted model for tests and demos, costs nothing
│   │   ├── retry.py              P3 · backoff, 429 handling
│   │   └── repair.py             P3 · malformed-output repair
│   │
│   ├── tools/                    P4 · owns "what tools exist and how they run"
│   │   ├── definition.py         ToolDefinition: effect, idempotency, reversibility, recovery (derived)
│   │   ├── registry.py           ToolRegistry
│   │   ├── validator.py          ToolValidator: does the proposal have the right shape?
│   │   ├── executor.py           ToolExecutor: timeout, cancellation, passes the dedup key
│   │   ├── errors.py             ToolError taxonomy
│   │   └── builtin/              fixtures
│   │       ├── calculator.py
│   │       ├── weather.py
│   │       ├── search.py
│   │       └── read_file.py
│   │
│   ├── policy/                   P1 seam, P28 fills it · owns "is this allowed?"
│   │   └── gate.py               PolicyGate interface + NoopPolicyGate
│   │
│   ├── context/                  P1 envelope, P18 engine · owns "what the model sees"
│   │   ├── builder.py            ContextBuilder: picks fragments from state
│   │   ├── renderer.py           THE choke point: refuses data_only in instruction regions
│   │   └── allocator.py          P18 · TokenBudgetAllocator
│   │
│   ├── state/                    P5 · owns "where is this run"
│   │   └── projection.py         AgentState, built by replaying events
│   │
│   ├── store/                    P6 · owns "the truth"
│   │   ├── event_log.py          append-only EventLog interface
│   │   ├── sqlite_log.py         first implementation
│   │   └── migrations/           tenant_id is in the first one
│   │
│   ├── budget/                   P8
│   │   └── ledger.py             reserve, settle, debit events
│   │
│   ├── runtime/                  the HUB. The only package allowed to call the others.
│   │   ├── loop.py               P2 · the circle: build context, call model, gate, execute
│   │   ├── lifecycle.py          P7 · state machine, transition table
│   │   ├── modes.py              P6 · REPLAY, RESUME, EVAL as separate code paths
│   │   ├── recovery.py           P6, P9 · intent with no outcome → recovery path
│   │   └── failures.py           P9 · error classes and the recovery matrix
│   │
│   ├── observability/            P2 starts it, P10 grows it
│   │   └── trace.py              spans, correlation ids, cost and latency
│   │
│   ├── evals/                    P11
│   │   ├── golden.py             record runs from the log, replay in EVAL mode
│   │   ├── fixtures.py           model and tool fixtures, failure injection
│   │   └── metrics.py
│   │
│   ├── agents/
│   │   └── target_agent.py       your one named target agent (locked at P0)
│   │
│   └── api.py                    P1 · the public embedding API. The only thing callers import.
│
├── tests/                        mirrors src/. Locks in what you verified by hand.
│   └── injection/                "ignore your previous instructions" fixtures, kept forever
│
├── scripts/                      manual checks you run at a terminal
│   └── kill_mid_run.py           P6 · kill -9 at three points
│
└── lessons/                      the toy files from these notes. Reference only, never imported.
    ├── gap.py                    from 3.3
    └── concerns.py               from 3.4
```

Deferred, so **not created yet**: `memory/` (P22), `knowledge/` (P23), `scheduler/` (P17), `planning/` (Stage C, gated on P11).

---

## The one rule that keeps it clean: who may import whom

This is the hub-and-spoke rule from 3.2, written as imports.

```
                    contracts/        ◄── everyone may import this. It imports nobody.
                        ▲
   ┌──────────┬─────────┼─────────┬──────────┬─────────┐
 model/     tools/   policy/   context/   state/    budget/   store/
   │          │         │         │          │         │         │
   └──────────┴─────────┴────┬────┴──────────┴─────────┴─────────┘
                             ▼
                         runtime/   ◄── imports all of them. Nobody imports runtime.
                             ▼
                          api.py
```

|Package|May import|Must NOT import|
|---|---|---|
|`contracts/`|The standard library only|Anything else in `agent_core`|
|`model/`, `tools/`, `policy/`, `context/`, `state/`, `budget/`, `store/`|`contracts/`|Each other, and `runtime/`|
|`runtime/`|Everything above|`api.py`|
|`api.py`|`runtime/`, `contracts/`|The spokes directly|

Check yourself with one line: if `tools/executor.py` has `from agent_core.model import ...`, that's Break 1 from 3.2 (a tool secretly calling the model). Later you can enforce this with a tool like `import-linter`, so the rule is checked and not just remembered.

---

## Where each piece of the toy code from these notes goes

|Toy code from the notes|Real home|Notes|
|---|---|---|
|`fake_model(...)`|`model/fake_provider.py`|Implements `ModelProvider`|
|`model_provider(...)` returning `Proposal`|`model/provider.py` + `groq_provider.py`|Only this package knows the vendor|
|`Proposal` dataclass|`contracts/actions.py` as `ActionProposal`|Pure data|
|`build_context(...)`|`context/builder.py`|Chooses fragments|
|`render(...)` with `[DATA ONLY]`|`context/renderer.py`|The one choke point|
|`TOOLS`, `TOOL_SPECS`|`tools/registry.py`|Descriptions go to context, code stays here|
|`get_weather`, `refund`|`tools/builtin/weather.py` and so on|Effect only, no policy|
|`gate()` shape checks (known tool, arg types)|`tools/validator.py`|P4|
|`gate()` limit check (`amount <= 100`)|`policy/gate.py`|Rule lives here and nowhere else|
|`POLICY = {...}`|`policy/gate.py` now, agent's granted authority at P19|Open question from 3.4|
|`LOG`, `emit(...)`|`store/event_log.py`|Append-only|
|`state = {...}` dict|`state/projection.py`|Derived from events, never edited by hand|
|`execute(...)`|Split: `runtime/loop.py` calls `tools/executor.py`|Loop writes `ToolCallIntended` first|
|`resume(...)`|`runtime/recovery.py`|Intent with no outcome|
|`MEMORY`|Not yet|Deferred to P22|
|`max_steps`|`budget/ledger.py`|Replaces the counter at P8|
|`EDGES` / `call(...)` trace|`observability/trace.py`|Grows into P10|

---

## What exists after each phase

Use this as your checklist so you never build ahead.

|After phase|Files that now exist|
|---|---|
|P0|`DECISIONS.md`, `PROGRESS.md`, `agents/target_agent.py` (spec only)|
|P1|everything in `contracts/`, `model/provider.py`, `policy/gate.py` (no-op), `api.py` (shell)|
|P2|`model/groq_provider.py`, `model/fake_provider.py`, `runtime/loop.py`, `observability/trace.py`|
|P3|`model/retry.py`, `model/repair.py`|
|P4|all of `tools/`, `tests/injection/`|
|P5|`state/projection.py`|
|P6|`store/`, `runtime/modes.py`, `runtime/recovery.py`, `scripts/kill_mid_run.py`|
|P7|`runtime/lifecycle.py`|
|P8|`budget/ledger.py`|
|P9|`runtime/failures.py`|
|P10|`observability/trace.py` grows|
|P11|`evals/`|
|P18|`context/allocator.py`, `context/renderer.py` hardened|

---

## Two things to decide before P1

1. **Package name and layout.** `src/agent_core/` (the `src` layout) stops you from accidentally importing your working folder instead of the installed package. It's a small habit that avoids a confusing class of bugs.
2. **Where `lessons/` lives.** Keep it outside `src/`. It's for reading and breaking, and the real code must never import from it.

Log it in `DECISIONS.md`:

```
Layout: src/agent_core with contracts/ at the bottom and runtime/ as the only hub; spokes never import each other — makes the 3.2 hub-and-spoke rule a checkable import rule.
```