
Back to the kitchen.

The kitchen has a back door. Everything inside that door is yours: the staff, the recipes, the stove, the notebook. Everything outside is not: the farmer who delivers tomatoes, the gas company, the customers, the weather.

You don't control the farmer. The tomatoes can arrive late, rotten, or not at all. The gas can be cut. So at the back door you do two things:

1. You **check every delivery in** and write it in the delivery book: what came, when, from whom.
2. You **never assume the outside will behave**.

Later, if someone asks "why did we run out of tomatoes on Tuesday?", the delivery book answers it. You can even re-run Tuesday on paper using only the book, without calling the farmer again.

That's the boundary. **Inside is what you control. Outside is what you only talk to. Everything that crosses the door gets written down.**

---

## Simple technical version

An agent has an **inside** and an **outside**.

```
  OUTSIDE (not ours)                 THE BOUNDARY                INSIDE (ours)
  ──────────────────                 ────────────                ─────────────

  the caller / your app      ──►   public API, task in    ──►   ┐
  a human approver           ──►   human input            ──►   │
                                                                │
  the model vendor (Groq)    ◄──►  ModelProvider          ◄──►  │   loop
  the real world (bank,      ◄──►  ToolExecutor           ◄──►  │   context builder
   email, files, web)                                           │   validator
  TinySeal (at P28)          ◄──►  PolicyGate             ◄──►  │   budget ledger
  the clock, random numbers  ──►   injected "ports"       ──►   │   state projection
  the database / disk        ◄──►  EventLog interface     ◄──►  ┘   lifecycle, recovery
```

Left of the line, things can fail, lie, change, or take forever. Right of the line, everything is **our code**, and it should behave the same way every time you run it on the same inputs.

### What is inside and what is outside

|Inside (we write it, we can test it)|Outside (we only talk to it)|
|---|---|
|The loop|The model vendor and its network|
|Context builder and renderer|The real effect of a tool (the bank, the inbox, the file system)|
|Tool validator and registry|The caller of the agent|
|`PolicyGate` interface|TinySeal (the thing behind the gate, at P28)|
|Budget ledger|A human who has to approve something|
|State projection|The clock|
|Lifecycle, recovery logic|Random numbers and generated ids|
|`EventLog` interface|The actual database engine and disk|

Two of these are easy to get wrong:

- **A tool's Python function is inside your repo, but its effect is outside.** The `ToolExecutor` is inside. The bank it calls is not.
- **The `PolicyGate` is inside, but what sits behind it is outside.** Agent Core only knows the gate's interface. That is why TinySeal can plug in at P28 without a rewrite.

### Three rules for the boundary

**Rule 1: Everything that crosses is a contract.** Data goes in and data comes out, in a shape you defined and can save. That is the point of the P1 contracts: `ModelRequest`, `ActionProposal`, `ToolResult` and the rest are the shapes of things crossing the door.

**Rule 2: Anything from outside is untrusted and can fail.** Slow, malformed, missing, wrong, or hostile. Every crossing needs a plan for each of those.

**Rule 3: All the unpredictability enters at the boundary, and gets recorded there.** This is the one that makes replay possible.

### Rule 3 in detail: where unpredictability comes from

Inside code is predictable. If you give it the same inputs, it gives the same outputs. The unpredictable things are all outside:

|Source|Why you can't just ask for it again|How we handle it|
|---|---|---|
|Model reply|It can be different every call|Record it. Replay reads it from the log|
|Tool result|The world has changed|Record it|
|Current time|It is a different time on replay|Time is a boundary port. Record it|
|Random numbers, new ids|A different value each run|Also ports. Record them|
|Human answer|Can't be re-asked|Record it|
|Failure from outside|May not repeat|Record the failure too|

Now look at what that buys you. If inside code is predictable and every outside value is in the log, then **you can re-run a whole agent run from the log alone, without calling anything outside.** That is exactly what your three modes are:

|Mode|Outside calls|Inside code|
|---|---|---|
|REPLAY|None. Every outside value comes from the log|Same code, same log|
|RESUME|Allowed, for the new steps|Continues from the log|
|EVAL|None. Recorded responses stand in for the real ones|New code, old log|

The boundary is what lets all three exist in code. Without it, there's nowhere to put the switch.

### Every crossing, one table

|Crossing|Direction|Contract|What can go wrong|Recorded as|
|---|---|---|---|---|
|Task from the caller|In|Public API (P1)|Missing fields, wrong tenant|`RunCreated`|
|Model call|Out, then in|`ModelRequest` / `ModelResponse`|Timeout, 429, invalid JSON|`ModelRequested`, `ModelResponded`|
|Tool call|Out, then in|`ToolCall` / `ToolResult`|Timeout, error, crash mid-effect|`ToolCallIntended`, `ToolCompleted`, `UnknownOutcome`|
|Policy check|Out, then in|`ActionProposal`|Denied, or no answer|`ActionProposed` and the verdict|
|Human approval|In, later|Human input|Never answers, answers after restart|`HumanInputRequested` and the reply (P20)|
|Storage|Out|`EventLog`|Disk full, DB down|The write itself fails loudly|

---

## Code

Save as `boundary.py`. The idea is small: one object, `Boundary`, is the only way inside code can reach the outside. In live mode it makes the real call and records the answer. In replay mode it reads the answer from the log and makes no call.

```python
import time

OUTSIDE_CALLS = 0          # how many times we really touched the outside world


# ---------- THE BOUNDARY: every outside value passes through here ----------
class Boundary:
    def __init__(self, mode, log):
        self.mode = mode           # "live" or "replay"
        self.log = log
        self.pos = 0

    def cross(self, kind, fn, *args):
        if self.mode == "replay":
            rec = self.log[self.pos]
            self.pos += 1
            assert rec["kind"] == kind, f"log says {rec['kind']}, code asked {kind}"
            if "error" in rec:
                raise RuntimeError(rec["error"])      # outside failures replay too
            return rec["value"]

        try:
            value = fn(*args)                         # the ONLY real contact
        except Exception as e:
            self.log.append({"kind": kind, "error": str(e)})
            raise
        self.log.append({"kind": kind, "value": value})
        return value


# ---------- THE OUTSIDE WORLD (we do not control any of this) ----------
def clock():
    global OUTSIDE_CALLS; OUTSIDE_CALLS += 1
    return time.time()

def fake_model(question):
    global OUTSIDE_CALLS; OUTSIDE_CALLS += 1
    return {"tool": "get_weather", "arg": "Pune"}

def get_weather(city):
    global OUTSIDE_CALLS; OUTSIDE_CALLS += 1
    data = {"pune": "31°C, clear"}
    if city.lower() not in data:
        raise RuntimeError(f"no data for {city}")
    return data[city.lower()]


# ---------- INSIDE THE AGENT: must give the same result for the same log ----------
def agent(b, city_override=None):
    started = b.cross("clock", clock)
    reply = b.cross("model", fake_model, "weather in Pune?")
    result = b.cross("tool", get_weather, city_override or reply["arg"])
    return f"{result} (asked at {started:.2f})"


if __name__ == "__main__":
    log = []
    live = agent(Boundary("live", log))
    print("LIVE   :", live)
    print("         outside calls so far:", OUTSIDE_CALLS)

    time.sleep(1.1)
    before = OUTSIDE_CALLS
    again = agent(Boundary("replay", log))
    print("REPLAY :", again)
    print("         outside calls during replay:", OUTSIDE_CALLS - before)
    print("same answer:", live == again)
```

Run `python boundary.py`. The timestamp will be different on your machine, but everything else will match:

```
LIVE   : 31°C, clear (asked at 1791382315.23)
         outside calls so far: 3
REPLAY : 31°C, clear (asked at 1791382315.23)
         outside calls during replay: 0
same answer: True
```

Read it carefully. The live run touched the outside three times (clock, model, tool). The replay waited over a second, and it still got the **same timestamp**, with **zero** outside calls. The answer came entirely from the log.

### Break it on purpose

**Break 1: inside code reaches outside directly.** Someone adds a "finished at" time and calls `time.time()` inside the agent, skipping the boundary:

```python
import time, boundary
from boundary import Boundary

def agent_bad(b):
    reply = b.cross("model", boundary.fake_model, "weather in Pune?")
    result = b.cross("tool", boundary.get_weather, reply["arg"])
    return f"{result} (finished at {time.time():.2f})"     # reaches outside directly

log = []
live = agent_bad(Boundary("live", log))
time.sleep(1.1)
again = agent_bad(Boundary("replay", log))
print("LIVE  :", live)
print("REPLAY:", again)
print("same answer:", live == again)
```

```
LIVE  : 31°C, clear (finished at 1791382316.35)
REPLAY: 31°C, clear (finished at 1791382317.45)
same answer: False
```

The replay gave a different answer. Nothing crashed. The model and the tool were both replayed perfectly. One stray `time.time()` was enough to make the replay lie. This is a silent bug: it passes unless you compare outputs.

The fix is not "be careful." The fix is that **inside code has no way to reach the clock except through the boundary.**

**Try it: an outside failure replays with no call.** Make the tool fail, then replay:

```python
import boundary
from boundary import Boundary
log = []
try:
    boundary.agent(Boundary("live", log), city_override="Atlantis")
except RuntimeError as e:
    print("LIVE   : failed ->", e)
calls = boundary.OUTSIDE_CALLS
try:
    boundary.agent(Boundary("replay", log), city_override="Atlantis")
except RuntimeError as e:
    print("REPLAY : failed ->", e)
print("outside calls during replay:", boundary.OUTSIDE_CALLS - calls)
print("log:", log)
```

```
LIVE   : failed -> no data for Atlantis
REPLAY : failed -> no data for Atlantis
outside calls during replay: 0
log: [{'kind': 'clock', 'value': 1791382317.4664361}, {'kind': 'model', 'value': {'tool': 'get_weather', 'arg': 'Pune'}}, {'kind': 'tool', 'error': 'no data for Atlantis'}]
```

The failure is part of the log, and it comes back in the replay with no outside call. Failures from outside are facts to record, just like successes.

---

## Real-life examples

### How Claude does it

From the app's point of view, I'm outside. The loop around me is inside, and it talks to me through one interface. The pages I read, the files I open and the commands I request are all outside too. What comes back from them arrives as data, and the app decides what to keep. That separation is why the same conversation can be saved and shown again later without re-running anything.

### The prior art your roadmap points to

Temporal, Restate and DBOS, the systems the roadmap names as real comparison class, are built on exactly this split. The code that makes decisions (Temporal calls it a _workflow_) must be predictable. Anything that touches the outside world (Temporal calls it an _activity_) has its result recorded in the history, and on replay the recorded result is used instead of running it again. Agent Core is the same design: the loop is the workflow, and model calls and tool calls are the activities. Restate and DBOS use different names for the same idea.

### Where real systems leak

|Leak|Looks like|What breaks|
|---|---|---|
|Inside code calls `datetime.now()`|A timestamp in a prompt|Replay and golden runs differ|
|Inside code calls `uuid4()`|A fresh id on every run|Dedup keys change on retry|
|A tool reads a global config file|Config changes between runs|Run can't be explained from its log|
|Retry logic that asks the model again silently|A second call nobody recorded|Costs and replays are wrong|
|Reading the environment (`os.environ`) mid-run|A different value after restart|Resume behaves differently|

---

## Where this lands in our build

**1. The P1 contracts are the shapes of crossing the door.** Contracts that can't be saved are not contracts. The roadmap's P1 manual check, "hand-write one event of each type as JSON," is a check that every crossing can be recorded.

**2. The boundary is what makes P6 and P11 possible.** REPLAY, RESUME and EVAL are three settings of the switch you saw in `Boundary`. Golden Runs at P11 are replays of recorded outside values against new inside code. Break 1 is the bug that would silently ruin them.

**3. Clock, ids and randomness are injected ports.** Nothing in `runtime/` may call `time.time()`, `uuid4()` or `random` directly. They come through a port, and their values go in the log. A cheap safeguard is a test that fails if those names appear in `runtime/`.

**4. `api.py` is the outer boundary.** Callers import only `api.py`. That's what makes the P26 embedding and the P28 TinySeal integration plugs and not rewrites.

**5. TinySeal is outside.** Agent Core only knows the `PolicyGate` interface and the `ActionProposal` and `Observation` contracts. Neither side imports the other's internals.

**6. Roadmap P5 says "two clocks: wall time and logical step index. Replay needs the logical one."** That is Break 1 in the roadmap's own words. Wall time is outside. The step index is inside.

**7. Add to your open-questions list:**

```
A wall-time budget (P8) needs a clock. Is the clock a recorded port, or derived from event timestamps in the log? (Resolves at P8.)
Streaming partial output (P21) crosses the boundary before it is "settled." What exactly gets recorded, and when? (Resolves at P21.)
```

**8. Log it in `DECISIONS.md`:**

```
Clock, ids and randomness are injected boundary ports whose values are recorded in the log; runtime/ never calls time, uuid or random directly — keeps REPLAY and EVAL deterministic (P6, P11).
```

---

## Say it to your manager

> "An agent has an inside and an outside. Inside is our code: the loop, the checks, the budget, the state. Outside is everything we don't control: the model vendor, the real effects of tools, the clock, humans, and TinySeal. Everything that crosses between them goes through a defined contract and is recorded in the event log. That includes failures. Because all the unpredictability is recorded at the boundary, we can replay any run from the log alone, with no outside calls, and we can test new code against old recordings for free."

---

## Common mistakes

- **Calling `time.time()`, `uuid4()` or `random` inside the loop.** It's an unrecorded outside call (Break 1).
- **Treating a tool's code as "inside" and trusting it.** Its effect is outside.
- **Recording successes but not failures.** The failure is the thing you'll want to replay.
- **Letting the caller reach internals.** Then there is no boundary to keep stable.
- **Letting TinySeal's types leak into Agent Core.** The seam is `PolicyGate`, nothing else.
- **Believing "it replays fine" without comparing outputs.** Break 1 never crashes.

---

