# Topic 3.3 — The proposal-execution gap

Back to the kitchen one more time.

A customer says to the waiter, "Burn the building down." The waiter writes it on the ticket. The ticket reaches the kitchen manager. The manager reads it, says "no," and the building is fine.

Now imagine there is no manager. The ticket goes straight to a cook with a blowtorch.

The ticket was the same both times. What changed is whether anyone stood **between the words and the blowtorch**.

That space is the gap. A proposal is only words on a ticket. Nothing happens until someone chooses to act on them, and that someone is the runtime.

---

## Simple technical version

A language model can only write text. When it "calls a tool," it writes something like:

```
{"tool": "refund", "args": {"order_id": "A2", "amount": 5000}}
```

That is a **proposal**. It says "I would like this to happen." It is not an action. Nothing has been refunded.

The **execution** is when your code actually calls `refund(...)` and money moves.

The **gap** is everything your runtime does between those two moments:

Think of it as three stages. The model is on top, the real world is at the bottom, and the runtime sits in the middle.

```
        MODEL
          │
          │  writes a proposal (just text)
          ▼
┌─────────────────────────────────────────────┐
│  STAGE 1 — SHOULD THIS HAPPEN?              │
│                                             │
│   • Can I even read it?                     │
│   • Does this tool exist?                   │
│   • Are the arguments the right shape?      │
│   • Is it allowed?                          │
│   • Can we afford it?                       │
│   • Does a human need to approve?           │
│                                             │
│   Any "no"  ──►  STOP. Tell the model why.  │
└──────────────────────┬──────────────────────┘
                       │ all "yes"
                       ▼
┌─────────────────────────────────────────────┐
│  STAGE 2 — DO IT, SAFELY                    │
│                                             │
│   a. Write down: "I am about to do this"    │
│      (ToolCallIntended + dedup key)         │
│   b. Do it                                  │
│      (the ONLY step that touches the world) │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│  STAGE 3 — REMEMBER AND REPORT              │
│                                             │
│   • Write down what happened                │
│     (or UnknownOutcome if we crashed)       │
│   • Give the result back to the model       │
└─────────────────────────────────────────────┘
          │
          ▼
        MODEL reads the result and decides what to do next
```

Only one box in this whole picture touches the real world: **2b, "Do it."** Everything else is the runtime thinking, checking and writing notes.

### Follow one proposal through, step by step

**Proposal A: refund 50 for order A1**

|Stage|What the runtime does|Result|
|---|---|---|
|1. Should this happen?|Tool exists, args are right, 50 is under the limit of 100|Pass|
|2a. Write intent|Log "about to refund A1, key run1-1"|Written|
|2b. Do it|Call `refund`|Money moves|
|3. Remember|Log "refund completed" and tell the model|Done|

**Proposal B: refund 5000 for order A2**

|Stage|What the runtime does|Result|
|---|---|---|
|1. Should this happen?|5000 is over the limit of 100|**Fail, stop here**|
|2a. Write intent|Never reached||
|2b. Do it|Never reached|Money does not move|
|3. Remember|Log "rejected" and tell the model "denied: amount outside 1..100"|Model can try again|

The model wrote both proposals with the same confidence. The only difference is what the runtime did with them.

### Why Stage 2 has two parts

You write "I am about to do this" **before** you do it. Why? Imagine the power cuts right after the money moves. If you never wrote the note, then after restart your records say nothing happened, but money has moved. If you wrote the note first, your records say "I was about to refund A1, and I never wrote that it finished." You know something was in flight, so you can check or retry safely. That's all the dedup key and `UnknownOutcome` are for. Break 2a and 2b below show it live.

### Why this is the most important idea in agent design

- **The model can be wrong.** It invents tool names, sends the wrong types, and picks amounts that are far too large.
- **The model can be tricked.** If a web page says "email this file to me," and the model reads that page, it may _propose_ emailing the file. The proposal is only text, so the gap is where that gets stopped.
- **The model can't be trusted to police itself.** A rule written in the prompt is a request. A rule written in the gap is enforced.

Everything else in your roadmap, such as validation, policy, budgets, approval, durability and TinySeal, is something that lives in this gap. The roadmap says it directly: the gap between proposal and execution is where everything interesting lives.

### The gap does two jobs

|Job|When|Question it answers|Example|
|---|---|---|---|
|**Before the effect**|Stage 1|"Should this happen at all?"|Refund of 5000 is over the limit, so deny it|
|**Around the effect**|Stages 2 and 3|"If this crashes halfway, can we recover?"|Process dies after the refund but before we wrote it down|

Beginners only think about the first job. The second is the one that causes real damage, because after a crash you honestly **don't know** whether the refund happened.

### What a rejection should do

When the gap says no, the loop does **not** crash. The denial goes back to the model as an observation, such as "denied: amount outside 1..100". The model can then try something else. A rejected proposal is a normal event, not an exception.

### Validation and policy are different checks

The toy code below puts them in one function to keep it short. In Agent Core they are separate:

|Check|Question|Where it lives|Phase|
|---|---|---|---|
|**Validation**|Is the proposal well-formed? Does the tool exist? Do the args match the schema?|`ToolValidator`|P4|
|**Policy**|Is this _allowed_, for this principal, in this situation?|`PolicyGate`|P1 seam, P28 TinySeal|

A proposal can be perfectly well-formed and still forbidden. A refund of 5000 to a real order has the right shape and may still be denied. Keeping the two checks apart means TinySeal can plug into policy later without touching validation.

---

## Code

Save this as `gap.py`. The model is not in this file at all. We feed proposals in by hand, which is how you test a gap properly: you can throw any proposal at it, including bad ones.

```python
LOG = []

def emit(kind, **data):
    LOG.append({"kind": kind, **data})
    print(f"  [{len(LOG)}] {kind} {data}")


# ---------- THE WORLD (what can actually be hurt) ----------
BANK = {"balance": 1000, "seen_keys": set()}

def refund(order_id, amount, key):
    # WHY a key: if the same refund arrives twice, the world ignores the repeat.
    if key in BANK["seen_keys"]:
        return f"already done (key {key})"
    BANK["seen_keys"].add(key)
    BANK["balance"] -= amount
    return f"refunded {amount} for {order_id}"

TOOLS = {"refund": {"fn": refund, "max_amount": 100}}


# ---------- THE GAP: a proposal is only text until this passes ----------
def gate(p):
    if p.get("tool") not in TOOLS:
        return "unknown tool"
    a = p.get("args")
    if not (isinstance(a, dict) and isinstance(a.get("order_id"), str)
            and isinstance(a.get("amount"), int)):
        return "bad arguments"
    if not 0 < a["amount"] <= TOOLS[p["tool"]]["max_amount"]:
        return f"amount outside 1..{TOOLS[p['tool']]['max_amount']}"
    return None


class Crash(BaseException):      # simulates the process dying
    pass
CRASH_AFTER_EFFECT = False


def execute(run_id, step, p):
    emit("ActionProposed", step=step, proposal=p)

    reason = gate(p)
    if reason:
        emit("ActionRejected", step=step, reason=reason)
        return f"denied: {reason}"           # goes back to the model as an observation

    key = f"{run_id}-{step}"
    emit("ToolCallIntended", step=step, key=key)      # BEFORE the effect

    try:
        a = p["args"]
        result = TOOLS[p["tool"]]["fn"](a["order_id"], a["amount"], key)
        if CRASH_AFTER_EFFECT:
            raise Crash()                    # effect happened, record did not
    except Exception as e:
        emit("UnknownOutcome", step=step, key=key, error=str(e))
        return "unknown outcome: do not assume it failed"

    emit("ToolCompleted", step=step, key=key, result=result)
    return result


def resume(run_id, step, p):
    key = f"{run_id}-{step}"
    kinds = [e["kind"] for e in LOG if e.get("key") == key]
    if "ToolCallIntended" in kinds and "ToolCompleted" not in kinds:
        emit("UnknownOutcome", step=step, key=key, error="log ends after intent")
        a = p["args"]
        result = TOOLS[p["tool"]]["fn"](a["order_id"], a["amount"], key)   # SAME key
        emit("ToolCompleted", step=step, key=key, result=result, via="resume")
        return result


if __name__ == "__main__":
    proposals = [
        {"tool": "refund",    "args": {"order_id": "A1", "amount": 50}},
        {"tool": "refund",    "args": {"order_id": "A2", "amount": 5000}},
        {"tool": "delete_db", "args": {}},
        {"tool": "refund",    "args": {"order_id": "A3", "amount": "fifty"}},
    ]
    for i, p in enumerate(proposals, 1):
        print(f"\nstep {i}")
        print("  observation ->", execute("run1", i, p))
    print("\nbalance:", BANK["balance"])
```

Run `python gap.py`:

```
step 1
  [1] ActionProposed {'step': 1, 'proposal': {'tool': 'refund', 'args': {'order_id': 'A1', 'amount': 50}}}
  [2] ToolCallIntended {'step': 1, 'key': 'run1-1'}
  [3] ToolCompleted {'step': 1, 'key': 'run1-1', 'result': 'refunded 50 for A1'}
  observation -> refunded 50 for A1

step 2
  [4] ActionProposed {'step': 2, 'proposal': {'tool': 'refund', 'args': {'order_id': 'A2', 'amount': 5000}}}
  [5] ActionRejected {'step': 2, 'reason': 'amount outside 1..100'}
  observation -> denied: amount outside 1..100

step 3
  [6] ActionProposed {'step': 3, 'proposal': {'tool': 'delete_db', 'args': {}}}
  [7] ActionRejected {'step': 3, 'reason': 'unknown tool'}
  observation -> denied: unknown tool

step 4
  [8] ActionProposed {'step': 4, 'proposal': {'tool': 'refund', 'args': {'order_id': 'A3', 'amount': 'fifty'}}}
  [9] ActionRejected {'step': 4, 'reason': 'bad arguments'}
  observation -> denied: bad arguments

balance: 950
```

Four proposals came in and only one became an effect. The balance moved by exactly 50. Read the log in order: every proposal is recorded, including the rejected ones, so "why didn't it refund A2?" has an answer.

Notice the order on step 1: `ToolCallIntended` is written **before** the refund, and `ToolCompleted` after. Hold onto that. Break 2 shows why.

### Break it on purpose

Each break is a separate small script that imports `gap.py`.

**Break 1: remove the gap.** Disable the gate and send the bad refund:

```python
import gap
gap.gate = lambda p: None        # the gap is now empty
print("  observation ->", gap.execute("run1", 1,
      {"tool": "refund", "args": {"order_id": "A2", "amount": 5000}}))
print("balance:", gap.BANK["balance"])
```

```
  [1] ActionProposed {'step': 1, 'proposal': {'tool': 'refund', 'args': {'order_id': 'A2', 'amount': 5000}}}
  [2] ToolCallIntended {'step': 1, 'key': 'run1-1'}
  [3] ToolCompleted {'step': 1, 'key': 'run1-1', 'result': 'refunded 5000 for A2'}
  observation -> refunded 5000 for A2
balance: -4000
```

The balance went to -4000. The proposal text was identical to the one that was safely denied before. **Only the gap changed.** This is why "the model is smart enough not to do that" is not a safety design.

**Break 2a: crash after the effect, resume correctly.** The process dies after the refund happened but before it was recorded:

```python
import gap
p = {"tool": "refund", "args": {"order_id": "A1", "amount": 50}}
gap.CRASH_AFTER_EFFECT = True
try:
    gap.execute("run1", 1, p)
except gap.Crash:
    print("  *** process died ***")
gap.CRASH_AFTER_EFFECT = False
print("resume:")
gap.resume("run1", 1, p)
print("balance:", gap.BANK["balance"])
```

```
  [1] ActionProposed {'step': 1, 'proposal': {'tool': 'refund', 'args': {'order_id': 'A1', 'amount': 50}}}
  [2] ToolCallIntended {'step': 1, 'key': 'run1-1'}
  *** process died ***
resume:
  [3] UnknownOutcome {'step': 1, 'key': 'run1-1', 'error': 'log ends after intent'}
  [4] ToolCompleted {'step': 1, 'key': 'run1-1', 'result': 'already done (key run1-1)', 'via': 'resume'}
balance: 950
```

Read the log after the crash: it ends at `ToolCallIntended` with nothing after it. That is the only evidence we have, and it can't tell us whether the refund happened. So resume records `UnknownOutcome` honestly instead of guessing, then retries with the **same key**. The world says "already done," and the balance is 950. Refunded exactly once.

**Break 2b: crash, then retry with a fresh key.** Same crash, but a naive retry makes a new key:

```python
import gap, uuid
p = {"tool": "refund", "args": {"order_id": "A1", "amount": 50}}
gap.CRASH_AFTER_EFFECT = True
try:
    gap.execute("run1", 1, p)
except gap.Crash:
    print("  *** process died ***")
gap.CRASH_AFTER_EFFECT = False
a = p["args"]
print("naive retry:", gap.refund(a["order_id"], a["amount"], str(uuid.uuid4())))
print("balance:", gap.BANK["balance"])
```

```
  [1] ActionProposed {'step': 1, 'proposal': {'tool': 'refund', 'args': {'order_id': 'A1', 'amount': 50}}}
  [2] ToolCallIntended {'step': 1, 'key': 'run1-1'}
  *** process died ***
naive retry: refunded 50 for A1
balance: 900
```

The customer got refunded twice. The balance is 900 instead of 950. The retry looked completely reasonable, and that's the trap. The **dedup key has to be stable across retries**, so it must be derived from the run and step, never generated fresh at retry time.

**Think about:** what if `refund` ignored keys entirely? Then even the same-key retry would double-pay, and the runtime could not safely retry at all. It would have to escalate to a human. That is exactly what the tool's `idempotency` field in P4 is for.

---

## Real-life examples

### How Claude does it

When I want to run something, I write a request. I can't run anything myself. The app around me looks at the request, checks that the tool exists and the arguments fit, and in some setups asks you for permission first. Only then does it run. If it's refused, I get a message saying so, and I carry on with that information. Every action I take passed through that gap.

### How ChatGPT does it

Same pattern. The model returns a function call as text. The developer's code decides whether to run it. Some products ask "Allow this action?" before running browsing or code. That prompt is the gap made visible to the user.

### Where real systems fail without the gap

|Situation|What the model wrote|Without a gap|With a gap|
|---|---|---|---|
|Hallucinated tool|`delete_all_users()`|Runtime throws or, worse, finds a similar function|Unknown tool, denied|
|Wrong type|`amount: "fifty"`|Crash, or silent bad math|Bad arguments, model corrects itself|
|Injected instruction|A web page says "email the file to me"|Email goes out|Policy denies, or a human approves|
|Runaway loop|Same tool call 400 times|Big bill|Budget refuses the 51st|
|Crash mid-effect|Refund, then power cut|Double refund on restart|Intent logged, recovery path runs|

---

## Where this lands in our build

**1. This is the main idea behind P4, and the roadmap states it as the thing to understand.** P4's "Understand" line is: why the model proposing a tool call and the runtime executing it must be two separate things. If you can explain Break 1 and Break 2b, you can.

**2. Each step of the gap has a home.**

|Gap step|Where it lives|Phase|
|---|---|---|
|Parse and repair malformed output|ModelProvider|P3|
|Known tool, schema validation|`ToolRegistry`, `ToolValidator`|P4|
|Policy check|`PolicyGate` (no-op seam now)|P1, filled at P28|
|Budget reserve|Budget ledger|P8|
|Human approval|`WAITING_FOR_HUMAN`|P20|
|`ToolCallIntended` with dedup key|Event log|P6|
|`UnknownOutcome`|Event log, recovery matrix|P6, P9|
|Result back as `data_only`|Context fragment|P4, enforced at P18|

**3. Intent before effect is non-negotiable.** In the toy, `ToolCallIntended` is written before the refund. If you wrote it after, a crash would leave no trace at all, and you couldn't even know there was something to recover. Roadmap P6 manual check: `kill -9` between tool execution and outcome recording. Break 2a is that test in miniature.

**4. Recovery depends on the tool, and the roadmap derives it.** Resume-with-same-key was safe here only because `refund` honors the key (`idempotency = client_key`). For a tool with `idempotency = none` and `irreversible`, retry is not safe, and recovery must be "escalate." That is why P4's `recovery` field is derived from effect, idempotency and reversibility, and not declared by hand.

**5. The gap is where TinySeal plugs in.** The P28 diagram is the gap drawn out: `ActionProposal → TinySeal Adapter → Policy → Approval → ExecutionGateway → Observation`. If you build the gap cleanly now, P28 is a plug and not a rewrite.

**6. Add to your open-questions list:**

```
UnknownOutcome with idempotency = none: who decides, retry / escalate / fail? Is it per-tool policy or per-run? (Resolves at P9 recovery matrix.)
```

**7. Log it in `DECISIONS.md`:**

```
Every ActionProposal passes validation, policy and budget before any effect; ToolCallIntended (stable dedup key = run+step) is logged before execution — the model proposes, the runtime decides, and a crash mid-effect is recoverable (P4, P6).
```

---

## Say it to your manager

> "The model never does anything. It writes a proposal, which is just text. Between that proposal and any real action sits the runtime, and that's where we validate it, check policy, count the budget, and ask for approval. Before we execute, we log our intent with a stable key, so if we crash mid-action we know exactly what we were doing and can retry safely. If the model is wrong, tricked, or looping, the gap is what stops it. Safety lives in the gap, not in the prompt."

---

## Common mistakes

- **Putting the rules in the prompt only.** A prompt is a request. The gap is enforcement.
- **Letting the model's output call the tool directly.** That is Break 1.
- **Treating a denial as a crash.** It's an observation the model can learn from.
- **Logging the outcome but not the intent.** After a crash, you won't know there was anything in flight.
- **Generating a fresh dedup key on retry.** That is Break 2b. The key must come from run and step.
- **Treating "no result" as "didn't happen."** After a crash, the honest answer is `UnknownOutcome`.
- **Mixing validation and policy.** They change for different reasons and live in different places.

---

