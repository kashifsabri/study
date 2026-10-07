
Think of a school report card.

- The **marks** on it are facts about _this_ report: Maths 82, English 74. They live here. They're **owned**.
- The **student's id number** is just a pointer. The card doesn't copy the student's whole file, address and photo. It says "student 4471" and the school office owns that file. That's a **reference**.
- The **teacher's name** is written on the card as it was that term. If the teacher later marries and changes her name, or leaves the school, last year's report card still says what it said. That's a **snapshot**: a copy made on purpose, because the original can change and the card must stay true to the moment it was written.

Every field on every record in your system is one of those three. Knowing which one decides where a fact lives and whether it can ever go stale.

---

## Simple technical version

### The three kinds of field

|Kind|What it is|Example|Rule|
|---|---|---|---|
|**Owned**|A fact that belongs to this record alone|`Run.status`, `Turn.user_input`|Only this record changes it|
|**Reference**|The id of another record. The other record owns the facts|`Step.run_id`|Store the id, never a copy of the record|
|**Snapshot**|A copy of a value, taken on purpose, at a moment in time|`Run.principal` (who started it, as they were then)|Never updated afterwards|

### The one rule that decides between them

> **Reference by default. Copy only when the original can change and you need the value as it was.**

Ask two questions of any field that points at something else:

1. Do I want the **current** truth? Then store a reference (an id) and look it up when needed.
2. Do I want the truth **as it was at that moment**, and can the original change? Then take a snapshot.

If the original can never change (an immutable `AgentVersion`), a reference already behaves like a snapshot. The id will always give the same answer.

### A fourth, special case: shortcut references

Sometimes a record stores an id it could have worked out by walking up its parents. In your schema a `Run` stores `turn_id`, but also `session_id`, `agent_id` and `version_id`. The last three could be found by following `turn_id` upward.

Why store them? So you can ask "all runs for this agent" in one lookup, without walking up through three tables each time.

The cost: **two sources of truth**, which can disagree. So shortcuts follow three rules:

- They are filled in **by code**, once, at creation. Nobody types them by hand.
- They are **never changed** afterwards.
- A **check** confirms they agree with the parent chain.

### Direction: children point up

A `Step` stores its `run_id`. A `Run` does **not** keep a list of its step ids. To find a run's steps, you search for steps whose `run_id` matches. This way there is one place to update, and no list that can go stale or miss an entry.

The exception is a list of references to things the record does **not** own. `AgentVersion.tool_ids[]` is a list of tools, but the version doesn't own the tools. The tool registry does.

### Your schema, one entity at a time

|Entity|Owns|References|Notes|
|---|---|---|---|
|**Agent**|`name`, `description`, `tenant_id`, `created_at`|Nothing. It's the top||
|**AgentVersion**|`version_number`, `instructions`, `model_config`, `limits`, `context_config`, `granted_authority`, `tenant_id`|`agent_id`, and `tool_ids[]` (tools it doesn't own)|Immutable|
|**Session**|`created_at`|`agent_id`, `version_id`|`agent_id` is a shortcut, since the version already knows its agent. `principal` is a snapshot of who started it|
|**Turn**|`index`, `user_input`, `created_at`|`session_id`||
|**Run**|`status`, `created_at`, `completed_at`, `final_answer`, `error`|`turn_id` (parent). `session_id`, `agent_id`, `version_id` (shortcuts)|`principal` is a snapshot. `tenant_id` also appears here|
|**Step**|`index`, `wall_clock`, `status`, `created_at`|`run_id`|Owns the next level too (see below)|

A `Step` also **owns** the records inside it: Context, ModelRequest, ModelResponse and ActionProposal. Each of those stores `step_id` (and `run_id`) pointing back up. What each of them contains is topic 4.4.

---

## Code

Save as `ownership.py`. It builds one Run and labels every field as owned, parent reference, shortcut or snapshot. Then it runs two checks: that every reference points at a real record, and that the shortcuts agree with the parent chain.

```python
# ---------- data. Each record OWNS its own facts and REFERENCES others by id ----------
AGENTS   = {"a1":  {"id": "a1", "name": "support-bot", "tenant_id": "tenA"}}
VERSIONS = {"av1": {"id": "av1", "agent_id": "a1", "number": 1,
                    "instructions": "Be polite. Refunds up to 100.", "tenant_id": "tenA"}}
USERS    = {"u1":  {"role": "viewer"}}          # owned by someone OUTSIDE the agent
SESSIONS = {"s1":  {"id": "s1", "agent_id": "a1", "version_id": "av1",
                    "principal": {"user_id": "u1", "tenant_id": "tenA", "role": "viewer"}}}
TURNS    = {"t1":  {"id": "t1", "session_id": "s1", "index": 0, "user_input": "refund A1"}}
RUNS, STEPS = {}, {}


# ---------- shortcuts and snapshots are filled in by CODE, once, at creation ----------
def create_run(run_id, turn_id):
    turn = TURNS[turn_id]
    session = SESSIONS[turn["session_id"]]
    RUNS[run_id] = {
        "id": run_id, "status": "CREATED",                         # owned
        "turn_id": turn_id,                                        # parent reference
        "session_id": session["id"],                               # shortcut (derivable)
        "agent_id": session["agent_id"],                           # shortcut (derivable)
        "version_id": session["version_id"],                       # shortcut (derivable)
        "principal": dict(session["principal"]),                   # SNAPSHOT: a copy, on purpose
    }

def create_step(step_id, run_id, index):
    STEPS[step_id] = {"id": step_id, "run_id": run_id, "index": index}


# ---------- check 1: does every reference point at something real? ----------
REFS = [("version", VERSIONS, "agent_id", AGENTS), ("session", SESSIONS, "version_id", VERSIONS),
        ("turn", TURNS, "session_id", SESSIONS), ("run", RUNS, "turn_id", TURNS),
        ("step", STEPS, "run_id", RUNS)]

def find_orphans():
    return [f"{name} {rec['id']}: {field}={rec[field]!r} does not exist"
            for name, table, field, target in REFS
            for rec in table.values() if rec[field] not in target]


# ---------- check 2: do the shortcuts agree with the parent chain? ----------
def check_run(run):
    turn = TURNS[run["turn_id"]]
    session = SESSIONS[turn["session_id"]]
    problems = []
    if run["session_id"] != session["id"]:
        problems.append(f"session_id {run['session_id']} != chain {session['id']}")
    if run["agent_id"] != session["agent_id"]:
        problems.append(f"agent_id {run['agent_id']} != chain {session['agent_id']}")
    if run["version_id"] != session["version_id"]:
        problems.append(f"version_id {run['version_id']} != chain {session['version_id']}")
    return problems


if __name__ == "__main__":
    create_run("r1", "t1")
    create_step("st1", "r1", 0)

    kinds = {"id": "owned", "status": "owned", "turn_id": "PARENT reference",
             "session_id": "shortcut reference", "agent_id": "shortcut reference",
             "version_id": "shortcut reference", "principal": "SNAPSHOT copy"}
    print("Run r1:")
    for field, value in RUNS["r1"].items():
        print(f"  {field:<11} {kinds[field]:<19} {value}")

    print("\norphans:", find_orphans())
    print("shortcut problems:", check_run(RUNS["r1"]))
```

Run `python ownership.py`:

```
Run r1:
  id          owned               r1
  status      owned               CREATED
  turn_id     PARENT reference    t1
  session_id  shortcut reference  s1
  agent_id    shortcut reference  a1
  version_id  shortcut reference  av1
  principal   SNAPSHOT copy       {'user_id': 'u1', 'tenant_id': 'tenA', 'role': 'viewer'}

orphans: []
shortcut problems: []
```

Look at where each value came from. `create_run` reads the shortcuts and the snapshot from the session. No human typed `av1`. And `principal` is copied with `dict(...)`, so it's a separate copy that won't change if the session's dictionary changes.

### Try it: an orphan

A step points at a run that doesn't exist:

```python
import ownership as o
o.create_run("r1", "t1")
o.create_step("st9", "r99", 0)
print("orphans:", o.find_orphans())
```

```
orphans: ["step st9: run_id='r99' does not exist"]
```

A reference is only a promise that the other record exists. The check keeps the promise honest. In a real database, a foreign key does this job.

### Break it on purpose

**Break 1: a hand-typed shortcut that disagrees with the chain.** Someone sets `version_id` by hand:

```python
import ownership as o
o.VERSIONS["av2"] = {"id": "av2", "agent_id": "a1", "number": 2,
                     "instructions": "Refunds up to 5000.", "tenant_id": "tenA"}
o.create_run("r1", "t1")
o.RUNS["r1"]["version_id"] = "av2"               # someone typed it by hand
chain_version = o.SESSIONS[o.TURNS[o.RUNS["r1"]["turn_id"]]["session_id"]]["version_id"]
print("ask the run directly      :", o.RUNS["r1"]["version_id"])
print("ask by walking the parents:", chain_version)
print("check_run says            :", o.check_run(o.RUNS["r1"]))
```

```
ask the run directly      : av2
ask by walking the parents: av1
check_run says            : ['version_id av2 != chain av1']
```

Now "which version ran this?" has two answers. Which instructions and which refund limit applied depends on which path you asked. The fix is not to pick one by luck. It's to make shortcuts impossible to set by hand, and to have a check that catches a mismatch.

**Break 2: who was allowed, by reference versus by snapshot.** The user is promoted next week:

```python
import ownership as o
o.create_run("r1", "t1")
o.USERS["u1"]["role"] = "admin"                  # the user is promoted next week
print("by reference (look up the user now):", o.USERS[o.RUNS["r1"]["principal"]["user_id"]]["role"])
print("by snapshot  (copy taken at run time):", o.RUNS["r1"]["principal"]["role"])
```

```
by reference (look up the user now): admin
by snapshot  (copy taken at run time): viewer
```

At the time of the run the user was a `viewer`. If you stored only a reference, an audit next month says an `admin` authorized that run, which rewrites history. The snapshot keeps it true. This is exactly the report-card teacher's name: a copy on purpose, for a value that can change.

---

## Real-life examples

### An online shop

An order line stores the **price at the time of purchase** (a snapshot), and also the product id (a reference). If it only stored the product id, then when the price changes next month, last year's invoices would show a different total. The price must be a snapshot. The product name and description can stay a reference.

### This chat

The model was switched partway through this conversation. An app that recorded the model once for the whole conversation (a reference to "the current model") would mislabel the earlier messages. An app that records the model on each message (a snapshot) stays correct. In your schema, `ModelResponse` keeps `model_id` and `model_version` for this reason.

### Where real systems go wrong

|Mistake|What happens|
|---|---|
|Copy a value that should be a reference|The copy goes stale, and two records disagree|
|Reference a value that should be a snapshot|History changes when the original changes|
|Shortcut ids typed by hand|Two answers to "which version?"|
|No check that references exist|Orphans accumulate and traces dead-end|
|Parent stores a list of children|The list drifts out of sync with reality|

---

## Where this lands in our build

**1. This is P1.** It decides which fields each contract carries. The roadmap's P1 manual check, "hand-write one event of each type as JSON," will quickly show which fields you struggle to fill in, and those are the ones you haven't classified.

**2. Snapshots appear in your schema already.** `Run.principal` is one. `ToolCall.tool_version` is another, because a tool may change after the call. `ModelResponse.model_version` is another.

**3. Shortcuts are what make events cheap to query.** At P6 each event carries `run_id` and `step_id` and the tenant, so you can find everything for a run without walking up. The same "set once by code, then check it" rule applies.

**4. Orphan checks belong in tests and in the database.** A real foreign key at P6 does what `find_orphans` does here.

**5. Add to your open-questions list:**

```
AgentVersion stores tool_ids[] but not tool versions. If a tool changes after the version is created, the version's behavior changes even though it is "immutable." Should the version pin tool versions? (Resolves at P19.)

tenant_id appears on Agent, AgentVersion and Run, inside Session.principal, but not on Turn or Step. Is inheriting it through the parent enough, or should every row carry it? Also, Run has it twice (tenant_id and principal.tenant_id), so they must always agree. (Resolves at P26.)
```

**6. Log it in `DECISIONS.md`:**

```
Each fact lives in one owner record and other records store its id; copies (Run.principal, ToolCall.tool_version, ModelResponse.model_version) are deliberate snapshots; shortcut ids are set by code at creation and verified by a check.
```

---

## Say it to your manager

> "Every field is one of three things: owned by the record, a reference to another record's id, or a deliberate snapshot of a value that could change. We store ids, not copies, so there's one place for each fact. We copy only when we need the value as it was, like who was logged in when a run started. Some ids are shortcuts for faster lookups, so they're set by code once and verified, never typed by hand."

---

## Common mistakes

- **Copying instead of referencing.** The copy goes stale.
- **Referencing instead of snapshotting.** History changes after the fact (Break 2).
- **Typing shortcut ids by hand.** They'll disagree with the chain (Break 1).
- **Letting a parent keep a list of its children.** Use the child's parent id.
- **Assuming a reference is valid.** Check that it points at something real.
- **Treating "immutable" as a promise when it points at mutable things.** A frozen version that references a changing tool isn't fully frozen.

---
