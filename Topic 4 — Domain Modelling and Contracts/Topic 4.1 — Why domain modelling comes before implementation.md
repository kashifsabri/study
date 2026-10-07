

Imagine a courier company that starts by writing the delivery app, with no agreed words.

The driver says "parcel." The office says "order." The warehouse says "shipment." A failed delivery and the re-delivery the next day are both called "a delivery."

Then the boss asks three simple questions:

- "How many deliveries did we make last month?" (Does a failed attempt count?)
- "Which driver had this parcel when it got damaged?" (Is that written anywhere?)
- "Why did this customer get charged twice?" (Which record is the real one?)

Nobody can answer, because the app was built before anyone decided **what the things are, how they connect, and who owns what.**

The fix is not better code. The fix is to decide the words and the links **first**: a _parcel_ belongs to a _shipment_, a shipment has _delivery attempts_, each attempt has a _driver_ and an _outcome_. Then write the app to follow them.

That's domain modelling.

---

## Simple technical version

**Domain modelling** means deciding, before writing the loop:

1. **The nouns.** What are the things in the system? (Agent, Session, Turn, Run, Step, and so on.)
2. **The ownership.** Which thing belongs inside which? (A Step belongs to a Run. A Run belongs to a Turn.)
3. **The links.** Which things point to which, by id? (A tool call points to the step it happened in.)
4. **What can change.** Which things are fixed once written, and which are updated?

You do this on paper, then you write it as contracts (4.8), and only then does the loop get written.

### Why first? Four reasons

|Reason|What it means|What goes wrong without it|
|---|---|---|
|**One shared vocabulary**|Everyone, and every file, uses the same word for the same thing|"Run" means three different things in three files|
|**Questions become answerable**|Links let you walk from any record to the ones around it|You can't tell which version, which attempt, or which step caused something|
|**Ownership is decided**|You know where each piece of data lives and who it belongs to (including the tenant)|Data ends up in the wrong place, or without an owner|
|**Persisted shapes are hard to change**|Once records are saved, the shape is stuck with you (4.6, 4.7)|You discover a mistake after a million records exist|

The fourth reason is the main one. **Code is cheap to change. Saved data is not.** If you get a function wrong, you edit it. If you get a saved record wrong, every old record is still in the wrong shape.

### The test: can you answer these without guessing?

A good model makes questions like these a matter of following links:

- Which agent version produced this answer?
- How many attempts did this user message take, and how did each one end?
- Which step caused this tool call?

These are exactly the questions you'll ask when something goes wrong. If the model can't answer them, you find out during an incident, not during design.

---

## Code

Save as `domain_first.py`. It stores the same facts two ways. The **blob** version is one dictionary per request, the way you'd write it if you started coding straight away. The **modelled** version has one record per noun, linked by ids.

```python
# ---------- NO MODELLING: everything about one request in a single blob ----------
BLOB = {"user": "u1", "text": "refund A1", "answer": "Refunded A1.", "tries": 2}


# ---------- MODELLED: each noun is its own record, linked by ids ----------
AGENT_VERSIONS = {"av1": {"agent": "support-bot", "version": 1, "model": "model-x"}}
SESSIONS  = {"s1": {"version_id": "av1", "user": "u1"}}
TURNS     = {"t1": {"session_id": "s1", "index": 0, "user_input": "refund A1"}}
RUNS      = {"r1": {"turn_id": "t1", "status": "FAILED",    "final_answer": None},
             "r2": {"turn_id": "t1", "status": "COMPLETED", "final_answer": "Refunded A1."}}
STEPS     = {"st1": {"run_id": "r1", "index": 0},
             "st2": {"run_id": "r2", "index": 0},
             "st3": {"run_id": "r2", "index": 1}}
TOOL_CALLS = {"tc1": {"step_id": "st1", "tool": "refund"},
              "tc2": {"step_id": "st2", "tool": "refund"}}


# ---------- three questions, answered by following the links ----------
def which_version(run_id):
    turn = TURNS[RUNS[run_id]["turn_id"]]
    session = SESSIONS[turn["session_id"]]
    v = AGENT_VERSIONS[session["version_id"]]
    return f"{v['agent']} v{v['version']} ({v['model']})"

def attempts_for_turn(turn_id):
    return {rid: r["status"] for rid, r in RUNS.items() if r["turn_id"] == turn_id}

def where_was_tool_call(tc_id):
    step = STEPS[TOOL_CALLS[tc_id]["step_id"]]
    return f"step {step['index']} of run {step['run_id']}"


if __name__ == "__main__":
    print("Q1. Which agent version produced the answer in run r2?")
    print("    blob   :", BLOB.get("version", "UNKNOWN - never stored"))
    print("    modelled:", which_version("r2"))

    print("Q2. How many attempts did this user message take, and how did each end?")
    print("    blob   :", f"{BLOB['tries']} tries (no way to see what happened in each)")
    print("    modelled:", attempts_for_turn("t1"))

    print("Q3. Which step caused tool call tc2?")
    print("    blob   :", "IMPOSSIBLE - tool calls are not recorded")
    print("    modelled:", where_was_tool_call("tc2"))
```

Run `python domain_first.py`:

```
Q1. Which agent version produced the answer in run r2?
    blob   : UNKNOWN - never stored
    modelled: support-bot v1 (model-x)
Q2. How many attempts did this user message take, and how did each end?
    blob   : 2 tries (no way to see what happened in each)
    modelled: {'r1': 'FAILED', 'r2': 'COMPLETED'}
Q3. Which step caused tool call tc2?
    blob   : IMPOSSIBLE - tool calls are not recorded
    modelled: step 0 of run r2
```

How to read it:

- **Q1:** the blob never stored the version. It's not that the code is hard, the information doesn't exist. The modelled version walks `run → turn → session → agent version`.
- **Q2:** the blob has the number `2`, but not what each attempt did. The modelled version shows that `r1` failed and `r2` completed.
- **Q3:** the blob has nothing to look at. In the modelled version, `tc2` points to a step, and the step points to a run.

Notice that nothing here is clever. The answers come from **ids that point at other ids**. Deciding those links is the modelling work.

### Break it on purpose

**Break 1: two people, two meanings of "run".** Two teams write reports from the same data:

```python
import domain_first as d

def runs_billing(): return len(d.SESSIONS)     # "a run is a whole conversation"
def runs_debug():   return len(d.RUNS)         # "a run is one attempt"

print("billing report says:", runs_billing(), "run(s)")
print("debug report says  :", runs_debug(), "run(s)")
```

```
billing report says: 1 run(s)
debug report says  : 2 run(s)
```

Same data, same word, different numbers. Neither is a bug in the code. The word was never defined. In an agent system this gets worse: "run" might mean the whole conversation, one user message, or one execution attempt. Your roadmap separates them into Session, Turn and Run for exactly this reason.

**Break 2: change a shape after data is already stored.** Later you decide sessions should hold a `principal` (user and tenant) instead of a plain `user`:

```python
import domain_first as d

# new code assumes every session has a 'principal'; stored sessions still say 'user'
try:
    print(d.SESSIONS["s1"]["principal"])
except KeyError as e:
    print("KeyError:", e, "-> the stored record did not change when the code did")
```

```
KeyError: 'principal' -> the stored record did not change when the code did
```

You edited the code in a minute. The stored record is still the old shape. To fix it you must either rewrite every stored record, or make the code understand both shapes forever. Topic 4.7 covers how to do that properly. For now, just feel the cost: **this is why the shapes get decided before anything is saved.**

---

## Real-life examples

### How Claude and ChatGPT show it

In a chat app, a conversation contains messages. When you press "regenerate," you get a second answer to the _same_ message. The app had to decide that "your message" and "an attempt at answering it" are two different things. In our model that is **Turn** (your message) and **Run** (one attempt). If the app had mixed them up, "regenerate" wouldn't be possible without overwriting the first answer.

### The prior art

Temporal, which your roadmap names as the real comparison class, separates a **Workflow** (the logical job, with a stable id) from a **Run** (one execution of it, with its own run id). The same distinction, decided early, so that retries and restarts have somewhere to live.

### Where real teams get hurt

|What was skipped|What it cost later|
|---|---|
|No separate record for "attempt"|Retries overwrite the failed attempt, so you lose the evidence|
|Agent config not versioned|"It worked last week" can't be reproduced|
|No tenant on the records|Adding it later means touching every table and every query|
|Vocabulary left informal|Reports disagree, and arguments turn into definitions|

---

## Where this lands in our build

**1. This is P1.** The roadmap's P1 defines the contracts before P2 builds the loop. P0 already names the terms (Run, Step, Event, State, Context, Observation). P1 freezes them in a shape you can save.

**2. `tenant_id` goes in from the first version.** It's in the roadmap's rule: tenancy goes into the schema at the start, not a year later. Break 2 is the reason: adding a field to saved records is a migration.

**3. The P1 manual check is the modelling test.** "Hand-write one event of each type as JSON. If it is awkward to write by hand, the contract is wrong." Writing records by hand is how you find a missing link.

**4. Add to your open-questions list:**

```
The schema has Run.status and Step.status as fields, but the event log is the truth and state is a projection. Are these fields stored, or derived from events? (Resolves at P5 and P6.)
```

**5. Log it in `DECISIONS.md`:**

```
Domain nouns, ownership and id links are decided in P1 before any loop code; saved shapes are treated as permanent, so contracts come first and the loop follows them.
```

---

## Say it to your manager

> "Before writing the loop, we decide what the things in the system are, which one owns which, and how they link by id. That gives everyone one vocabulary, and it means questions like 'which version produced this answer' or 'what happened in each attempt' are answered by following links. We do it first because code is cheap to change but saved records aren't, so a wrong shape discovered after launch means migrating every record."

---

## Common mistakes

- **Starting with the loop and "adding structure later."** The shapes you improvise become the saved shapes (Break 2).
- **Using one word for two things.** "Run" meaning both a conversation and an attempt (Break 1).
- **Storing a number where you need a list.** `tries: 2` hides what each attempt did.
- **Not storing which version produced a result.** The question is unanswerable afterwards.
- **Leaving tenant out "for now."** It's the most expensive field to add later.

---

