# Agentic AI — Universal Curriculum

  

A complete, framework-independent curriculum for understanding and building agentic AI systems from scratch. Works regardless of what you are building, what stack you use, or what framework you choose.

  

Concepts first. Implementation second. Frameworks last.

  

---

  

## The progression

  

```

Topics 1–3      What an agent is

Topics 4–7      How it is built

Topics 8–11     How it stays safe

Topic 12        How it survives failure     ← hardest, spend the most time

Topics 13–14    How it improves

                ↑ Every serious agent needs everything above this line ↑

Topics 15–18    How it reasons

Topic 19        How it manages context

Topics 20–23    How it scales

Topics 24–25    How it knows things

Topics 26–28    How it reaches production

Topic 29        How frameworks relate to it

Topic 30        How multiple agents work together

Topics 31–36    Advanced production patterns

```

  

---

  

## Topic 1 — Foundations of Intelligent Agents

  

1. What is an agent — definition, properties, boundaries

2. Agent vs chatbot vs workflow vs automation

3. Types of agents — reactive, deliberative, goal-based, learning

4. Where language models fit in an agent

5. What a language model can and cannot do

6. Why a language model alone is not an agent

7. The environment — what the agent perceives and acts on

8. Perception, reasoning, action — the three responsibilities

9. Real world examples — coding agents, support agents, research agents, browser agents

  

---

  

## Topic 2 — The Agent Loop

  

1. Why agents run in a loop and not a pipeline

2. The minimal loop — perceive, reason, act

3. The ReAct loop — reason, act, observe, repeat

4. What a step is

5. What an observation is and why it matters

6. How the loop terminates — goal met, budget exhausted, failure

7. Why the model proposes and the runtime executes — never the other way

8. Reading a real agent trace end to end

  

---

  

## Topic 3 — Agent Architecture

  

1. The core components every agent must have — loop, model, tools, context, state, memory

2. How components relate to each other

3. The proposal-execution gap — the most important architectural idea in agent design

4. Separation of concerns — what belongs where

5. Agent boundaries — what is inside the agent and what is outside

6. Designing for replaceability — why every component should be swappable

7. The runtime as the authority — the model only suggests, the runtime decides

  

---

  

## Topic 4 — Domain Modelling and Contracts

  

1. Why domain modelling comes before implementation

2. The ownership hierarchy every agent system needs:

   ```

   Agent

     └── AgentVersion

           └── Session

                 └── Turn

                       └── Run

                             └── Step

   ```
   

3. What each entity owns and what it references

4. Value objects within a step — Context, ModelRequest, ModelResponse, ActionProposal, PolicyDecision, ToolCall, ToolResult, Observation

5. Input schema and output schema — every contract has both, always

6. Why contracts you persist are different from classes you instantiate

7. Schema versioning — once you persist a contract its shape is permanent

8. Writing contracts as code — interfaces, dataclasses, typed models

9. Stability — why core contracts must be frozen early

  

**The contract tree in full:**

```

Agent

  └── AgentVersion (immutable — instructions, model, tools, limits)

        └── Session (one conversation)

              └── Turn (one user message + agent response)

                    └── Run (one execution attempt)

                          └── Step (one loop iteration)

                                ├── Context

                                │     └── ContextFragment (one piece of context)

                                ├── ModelRequest

                                ├── ModelResponse

                                └── ActionProposal

                                      ├── [tool_call]

                                      │     → PolicyDecision

                                      │       → ToolCall → ToolResult → Observation → next Step

                                      └── [final_answer]

                                            → Run completes

```

  

**Key schemas:**

  

```

Agent

  id, name, description, tenant_id, created_at

  

AgentVersion

  id, agent_id, version_number (immutable), instructions,

  model_config (provider, model_id, temperature, max_tokens),

  tool_ids[], limits (max_steps, max_tokens, max_cost, max_wall_seconds),

  context_config, granted_authority, tenant_id

  

Session

  id, agent_id, version_id, principal (user_id, tenant_id, role), created_at

  

Turn

  id, session_id, index, user_input, created_at

  

Run

  id, turn_id, session_id, agent_id, version_id, principal,

  status (CREATED|RUNNING|WAITING|PAUSED|COMPLETED|FAILED|CANCELLED),

  created_at, completed_at, final_answer, error, tenant_id

  

Step

  id, run_id, index (logical — for replay), wall_clock,

  status (RUNNING|COMPLETED|FAILED), created_at

  

ContextFragment

  fragment_id, source, instruction_authority (can_instruct|data_only),

  content_trust (trusted|semi_trusted|untrusted),

  content_hash, content, captured_at, token_count, metadata

  

ModelRequest

  id, step_id, run_id, provider, model_id, messages[],

  tools[], temperature, max_tokens, created_at

  

ModelResponse

  id, request_id, step_id, run_id, provider, model_id, model_version,

  content[], stop_reason, usage (input_tokens, output_tokens, total_tokens),

  cost_usd, latency_ms, created_at

  

ActionProposal

  id, step_id, run_id, response_id,

  type (TOOL_CALL|FINAL_ANSWER),

  tool_name, tool_args, final_answer, reasoning, created_at

  

PolicyDecision

  id, proposal_id, run_id,

  decision (APPROVED|REJECTED|MODIFIED),

  reason, amended_args, decided_by, decided_at

  

ToolCall

  id, step_id, run_id, decision_id, tool_name, tool_version,

  args (validated against input_schema), dedup_key, started_at, principal

  

ToolResult

  id, tool_call_id, run_id,

  status (SUCCESS|ERROR|TIMEOUT|CANCELLED|UNKNOWN),

  output (validated against output_schema), error, completed_at, latency_ms

  

Observation

  id, step_id, run_id, tool_call_id,

  source (TOOL_RESULT|MODEL_RESPONSE|HUMAN_INPUT|SYSTEM),

  fragment (ContextFragment), recorded_at

```

  

---

  

## Topic 5 — Model Interaction

  

1. What a ModelProvider is and why it exists

2. The generate interface — text generation

3. generate_structured — typed output from the model

4. decide — fast structured decisions for classification and routing (Topic 36)

5. Why structured output is a reliability problem, not a parsing problem

6. Malformed output — detection, repair, rejection

7. Transient failures and retries with backoff

8. Rate limits and backpressure

9. Token counting and cost tracking as first-class response fields

10. Latency measurement per call

11. Model identity and version — always record which model answered

12. Provider abstraction — swap models without touching the loop

  

**ModelProvider interface:**

```

generate(messages, tools, config) → ModelResponse

generate_structured(messages, output_schema, config) → typed output

decide(state, questions) → structured decisions with confidence

```

  

---

  

## Topic 6 — Tool Design and the Tool System

  

1. What a tool is — a capability the agent can invoke

2. Tool definition — name, description, input schema, output schema

3. JSON Schema — validating every input before execution

4. The tool registry — how the agent discovers available tools

5. Tool execution — the full lifecycle of a single tool call

6. Tool results and tool errors

7. Error taxonomy — validation error, timeout, failure, unknown outcome

8. Timeouts and cancellation

9. Tool versioning

10. Side effect classification:

    ```

    effect        read | create | update | delete | external

    idempotency   natural | client_key | none

    reversibility reversible | compensatable | irreversible

    recovery      derived from the three above — never declared independently

    ```
    

11. Dedup keys — passed to every effectful tool, preventing double execution

12. Tool lifecycle events

13. Writing good tool descriptions — written for a model to read, not a human

  

**ToolDefinition schema:**

```

id, name, version, description,

input_schema (JSON Schema),

output_schema (JSON Schema),

effect, idempotency, reversibility, recovery (derived),

timeout_ms, tenant_id

```

  

**Recovery matrix (derived):**

```

READ + NATURAL + REVERSIBLE         → retry_safe

CREATE + CLIENT_KEY + REVERSIBLE    → retry_safe

CREATE + NONE + IRREVERSIBLE        → manual

DELETE + NONE + IRREVERSIBLE        → manual

EXTERNAL + CLIENT_KEY + COMPENSATABLE → reconcile

```

  

---

  

## Topic 7 — Trust, Authority and Safety

  

1. Why trust is an architectural constraint, not a security feature

2. The confused deputy problem — when the agent is tricked into acting on bad data

3. Instruction authority vs content trust — two separate fields, not one level

4. The authority table:

   ```

                      instruction_authority    content_trust

   system             can_instruct             trusted

   operator           can_instruct             trusted

   user               can_instruct             semi_trusted

                      (within granted limits)

   tool output        data_only                untrusted

   retrieved knowledge data_only              untrusted

   memory             data_only                untrusted

   ```
   

5. Why tool output is data only and can never carry instructions

6. Prompt injection — what it is, how it works, why it is a design constraint not a patch

7. The single enforcement choke point — one place in the code, the context renderer

8. Designing injection resistance in from day one

9. The injection test — a fixture that returns "ignore your previous instructions" in tool output, kept in the test suite forever

  

---

  

## Topic 8 — Agent State

  

1. What state a running agent needs

2. Core state fields — goal, current step, observations, tool results, errors, budget, answer

3. State as a projection — derived from events, not the source of truth

4. Serialisation and deserialisation

5. Two clocks — wall time and logical step index

6. Why the logical clock matters for replay and the wall clock does not

7. Checkpoints as a performance optimisation, never as truth

  

**AgentState schema:**

```

Derived from the event log — never stored as primary data.

  

run_id, status, goal, current_step (logical index),

observations[], tool_results[], errors[],

budget (steps_used/limit, tokens_used/limit, cost_used/limit, wall_ms_used/limit),

final_answer, logical_clock, wall_clock

```

  

---

  

## Topic 9 — Lifecycle and State Machine

  

1. Why a formal lifecycle prevents entire classes of bugs

2. The seven states — CREATED, RUNNING, WAITING, PAUSED, COMPLETED, FAILED, CANCELLED

3. What each state means and when it applies

4. The full valid transition table:

```
   CREATED   → RUNNING

   RUNNING   → WAITING | PAUSED | COMPLETED | FAILED | CANCELLED

   WAITING   → RUNNING | CANCELLED | FAILED

   PAUSED    → RUNNING | CANCELLED

   COMPLETED → (terminal)

   FAILED    → (terminal)

   CANCELLED → (terminal)


```

   ```
   

5. Enforcing transitions — illegal ones rejected at write time

6. Cancellation semantics — what happens to in-flight work

7. Pause and resume

8. Terminal states and what can never happen after them

  

---

  

## Topic 10 — Budgets and Resource Control

  

1. Why every agent needs hard limits — an agent without budgets is a liability

2. What to budget — steps, model calls, tokens, cost, wall time, tool calls, retries

3. Why a counter breaks in a system that replays events

4. Reserve-then-settle — the correct pattern

5. Budget debits as events — spending recorded, not mutated

6. Budgets surviving restart

7. BudgetExceeded as a first-class failure, not a special case

8. Budget policies — per agent version, per run, per user

  

**BudgetState schema:**

```

Derived from BudgetConfig (limits) + BudgetDebited events (spends).

  

Per dimension: used, limit, remaining, exceeded (bool)

Dimensions: steps, model_calls, tokens, cost_usd, wall_seconds, tool_calls, retries

```

  

---

  

## Topic 11 — Failure Taxonomy and Recovery

  

1. Why failure must be part of the design, not handled after the fact

2. Categories — model failures, tool failures, budget failures, infrastructure failures

3. The full taxonomy:

   - ModelTransientError

   - ModelInvalidOutput

   - ToolValidationError

   - ToolTimeout

   - ToolFailure

   - UnknownToolOutcome

   - BudgetExceeded

   - LeaseLost

   - RunCancelled

4. The recovery matrix — written as a table, not prose:

   ```

   Error class              Recovery policy

   ModelTransientError      retry with backoff

   ModelInvalidOutput       retry with repair

   ToolValidationError      fail (bad proposal from model)

   ToolTimeout              retry or reconcile (per tool policy)

   ToolFailure              retry or escalate (per tool policy)

   UnknownToolOutcome       reconcile or manual (per tool policy)

   BudgetExceeded           fail

   LeaseLost                requeue

   RunCancelled             fail

   ```

5. Retry vs retry-with-repair vs replan vs escalate vs fail — when each applies

6. Unknown outcomes — the hardest case, and the one that actually bites

7. Injecting every error class by hand and watching each recovery path run

  

---

  

## Topic 12 — Durable Execution

  

> The hardest topic in the curriculum. Spend the most time here. Everything after it depends on getting this right.

  

1. The durability problem — what happens when a process dies mid-execution

2. The crash question — you called a tool, the process died, did it happen?

3. The event log — append-only, ordered, the one source of truth

4. Events vs state — why events are primary and state is derived

5. The full event set:

   ```

   RunCreated, StepStarted, ContextAssembled,

   ModelRequested, ModelResponded, ActionProposed, PolicyDecided,

   ToolCallIntended, ToolCallCompleted, UnknownOutcome,

   ObservationRecorded, StepCompleted, BudgetDebited,

   RunCompleted, RunFailed, HumanInputRequested, HumanInputReceived

   ```

6. Intent before effect — ToolCallIntended written before execution

7. UnknownOutcome as a first-class event, not an exception

8. State as a projection of the event log

9. Checkpoints as a performance optimisation only

10. REPLAY — reconstruct from events, no model call, no tool call, ever

11. RESUME — reconstruct history, then continue with new steps

12. EVAL — replay recorded model and tool responses against new code, no network

13. Why these three modes must be separated in code, not just in documentation

14. Event schema versioning — a version field on every event, with an upcast strategy

15. Identity fields — tenant and principal on every event

16. kill -9 mid-run at three different points, restart, reconstruct, explain each case

  

**EventEnvelope schema:**

```

event_id, schema_version, event_type, run_id, step_id,

tenant_id, principal, occurred_at, logical_clock, payload

```

  

**Key event payloads:**

```

ToolCallIntended    tool_call_id, tool_name, tool_version, args, dedup_key

ToolCallCompleted   tool_call_id, result_id, status, latency_ms

UnknownOutcome      tool_call_id, dedup_key, recovery_policy

BudgetDebited       dimension, amount, remaining

RunCompleted        final_answer, steps_taken, total_cost_usd, total_tokens, wall_ms

RunFailed           error_code, error_message, recoverable

```

  

---

  

## Topic 13 — Observability and Tracing

  

1. Why observability is not optional in agentic systems

2. The difference between logging, tracing and metrics

3. Traces and spans — the model for capturing what happened

4. Correlation IDs — connecting events across every component

5. What to capture — model calls, tool calls, tokens, cost, latency, errors, state transitions

6. Structured traces vs unstructured logs

7. The observability bar — answer "why did it do that?" from the trace alone, source closed

8. Cost and latency per run, per step, per component

9. Connecting traces to events in the durable log

10. OpenTelemetry — emit in a standard format so any tool can consume the traces

  

---

  

## Topic 14 — Evaluation

  

1. Why you cannot safely improve an agent you cannot measure

2. The difference between testing and evaluation

3. Golden tasks — inputs with expected outputs, what makes a good one

4. Tool fixtures — deterministic tool responses for testing

5. Model fixtures — deterministic model responses for testing

6. Failure injection — deliberate breaking as a permanent part of the test suite

7. Golden runs — record a real run, replay against new code, zero tokens, deterministic regression

8. Metrics — completion rate, tool correctness, loop rate, step count, cost, recovery rate

9. Regression evaluation — catching when a change breaks something that worked

10. How to know if a change made the agent better or just different

11. Trajectory evaluation — was the sequence of actions correct, not just the final answer

12. LLM-as-a-judge — using a model to evaluate outputs

13. Judge reliability and bias — when to trust the judge and when not to

14. Offline evaluation vs online evaluation — the difference and when each applies

  

---

  

> **Foundation complete. A student who has mastered Topics 1–14 can build a durable, budgeted, observable, evaluable agent that survives process restart. This is the foundation every serious agent system is built on. Everything after this point is advanced.**

  

---

  

## Topic 15 — Goal Understanding

  

1. Why the raw user request is not enough

2. Intent extraction — what the user actually wants

3. Constraint identification — what limits the solution

4. Context gathering — what the agent already knows

5. Success criteria — how to know when the goal is met

6. Ambiguity detection — when to ask vs when to proceed

7. Structured goal representation

  

**GoalAnalysis schema:**

```

Input:  user_input, conversation_history, agent_context

Output: intent, constraints, context, success_criteria, ambiguities, confidence

```

  

---

  

## Topic 16 — Task Decomposition

  

1. What decomposition is and when it actually helps

2. Subtasks — breaking a complex goal into smaller pieces

3. Dependencies between tasks

4. Inputs and outputs per task

5. Dependency validation — detecting cycles and missing inputs

6. When decomposition hurts — added latency, cost, and failure surface

7. The rule — only decompose when measurement proves the simpler loop cannot handle it

  

**TaskGraph schema:**

```

Input:  GoalAnalysis

Output: tasks (id, name, description, input_schema, output_schema, depends_on[]),

        dependency_graph, validation_result

```

  

---

  

## Topic 17 — Planning and Execution Strategy

  

1. What a plan is — ordered groups of tasks

2. Sequential execution

3. Parallel execution — benefits and risks

4. Concurrency problems — cancellation propagation, partial completion, compensation

5. Replanning — adapting when execution diverges from the plan

6. Plan validation before execution begins

7. ReAct as a strategy

8. Plan-and-execute as a strategy

9. Reflection and self-critique as a strategy

10. Strategy selection — choosing based on the task

11. Plans propose; the runtime stays authoritative, always

  

**ExecutionPlan schema:**

```

Input:  TaskGraph, strategy

Output: groups (sequential|parallel), steps_per_group,

        estimated_cost, estimated_steps, replan_triggers

```

  

---

  

## Topic 18 — Verification and Self-Checking

  

1. Why an agent should not blindly trust its own output

2. Output validation — does the response satisfy the goal

3. Tool result verification — is this result plausible

4. Contradiction detection — does this conflict with earlier observations

5. Goal completion verification — has the goal actually been met

6. Confidence thresholds — when to proceed vs when to retry

7. Retry triggers vs replan triggers — when each applies

  

**VerificationResult schema:**

```

Input:  observation, goal, prior_observations

Output: valid (bool), goal_met (bool), contradictions[],

        confidence, action (proceed|retry|replan|escalate)

```

  

---

  

## Topic 19 — Context Engineering

  

1. Why context is one of the hardest problems in agent design

2. What a context fragment is — the atomic unit of context

3. Fragment fields — source, authority, trust, content, hash, timestamp, metadata

4. Context sources — system instructions, agent instructions, user input, conversation history, tool results, memory, knowledge, run state

5. Context assembly — selecting and ordering fragments

6. Token budget allocation — fitting everything into the model window

7. What to leave out — prioritisation and truncation strategies

8. The authority choke point — one place in the renderer that enforces trust for all fragments

9. Content addressing — storing references not payloads

10. Explainability — being able to explain exactly what was sent to the model and why

  

**ContextPlan schema:**

```

Input:  available fragments, token_budget, priority_config

Output: selected_fragment_ids, token_allocation_per_source,

        excluded_fragment_ids (with reason per exclusion), total_tokens

```

  

**ContextBuilder output:**

```

Input:  ContextPlan, fragment store

Output: ordered list of ContextFragments for the ModelRequest

        (all data_only fragments structurally fenced from instruction regions)

```

  

---

  

## Topic 20 — Agent Definition and Versioning

  

1. What an agent definition is — the complete specification

2. Instructions, model configuration, available tools, limits, context configuration

3. Why versions must be immutable — the agent that ran last week must be reproducible

4. Pinning every run to a specific version

5. Reproducing any past run exactly

6. Granted authority — what users are allowed to do within this version

7. Version history and rollback

  

**AgentVersion schema:**

```

Input:  instructions, model_config, tool_ids[], limits, context_config, granted_authority

Output: version_id, version_number (immutable after creation),

        agent_id, created_at, tenant_id

```

  

---

  

## Topic 21 — Human in the Loop

  

1. When humans must be involved — approval, ambiguity, high-risk actions, policy

2. Human in the loop as a first-class architectural pattern, not an afterthought

3. A dedicated lifecycle state for waiting on a human

4. Human input requests as durable events

5. No worker held while waiting — the run suspends and the worker is released

6. Human response and run resumption

7. Timeout handling — what if the human never responds

8. Surviving process restart while waiting

9. Audit trail — every human decision recorded permanently

  

**HumanInputRequest schema:**

```

Input:  run_id, step_id, prompt, context, timeout_seconds

Output: request_id, status (pending|responded|timed_out), response | null, responded_by | null

```

  

---

  

## Topic 22 — Streaming and Progressive Disclosure

  

1. Why users need feedback during long-running agent tasks

2. Token streaming — sending model output as it generates

3. Step-level progress — telling the user what the agent is doing

4. Why durability and streaming are in tension — partial output has no place in an event log

5. The correct pattern — stream from the live executor, commit only settled events

6. Reconnection — replay committed events then reattach to the live stream

7. Progress events as permanent records in the event log

  

---

  

## Topic 23 — Distributed Workers and Scheduling

  

1. Why a single process eventually becomes a bottleneck

2. The scheduling problem — matching runnable runs to available workers

3. Leases — a worker's exclusive claim on a run

4. Lease expiration and renewal

5. Fencing tokens — preventing two workers from running the same thing simultaneously

6. Worker crash recovery

7. Duplicate execution protection

8. Priority queues — running important work first

9. Backpressure — what to do when more work arrives than workers can handle

  

**Lease schema:**

```

Input:  run_id, worker_id, lease_duration_seconds

Output: lease_id, expires_at, fencing_token (monotonic integer — higher always wins)

```

  

---

  

## Topic 24 — Memory Systems

  

1. The four things that look like memory but are not the same — state, conversation, memory, knowledge

2. Why they must be kept separate

3. Short-term memory — within a single run

4. Long-term memory — persisted across runs

5. Episodic memory — what happened in past interactions

6. Semantic memory — facts and concepts the agent has learned

7. MemoryStore — reading, writing, querying

8. Retrieval — finding relevant memories for the current context

9. Expiration and forgetting

10. Provenance — knowing where every memory came from

11. Memory as data only — it can never instruct

  

**MemoryRecord schema:**

```

Input:  content, memory_type, source, agent_id, principal

Output: memory_id, content_hash, embedding, created_at,

        expires_at | null, provenance, tenant_id

```

  

**MemoryQuery schema:**

```

Input:  query_text | embedding, memory_types[], limit, min_relevance

Output: list of MemoryRecord with relevance_score

```

  

---

  

## Topic 25 — Knowledge Retrieval and RAG

  

1. What RAG is — retrieval augmented generation

2. Why agents need external knowledge beyond their training

3. Document ingestion — getting knowledge into the system

4. Chunking strategies — how to split documents for retrieval

5. Embeddings — representing meaning as vectors

6. Vector search — finding relevant chunks

7. Reranking — improving retrieval quality after the initial search

8. Context integration — turning retrieved chunks into context fragments

9. Source provenance — always know where knowledge came from

10. Retrieved content is untrusted — always data only, always

11. Knowledge freshness — how old is this, and does that matter

  

**KnowledgeChunk schema:**

```

Input:  document_id, content, chunk_index, metadata

Output: chunk_id, content_hash, embedding,

        source_url | null, captured_at, expires_at | null, tenant_id

```

  

**KnowledgeQuery schema:**

```

Input:  query_text | embedding, limit, min_relevance, max_age_seconds | null

Output: list of KnowledgeChunk with relevance_score and provenance

```

  

---

  

## Topic 26 — Security and Adversarial Robustness

  

1. The threat model for agentic systems — who can attack and how

2. Prompt injection — must be tested continuously from Topic 6, not only at hardening time

3. Malicious tool output — a tool returns instructions disguised as data

4. Malformed model output — unexpected shapes, oversized responses

5. Resource exhaustion attacks — token, cost, step, time

6. Infinite loops and runaway agents

7. Data exfiltration — agent tricked into leaking information

8. Credential abuse — agent tricked into misusing its access

9. Duplicate execution attacks

10. Worker compromise

11. How to test all of the above systematically and continuously

  

---

  

## Topic 27 — Reliability Testing

  

1. The difference between unit tests, integration tests and reliability tests

2. Process kill at every meaningful point in the loop — not just once

3. Network failure injection

4. Database failure injection

5. Concurrent worker collision scenarios

6. Context overflow scenarios

7. The goal — know exactly how your system fails, not hope it handles failure gracefully

  

---

  

## Topic 28 — Production Engineering

  

1. Multi-tenancy — isolating one customer's data and execution from another completely

2. Row-level security — enforcing tenancy at the database layer, not the application layer

3. Secrets management — credentials the agent uses to call tools

4. Schema migrations — changing persisted data safely after the system is live

5. Performance — where the bottlenecks actually are in a real agent system

6. Horizontal scaling — more workers, more throughput

7. API design — how external systems talk to the agent

8. Operational runbooks — what to do when something goes wrong at 2am

9. Alerting — what signals actually matter vs what is noise

10. Cost controls — preventing runaway spend in production

  

---

  

## Topic 29 — Existing Frameworks and Ecosystem

  

1. Why you study frameworks after building from scratch, not before

2. LangChain — what it abstracts and what it hides from you

3. LangGraph — graph-based orchestration, how its checkpointing compares to event sourcing

4. LlamaIndex — knowledge and retrieval focus

5. Temporal and Restate — durable execution engines, the closest prior art to what you built

6. How to evaluate any framework — what does it give you, what does it cost you, what does it hide

7. When to use a framework and when to build

  

---

  

## Topic 30 — Multi-Agent Systems

  

1. What a multi-agent system is

2. Orchestrator and subagent patterns

3. Agent-to-agent communication

4. Trust between agents — one agent calling another is still untrusted input

5. Shared state vs isolated state across agents

6. Failure propagation — what happens when a subagent fails

7. Cost and observability across agent boundaries

8. When multi-agent helps and when it only adds complexity

  

---

  

## Topic 31 — Common Agent Architecture Patterns

  

1. Router pattern — classify input, send to the right handler

2. Planner-executor pattern — plan first, execute second, keep them separate

3. Supervisor pattern — one agent oversees and corrects others

4. Orchestrator-worker pattern — coordinator delegates to specialised workers

5. Hierarchical agents — agents managing agents managing agents

6. Event-driven agents — agents that wake up in response to external events

7. Sequential vs parallel agent workflows — when each is appropriate

8. How to choose the right architecture for the problem

  

---

  

## Topic 32 — Model Context Protocol (MCP)

  

1. What MCP is and why it exists — a standard interface between agents and the outside world

2. The three roles — host, client, server

3. MCP architecture — how the pieces connect

4. Tools — capabilities a server exposes to the agent

5. Resources — data a server makes available

6. Prompts — reusable prompt templates a server provides

7. Capability discovery — how a client learns what a server offers

8. Transports — how client and server communicate

9. Authentication and authorisation

10. Building a simple MCP server

11. Connecting an MCP client to an agent

12. MCP security — what can go wrong

13. Treating MCP tool results as untrusted — always data only, same as any tool

  

---

  

## Topic 33 — Agent-to-Agent Interoperability (A2A)

  

```

MCP   Agent → Tools / Data

A2A   Agent → Agent

```

  

1. Why independent agents need interoperability standards

2. Agent discovery — how one agent finds another

3. Capability advertisement — how agents describe what they can do

4. Task delegation — handing a subtask to another agent

5. Agent-to-agent messaging format and protocol

6. Agent identity and trust — another agent's output is still untrusted input

7. A2A protocol basics

8. MCP vs A2A — when to use which

  

---

  

## Topic 34 — Permissions, Identity and Sandboxing

  

1. Authentication vs authorisation — the difference and why it matters

2. Agent identity — who the agent is when it calls a tool

3. User identity vs execution identity — they are not the same

4. Least-privilege permissions — the agent should have only what it needs for this task

5. Tool-level permissions — different tools need different access scopes

6. Scoped credentials — credentials that expire or are limited to one action

7. Permission checks before every tool execution

8. Sandboxing untrusted execution — isolating any code the agent runs

9. Filesystem and network isolation

10. Approval gates for dangerous or irreversible actions

11. Audit trails — every privileged action recorded permanently

  

---

  

## Topic 35 — Agent Triggers and Event-Driven Execution

  

1. User-triggered agents — the simplest case, a person clicks run

2. API-triggered agents — another system starts the run programmatically

3. Webhooks — an external event causes a run to start

4. Scheduled agents — running on a timer or cron

5. Event-driven agents — reacting to events arriving on a queue

6. Queue-triggered execution — a worker pulls the next runnable item

7. Background execution — long-running work with no user waiting for a response

8. Event deduplication — what if the same trigger arrives twice

9. Idempotent event handling — processing the same trigger more than once safely

  

---

  

## Topic 36 — Production Model Optimization

  

1. Model routing — sending different kinds of requests to different models

2. Choosing models by task — classification and routing do not need the most expensive model

3. The decide interface — fast structured decisions with confidence scores for routing and verification

4. Fallback models — what to do when the primary model is unavailable

5. Prompt caching — reusing computation for repeated context

6. Semantic caching — returning cached results for semantically similar inputs

7. Tool result caching — not re-running tools whose outputs have not changed

8. Batch inference — grouping requests for efficiency

9. Cost vs latency vs quality — the tradeoff every production agent must navigate explicitly

  

---

  

## Time estimate — 3 month plan

  

Assuming 3–4 hours per day, 5 days a week. 12 weeks total.

  

```

Week 1     Topics 1, 2, 3          What an agent is and how the loop works

Week 2     Topics 4, 5             Contracts, schemas, model interaction

Week 3     Topic 6                 Tool system — larger than it looks, do not rush

Week 4     Topics 7, 8, 9          Trust boundary, state, lifecycle

Week 5     Topics 10, 11           Budgets and failure — build both and break both

Week 6     Topic 12                Durable execution — give it the full week

Week 7     Topics 13, 14           Observability and evaluation

                                   ↑ Foundation complete ↑

Week 8     Topics 15, 16, 17, 18   Reasoning — only if evaluation proves you need it

Week 9     Topics 19, 20, 21       Context engine, versioning, human in the loop

Week 10    Topics 22, 23, 24, 25   Streaming, workers, memory, RAG

Week 11    Topics 26, 27, 28       Security, reliability, production hardening

Week 12    Topics 29–36            Frameworks, patterns, MCP, A2A, optimization

```

  

### If time runs short

  

```

Must complete      Topics 1–14    Weeks 1–7

Should complete    Topics 15–21   Weeks 8–9

Complete if able   Topics 22–28   Weeks 10–11

Nice to have       Topics 29–36   Week 12

```

  

Topic 12 will take most students longer than one week. If you need ten days instead of seven, take them. Every topic after it depends on it being understood properly.

  

---

  

## Definition of done — per topic

  

A topic is not done when you finish reading it. It is done when:

  

- You can explain the core concept out loud without notes

- You have written code that demonstrates it working

- You have broken it deliberately and watched what happened

- You can answer the question a senior engineer would ask about your design choices

  

The last item is the one that matters most. Understanding that survives a hard question is the only kind worth having.