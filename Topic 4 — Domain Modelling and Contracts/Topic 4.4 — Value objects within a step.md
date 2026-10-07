

Think of an insurance claim file.

A customer asks for 5000. The file does **not** hold one page that gets crossed out and rewritten. It holds separate stamped documents, each written once:

1. **The claim form:** what the customer _asked for_ (5000).
2. **The adjuster's decision:** "Approved, but reduced to 100. Reason: policy limit."
3. **The payment instruction:** what the company _actually told the bank to pay_ (100).
4. **The bank's confirmation:** what the bank _reports it did_ ("100 paid").
5. **The letter back to the customer:** what the customer is _told_.

If the adjuster reduces the amount, nobody erases the claim form. A new document is added. Each document has a different author, and each refers to the one before it by reference number. Years later, anyone can rebuild exactly what happened and who decided what.

Inside one step of the agent loop, the same thing happens. The records inside a step are **write-once documents, chained by id.**

---

## Simple technical version

A **step** is one trip around the loop. Everything that happens in that trip is saved as a small record. Each record is written **once and never edited.** If something changes, you write a new record.

That's what "value object" means here: a record of one thing that happened, fixed at the moment it was written.

### The path through one step

```
Step
 │
 ├─ Context ───────────── built from ContextFragments
 │
 ├─ ModelRequest ──────── exactly what was sent to the model
 │
 ├─ ModelResponse ─────── exactly what came back
 │
 └─ ActionProposal ────── the reply, read into a clean decision
       │
       ├── [tool_call]
       │      ↓
       │   PolicyDecision ── the runtime's verdict
       │      ↓
       │   ToolCall ─────── what the runtime actually started
       │      ↓
       │   ToolResult ───── what the tool reported
       │      ↓
       │   Observation ──── how the result goes back to the model
       │      ↓
       │   next Step
       │
       └── [final_answer]
              ↓
           Run completes
```

### What each one is

|Object|Its one job|Produced by|Points back to|
|---|---|---|---|
|**Context**|The set of fragments chosen for this step's model call|Context builder|Fragment ids|
|**ContextFragment**|One piece of context, with its authority and trust labels|Whoever supplied it, labeled by the renderer|Nothing|
|**ModelRequest**|Exactly what was sent to the model|The runtime|`step_id`, `run_id`|
|**ModelResponse**|Exactly what came back, plus cost and timing|The model vendor, wrapped by the provider|`request_id`|
|**ActionProposal**|The reply read into a clean choice: call a tool, or give the final answer|The provider (parsing the response)|`response_id`|
|**PolicyDecision**|The runtime's verdict on that proposal|The runtime (`decided_by`)|`proposal_id`|
|**ToolCall**|What the runtime actually started|The runtime|`decision_id`|
|**ToolResult**|What the tool reported|The tool, i.e. the outside world|`tool_call_id`|
|**Observation**|How a result goes back into the next context|The runtime|`tool_call_id`|

Your schema gives `Context` no field list of its own. Only `ContextFragment` has one. So here `Context` means "the ordered list of fragments chosen for this step."

### Key fields worth knowing (from your schemas)

|Object|Fields that matter most|
|---|---|
|ModelRequest|`provider`, `model_id`, `messages[]`, `tools[]`, `temperature`, `max_tokens`|
|ModelResponse|`model_version`, `content[]`, `stop_reason`, `usage`, `cost_usd`, `latency_ms`|
|ActionProposal|`type` (TOOL_CALL or FINAL_ANSWER), `tool_name`, `tool_args`, `final_answer`, `reasoning`|
|PolicyDecision|`decision` (APPROVED, REJECTED, MODIFIED), `reason`, `amended_args`, `decided_by`|
|ToolCall|`tool_name`, `tool_version`, `args`, `dedup_key`, `principal`|
|ToolResult|`status` (SUCCESS, ERROR, TIMEOUT, CANCELLED, UNKNOWN), `output`, `error`|
|Observation|`source` (TOOL_RESULT, MODEL_RESPONSE, HUMAN_INPUT, SYSTEM), `fragment`|

### Why so many separate records?

Because each one is a **different fact, said by a different author.**

|Record|Whose statement is it?|
|---|---|
|ActionProposal|The model said: "I'd like this."|
|PolicyDecision|The runtime said: "Yes / no / yes but changed."|
|ToolCall|The runtime did: "Here is what I actually started."|
|ToolResult|The world said: "Here is what happened."|
|Observation|The runtime tells the model: "Here is what you may see."|

If you merge them, you lose the ability to see where they differ. And they _do_ differ: a `MODIFIED` decision means the **ToolCall's args are not the proposal's args.** The model asked for 5000. The tool ran with 100. Those need to be two records, or the log will say something that never happened.

### The chain of ids

Each record points at the one before it. Following the chain backwards answers "why did that happen?"

```
ToolResult ──► ToolCall ──► PolicyDecision ──► ActionProposal ──► ModelResponse ──► ModelRequest ──► Context ──► fragments
```

Notice that a `ToolCall` points at a `decision_id`. That is its **credential**. A call with no decision behind it has no authorization, which is the point of 3.7: authority comes from something the runtime recorded, not from the model's words.

Also notice: `Observation.source` can be `HUMAN_INPUT` or `SYSTEM`, not only a tool result. The same wrapper brings human replies and system messages back into context, always as a labeled fragment. When the source isn't a tool, there's no tool call to point at, so `tool_call_id` has to be allowed to be empty.

---

## Code

Save as `value_objects.py`. It builds one complete step as frozen records, then walks the chain backwards. The model asks to refund 5000, the policy clamps it to 100, and the tool runs with 100.

```python
from dataclasses import dataclass

# ---------- every object is frozen: written once, never edited ----------
@dataclass(frozen=True)
class ContextFragment:
    id: str
    source: str
    instruction_authority: str      # can_instruct | data_only
    content: str

@dataclass(frozen=True)
class Context:
    id: str
    step_id: str
    fragment_ids: tuple             # ordered REFERENCES to fragments

@dataclass(frozen=True)
class ModelRequest:
    id: str
    step_id: str
    context_id: str

@dataclass(frozen=True)
class ModelResponse:
    id: str
    request_id: str
    step_id: str
    text: str

@dataclass(frozen=True)
class ActionProposal:
    id: str
    response_id: str
    step_id: str
    type: str                       # TOOL_CALL | FINAL_ANSWER
    tool_name: str
    tool_args: dict

@dataclass(frozen=True)
class PolicyDecision:
    id: str
    proposal_id: str
    decision: str                   # APPROVED | REJECTED | MODIFIED
    reason: str
    amended_args: dict

@dataclass(frozen=True)
class ToolCall:
    id: str
    step_id: str
    decision_id: str
    tool_name: str
    args: dict
    dedup_key: str

@dataclass(frozen=True)
class ToolResult:
    id: str
    tool_call_id: str
    status: str                     # SUCCESS | ERROR | TIMEOUT | CANCELLED | UNKNOWN
    output: str

@dataclass(frozen=True)
class Observation:
    id: str
    step_id: str
    tool_call_id: str
    fragment: ContextFragment       # how the result re-enters context, labeled


# ---------- write-once store ----------
STORE = {}
def put(obj):
    assert obj.id not in STORE, f"{obj.id} already written - write a NEW record instead"
    STORE[obj.id] = obj
    return obj


# ---------- the runtime's verdict on a proposal ----------
def policy(p):
    amount = p.tool_args["amount"]
    if amount > 100:
        return PolicyDecision("d1", p.id, "MODIFIED", "amount over limit; clamped to 100",
                              {**p.tool_args, "amount": 100})
    return PolicyDecision("d1", p.id, "APPROVED", "within limit", p.tool_args)


# ---------- one full step ----------
def run_step():
    put(ContextFragment("f1", "system", "can_instruct", "You handle refunds. Max 100."))
    put(ContextFragment("f2", "user", "can_instruct", "refund order A1 by 5000"))
    put(Context("c1", "st1", ("f1", "f2")))
    put(ModelRequest("mreq1", "st1", "c1"))
    put(ModelResponse("mres1", "mreq1", "st1", '{"tool":"refund","order_id":"A1","amount":5000}'))
    prop = put(ActionProposal("p1", "mres1", "st1", "TOOL_CALL", "refund",
                              {"order_id": "A1", "amount": 5000}))
    dec = put(policy(prop))
    call = put(ToolCall("tc1", "st1", dec.id, "refund", dec.amended_args, "run1-st1"))
    res = put(ToolResult("tr1", call.id, "SUCCESS", f"refunded {call.args['amount']} for A1"))
    put(Observation("o1", "st1", call.id,
                    ContextFragment("f3", "tool_result", "data_only", res.output)))


# ---------- follow the id chain backwards ----------
def explain(result_id):
    r = STORE[result_id]
    tc = STORE[r.tool_call_id]
    d = STORE[tc.decision_id]
    p = STORE[d.proposal_id]
    m = STORE[p.response_id]
    q = STORE[m.request_id]
    c = STORE[q.context_id]
    frags = ", ".join(f"{STORE[f].id} {STORE[f].source}/{STORE[f].instruction_authority}"
                      for f in c.fragment_ids)
    print(f"ToolResult {r.id}: {r.status} '{r.output}'")
    print(f"  came from ToolCall {tc.id}: {tc.tool_name} {tc.args}")
    print(f"  authorized by PolicyDecision {d.id}: {d.decision} ({d.reason})")
    print(f"  which answered ActionProposal {p.id}: asked {p.tool_name} {p.tool_args}")
    print(f"  which was parsed from ModelResponse {m.id}")
    print(f"  which answered ModelRequest {q.id}")
    print(f"  which was built from Context {c.id}: [{frags}]")


# ---------- audit: is every call properly authorized and matched? ----------
def audit():
    problems = []
    for o in STORE.values():
        if isinstance(o, ToolCall):
            d = STORE.get(o.decision_id)
            if d is None:
                problems.append(f"{o.id}: decision {o.decision_id} does not exist")
            elif d.decision == "REJECTED":
                problems.append(f"{o.id}: ran although decision {d.id} is REJECTED")
            elif o.args != d.amended_args:
                problems.append(f"{o.id}: args {o.args} differ from decision {d.id} {d.amended_args}")
    return problems

def outcome(tool_call_id):
    for o in STORE.values():
        if isinstance(o, ToolResult) and o.tool_call_id == tool_call_id:
            return o.status
    return "UNKNOWN (call recorded, no result recorded)"


if __name__ == "__main__":
    run_step()
    explain("tr1")
    print("\naudit problems:", audit())
```

Run `python value_objects.py`:

```
ToolResult tr1: SUCCESS 'refunded 100 for A1'
  came from ToolCall tc1: refund {'order_id': 'A1', 'amount': 100}
  authorized by PolicyDecision d1: MODIFIED (amount over limit; clamped to 100)
  which answered ActionProposal p1: asked refund {'order_id': 'A1', 'amount': 5000}
  which was parsed from ModelResponse mres1
  which answered ModelRequest mreq1
  which was built from Context c1: [f1 system/can_instruct, f2 user/can_instruct]

audit problems: []
```

Read it from top to bottom. It's the whole story of one step, rebuilt from ids alone. The model asked for **5000**, the runtime changed it, and **100** was what actually moved. The proposal and the call are different records, so both facts are still there.

### Try it: write-once

```python
import value_objects as v
v.run_step()
try:
    v.STORE["mres1"].text = "something else"
except Exception as e:
    print(type(e).__name__, "->", e)
try:
    v.put(v.ModelResponse("mres1", "mreq1", "st1", "something else"))
except AssertionError as e:
    print("AssertionError ->", e)
```

```
FrozenInstanceError -> cannot assign to field 'text'
AssertionError -> mres1 already written - write a NEW record instead
```

Two layers of protection: you can't edit a record, and you can't write a second record with the same id. The only way to add information is to write a new record with a new id.

### Break it on purpose

**Break 1: one record for "proposal and call."** Suppose you saved only the proposal and treated it as what ran:

```python
import value_objects as v
v.run_step()
p, tc, tr = v.STORE["p1"], v.STORE["tc1"], v.STORE["tr1"]
print("one-record design would log   :", f"{p.tool_name} {p.tool_args}")
print("what actually ran (ToolCall)  :", f"{tc.tool_name} {tc.args}")
print("what the tool actually did    :", tr.output)
```

```
one-record design would log   : refund {'order_id': 'A1', 'amount': 5000}
what actually ran (ToolCall)  : refund {'order_id': 'A1', 'amount': 100}
what the tool actually did    : refunded 100 for A1
```

The one-record log says 5000 was refunded. The bank says 100. Nothing crashed, and the log is quietly false. You'd find out in an audit, or from an angry customer. Separate records are what keep "what was asked" apart from "what was done."

**Break 2: a call with no valid authorization.** Two calls appear: one pointing at a `REJECTED` decision, one pointing at a decision that doesn't exist:

```python
import value_objects as v
v.run_step()
v.put(v.PolicyDecision("d2", "p1", "REJECTED", "not allowed", {}))
v.put(v.ToolCall("tc2", "st1", "d2", "refund", {"order_id": "A1", "amount": 5000}, "run1-st1-b"))
v.put(v.ToolCall("tc3", "st1", "d99", "refund", {"order_id": "A1", "amount": 5000}, "run1-st1-c"))
for p in v.audit():
    print(" ", p)
```

```
  tc2: ran although decision d2 is REJECTED
  tc3: decision d99 does not exist
```

Because every `ToolCall` carries a `decision_id`, an audit can find calls that should never have happened. If the call record had no such link, there would be nothing to check it against.

**Try it: a crash between the call and the result.** The call was recorded but no result ever was:

```python
import value_objects as v
v.run_step()
v.put(v.ToolCall("tc9", "st1", "d1", "refund", {"order_id": "A1", "amount": 100}, "run1-st1-z"))
print("tc1 outcome:", v.outcome("tc1"))
print("tc9 outcome:", v.outcome("tc9"))
```

```
tc1 outcome: SUCCESS
tc9 outcome: UNKNOWN (call recorded, no result recorded)
```

A call with no result is exactly the "intent with no outcome" from 3.3. It's why `ToolResult.status` includes `UNKNOWN`, and why call and result are two separate records.

---

## Real-life examples

### How Claude does it

When I run a search, there's a request I make (the proposal), the app's check of whether it's allowed (the decision), the search it actually runs (the call), what the search returns (the result), and what I'm shown (the observation, which is the result presented as data). Each is a separate thing, and the app can show you its checks and its results independently of what I asked.

### How payments work

A bank keeps the payment **instruction**, the **approval**, the **execution** and the **confirmation** as separate records. If an instruction is approved for a smaller amount, the original instruction isn't edited. This is the same pattern for the same reason: when money is involved, "what was asked" and "what happened" must never be confused.

### Where real systems go wrong

|Mistake|What happens|
|---|---|
|Edit a record in place when something changes|History is rewritten, and the log lies|
|Store the proposal as if it were the executed call|Logs show actions that never ran as described|
|No link from a call to its authorization|Nothing to audit against|
|No separate result record|A crash looks identical to "nothing happened"|
|Feed raw tool output back unlabeled|The result gets read as an instruction|

---

## Where this lands in our build

**1. This is P1.** The roadmap lists `Context`, `ModelRequest`/`ModelResponse`, `ActionProposal`, `ToolCall`/`ToolResult` and `Observation`, plus the `ContextFragment` envelope. `PolicyDecision` is the output of the no-op `PolicyGate` seam P1 introduces.

**2. P4 fills in tool behavior.** Tool output is emitted as a `ContextFragment` with `instruction_authority = data_only`. That's the `Observation.fragment` in the demo, labeled `data_only`.

**3. P6 turns these into events.** The roadmap's event list includes `ModelRequested`, `ModelResponded`, `ActionProposed`, `ToolStarted`, `ToolCompleted` and `ObservationRecorded`, plus `ToolCallIntended` and `UnknownOutcome`. A `ToolResult` with status `UNKNOWN` is the `UnknownOutcome` case.

**4. P10 is `explain()`.** The roadmap's bar is "for any run, answer why did it do that from the trace alone." The id chain is what makes that possible.

**5. P18 builds the Context.** The renderer chooses fragments and places them. A Context that stores fragment **ids**, not copies of their text, matches the roadmap's "the log stores references and not payloads."

**6. Add to your open-questions list:**

```
1. The roadmap's P6 event list has no PolicyDecision event, and ToolCall has a started_at while intent must be logged BEFORE execution. Is ToolCall the intent record (with dedup_key), or are ToolCallIntended and ToolStarted separate from it? (Resolves at P6.)

2. Only the tool_call path produces a PolicyDecision. A FINAL_ANSWER goes straight to "Run completes." Should the runtime's acceptance of a final answer also be recorded as a decision, so refused or forced stops are visible? (Resolves around P15.)
```

**7. Log it in `DECISIONS.md`:**

```
Each step's facts are separate write-once records by author (proposal, decision, call, result, observation) chained by ids; ToolCall.args is what ran and ActionProposal.tool_args is what was asked — they may differ, so they are never merged.
```

---

## Say it to your manager

> "Everything that happens in a step is saved as its own record, written once and never edited. The model's proposal, the runtime's decision, the tool call, the tool's result, and the observation that goes back to the model are all separate, because they're different facts from different authors. Each points at the one before it by id. So for any outcome, we can follow the chain back to exactly what was asked, what was decided, what was run, and what the model saw. If a decision changes the arguments, both the original request and the modified one stay on record."

---

## Common mistakes

- **Merging proposal and call into one record.** The log will say what was asked, not what was done (Break 1).
- **Editing a record when something changes.** Write a new one instead.
- **Letting a call exist without a decision id.** There's nothing to audit it against (Break 2).
- **Treating "no result" as "didn't happen."** Mark the outcome as unknown.
- **Storing copies of fragment text in every Context.** Store the ids.
- **Feeding results back without a data-only label.** An Observation's fragment must be labeled.

---
