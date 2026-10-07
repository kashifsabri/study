

```
Agent
  └── AgentVersion
        └── Session
              └── Turn
                    └── Run
                          └── Step
```

Think of a customer-support phone line.

- **Agent:** the support team itself, "Billing Support." It's the same team next year.
- **AgentVersion:** the team's rulebook, Edition 3. It lists the scripts, the tools they may use, and their limits. Once printed, an edition is never edited. If the rules change, you print Edition 4.
- **Session:** one phone call from one customer.
- **Turn:** one thing the customer says, plus the answer they get. "I want a refund" and "Done."
- **Run:** one attempt to produce that answer. The first attempt hit a system crash. The second worked.
- **Step:** one small action inside an attempt: look up the order, check the limit, issue the refund.

Each one sits **inside** the one above it. A step belongs to exactly one attempt. An attempt belongs to exactly one thing the customer said. And so on up to the team.

That's the ownership hierarchy.

---

## Simple technical version

### The six levels

|Level|What it is|One-line question it answers|
|---|---|---|
|**Agent**|The identity: name, description, tenant|"Which agent is this?"|
|**AgentVersion**|A frozen configuration: instructions, model, tools, limits|"Exactly how was it set up?"|
|**Session**|One conversation, for one principal|"Which conversation?"|
|**Turn**|One user message and the agent's response to it|"Which user message?"|
|**Run**|One execution attempt at that turn|"Which attempt?"|
|**Step**|One iteration of the loop|"Which loop iteration?"|

### Three rules that make it a hierarchy

**Rule 1: Every child has exactly one parent.** A Step has one Run. A Run has one Turn. Nothing floats free.

**Rule 2: One parent, many children.** One Agent has many versions. One Session has many Turns. One Turn can have several Runs (retries). One Run has many Steps.

**Rule 3: The child holds the parent's id.** A Step stores `run_id`. A Run stores `turn_id`. That one link is what lets you walk up from any record to the agent, or down from the agent to every step.

### Why each level exists

Remove any level and something specific breaks.

|If you remove...|What breaks|
|---|---|
|The Agent / AgentVersion split|You can't change instructions without rewriting history. Every old run now claims to have used the new rules|
|Session|"A conversation" has no home, so multi-turn behavior is undefined|
|Turn|You can't tell a user message apart from the attempts made to answer it|
|Run|A retry overwrites the failed attempt, and a crash has nowhere to be recorded|
|Step|You can't replay a run, or say which loop iteration did what. A Step's `index` is a logical counter, and replay needs it|

### The one that is special: AgentVersion is immutable

Every other level changes while it runs. A Run goes from `RUNNING` to `COMPLETED`. A Session gets new Turns.

An AgentVersion **never changes after it's created.** To change the instructions, the model or a limit, you create a _new_ version. Why? So that for any Run you can answer "exactly what configuration produced this?" and the answer is still true next year.

### How to read the tree

Going **down**, you're asking "what happened inside this?" Going **up**, you're asking "what was this part of?" Debugging is mostly walking up from a bad step, or down from a bad answer.

---

## Code

Save as `hierarchy.py`. It builds all six levels as small dataclasses. Each stores only its own id, its parent's id, and one or two fields. The real schemas have more fields, and 4.3 covers them.

```python
from dataclasses import dataclass


# ---------- the six levels. Each has exactly ONE parent, held as <parent>_id ----------
@dataclass
class Agent:
    id: str
    name: str

@dataclass(frozen=True)                 # frozen = cannot be edited after creation
class AgentVersion:
    id: str
    agent_id: str
    number: int
    instructions: str

@dataclass
class Session:
    id: str
    version_id: str

@dataclass
class Turn:
    id: str
    session_id: str
    index: int
    user_input: str

@dataclass
class Run:
    id: str
    turn_id: str
    status: str

@dataclass
class Step:
    id: str
    run_id: str
    index: int
    what: str


# ---------- data: one agent, one version, one conversation, two messages ----------
AGENTS   = {"a1": Agent("a1", "support-bot")}
VERSIONS = {"av1": AgentVersion("av1", "a1", 1, "Be polite. Refunds up to 100.")}
SESSIONS = {"s1": Session("s1", "av1")}
TURNS    = {"t1": Turn("t1", "s1", 0, "hi, refund A1"),
            "t2": Turn("t2", "s1", 1, "and A2?")}
RUNS     = {"r1": Run("r1", "t1", "COMPLETED"),
            "r2": Run("r2", "t2", "FAILED"),        # first attempt at message 2 failed
            "r3": Run("r3", "t2", "COMPLETED")}     # second attempt worked
STEPS    = {"st1": Step("st1", "r1", 0, "look up order A1"),
            "st2": Step("st2", "r1", 1, "issue refund"),
            "st3": Step("st3", "r2", 0, "look up order A2"),
            "st4": Step("st4", "r3", 0, "look up order A2"),
            "st5": Step("st5", "r3", 1, "issue refund")}


# ---------- walk DOWN: print the tree ----------
def children(table, parent_field, parent_id):
    return [x for x in table.values() if getattr(x, parent_field) == parent_id]

def print_tree():
    for a in AGENTS.values():
        print(f"Agent {a.id} '{a.name}'")
        for v in children(VERSIONS, "agent_id", a.id):
            print(f"  AgentVersion {v.id} (v{v.number})")
            for s in children(SESSIONS, "version_id", v.id):
                print(f"    Session {s.id}")
                for t in children(TURNS, "session_id", s.id):
                    print(f"      Turn {t.id} [{t.index}] \"{t.user_input}\"")
                    for r in children(RUNS, "turn_id", t.id):
                        print(f"        Run {r.id} {r.status}")
                        for st in children(STEPS, "run_id", r.id):
                            print(f"          Step {st.id} [{st.index}] {st.what}")


# ---------- walk UP: from any step to the agent ----------
def path_up(step_id):
    st = STEPS[step_id]
    r = RUNS[st.run_id]
    t = TURNS[r.turn_id]
    s = SESSIONS[t.session_id]
    v = VERSIONS[s.version_id]
    return " -> ".join([st.id, r.id, t.id, s.id, v.id, v.agent_id])


if __name__ == "__main__":
    print_tree()
    print("\npath up from st5:", path_up("st5"))
```

Run `python hierarchy.py`:

```
Agent a1 'support-bot'
  AgentVersion av1 (v1)
    Session s1
      Turn t1 [0] "hi, refund A1"
        Run r1 COMPLETED
          Step st1 [0] look up order A1
          Step st2 [1] issue refund
      Turn t2 [1] "and A2?"
        Run r2 FAILED
          Step st3 [0] look up order A2
        Run r3 COMPLETED
          Step st4 [0] look up order A2
          Step st5 [1] issue refund

path up from st5: st5 -> r3 -> t2 -> s1 -> av1 -> a1
```

Read the tree like a story. The customer sent two messages. The first one took one attempt. The second one took **two attempts**: `r2` failed after one step, and `r3` redid it and finished. That story is only visible because Turn and Run are separate levels. The last line is walking **up** from the final step to the agent, using nothing but the parent ids.

### Try it: editing a version

```python
import hierarchy as h
try:
    h.VERSIONS["av1"].instructions = "Refunds up to 5000."
except Exception as e:
    print(type(e).__name__, "->", e)
```

```
FrozenInstanceError -> cannot assign to field 'instructions'
```

`frozen=True` makes Python refuse the edit. That's the immutability rule enforced in code, not left as a convention.

### Break it on purpose

**Break 1: remove the Turn level.** Runs hang directly off the session:

```python
import hierarchy as h
flat_runs = {r.id: {"session_id": "s1", "status": r.status} for r in h.RUNS.values()}
print("runs in session s1:", list(flat_runs))
print("Which of them were attempts at the SECOND message?  CANNOT TELL - no record links a run to a message")
print("How many attempts did message 2 take?               CANNOT TELL")
```

```
runs in session s1: ['r1', 'r2', 'r3']
Which of them were attempts at the SECOND message?  CANNOT TELL - no record links a run to a message
How many attempts did message 2 take?               CANNOT TELL
```

You'd have three runs in a pile. You could guess from timestamps, but a guess is not a record. The Turn level is what groups attempts under the message they were for.

**Break 2: a version that can be edited in place.** Same data, but the version isn't frozen:

```python
from dataclasses import dataclass
import hierarchy as h

@dataclass                       # NOT frozen
class LooseVersion:
    id: str
    instructions: str

v = LooseVersion("av1", "Be polite. Refunds up to 100.")
print("run r3 was produced with  :", v.instructions)
v.instructions = "Refunds up to 5000."          # someone 'improves' the agent next week
print("record of what r3 used now:", v.instructions)
```

```
run r3 was produced with  : Be polite. Refunds up to 100.
record of what r3 used now: Refunds up to 5000.
```

Run `r3` refunded under a limit of 100. After the edit, its own history says it was allowed up to 5000. Nobody changed `r3`, but the record of what it ran under is now false. That's why versions are frozen and changes mean a new version.

---

## Real-life examples

### Software releases

An app has a name, and it has releases: 2.3.0, 2.3.1. A release is frozen. If a customer reports a bug, support asks "which version?", because it must be a fixed thing. If releases could be edited after shipping, "version 2.3.1" would mean nothing. AgentVersion is the same idea.

### Chat apps

A conversation holds messages. Pressing "regenerate" makes another attempt at the same message. Message and attempt are separate things (Turn and Run). Inside one attempt, the app may search, read and write, and each of those is a Step.

### Temporal

Your roadmap names Temporal as prior art. It keeps a stable id for the logical job and a separate run id for each execution of it, so retries and restarts have somewhere to live. That is the Turn and Run split.

---

## Where this lands in our build

**1. This is P1.** The roadmap lists `Session → Turn → Run → Step` as part of P1 and says multi-turn was undefined without it.

**2. Run is where lifecycle lives.** The P7 state machine (`CREATED`, `RUNNING`, `WAITING`, `PAUSED`, `COMPLETED`, `FAILED`, `CANCELLED`) belongs to a Run. A Session and a Turn don't have this lifecycle.

**3. AgentVersion immutability is P19.** The roadmap says "versions immutable, every run reproducible against one." Break 2 is the reason.

**4. The log and trace use these ids.** Events are tied to a `run_id` and `step_id` at P6. At P10, "why did it do that?" is answered by walking this tree.

**5. The real schemas have shortcuts.** In your schema, a Run also stores `session_id`, `agent_id` and `version_id`, not just `turn_id`. The toy code only stores the direct parent. 4.3 explains what is owned and what is merely referenced.

**6. Add to your open-questions list:**

```
A Turn can have several Runs. Which Run's answer is "the" answer for that Turn: the last COMPLETED one, or one explicitly marked? (Resolves around P7.)
```

**7. Log it in `DECISIONS.md`:**

```
Ownership is Agent > AgentVersion > Session > Turn > Run > Step, each child holds its parent's id; AgentVersion is immutable and a config change creates a new version — keeps every run explainable and reproducible.
```

---

## Say it to your manager

> "Everything in the system sits in one tree. An Agent has versions, a version has sessions, a session has turns, a turn has runs, and a run has steps. Each record stores its parent's id, so we can walk from any step up to the agent or down from an agent to every step. A turn is separate from a run so that retries don't overwrite each other. And versions are immutable, so every run can be traced to the exact configuration that produced it."

---

## Common mistakes

- **Merging Turn and Run.** Retries then overwrite or hide the failed attempt (Break 1).
- **Editing an AgentVersion in place.** History quietly becomes false (Break 2).
- **Letting a record exist without a parent.** Orphans can't be traced or cleaned up.
- **Using "run" to mean a whole conversation.** That's a Session.
- **Skipping the Step level.** You lose the logical index that replay depends on.

---

