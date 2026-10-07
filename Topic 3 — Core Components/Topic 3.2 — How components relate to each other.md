

Back to the kitchen.

The chef never walks to the fridge. The stove never shouts orders at the chef. The fridge never talks to the stove. **Everything goes through the manager.**

- The manager writes the ticket and hands it to the chef.
- The chef calls out what they want.
- The manager decides whether to do it, then uses the stove.
- The manager updates the progress board.
- Then the manager writes a fresh ticket.

If the stove started talking to the chef directly, the manager would lose track. Nobody could tell why a dish came out the way it did or what it cost.

That's the whole topic. **One hub, many spokes, and the spokes never talk to each other.**

---

## Simple technical version

Three rules describe how the six parts relate.

**Rule 1: Hub and spoke.** Only the loop calls things. The model, the tools and the context builder never call each other.

**Rule 2: The model has one door in and one door out.** The input is the context. The output is a proposal. Nothing else reaches the model, and nothing it does leaves except through that proposal.

**Rule 3: Data goes around the circle one way.** State and memory feed the context, the context feeds the model, the model's proposal goes to the loop, the loop runs a tool, and the result goes back into state.

```
                 STATE   MEMORY   TOOL SPECS
                    \      |       /
                     ▼     ▼      ▼
                  ┌─────────────────┐
                  │ CONTEXT BUILDER │
                  └────────▲────────┘
                           │ 1. build
   ┌───────┐  2. call  ┌───┴───┐  3. run   ┌───────┐
   │ MODEL │◄──────────┤ LOOP  ├──────────►│ TOOLS │
   │       ├──────────►│       │◄──────────┤       │
   └───────┘  proposal └───┬───┘  result   └───────┘
                           │ 4. update
                           ▼
                         STATE
```

Every arrow touches the loop. That's what makes the system understandable.

### Who reads, writes and calls what

|Component|Reads|Writes|Calls|
|---|---|---|---|
|Loop|State, the proposal|State|Context builder, model, tools|
|Context builder|State, memory, tool specs|The context|Nothing|
|Model|The context, and only the context|A proposal|Nothing|
|Tools|Their arguments|The world, and a result|Nothing|
|State|Nothing (it's data)|Nothing|Nothing|
|Memory|Nothing (it's data)|Nothing|Nothing|

Look at the "Calls" column. Only one row is not "Nothing."

### How the model knows tools exist

The model can't see your tool registry. It only knows a tool exists because the context builder wrote a description of it into the context. The tool's **code** lives with the loop. The tool's **description** travels through the context. Those are two separate paths, and if the description is missing the model never proposes the tool, even though the code is right there.

### Why these rules matter

- **Cost:** every model call goes through the loop, so the loop can count them all. One counter sees every paid call.
- **Debugging:** to answer "why did the model say that?", look at the context it got. If the context is the only input, the answer is always in there.
- **Safety:** every tool run goes through the loop, so there's one place to check, approve or refuse it.
- **Replay:** if you record what goes in and out at each arrow, you can rebuild the whole run.

Break any rule and you lose one of these. The code below shows exactly how.

---

## Code

This is the weather agent from 3.1 with two additions: a `call()` helper that records every edge, and tool descriptions passed through context.

```python
EDGES = []

def call(caller, callee, fn, *args):
    # WHY: every call between components goes through here, so we can
    # see exactly who called whom. This is a tiny trace.
    EDGES.append(f"{caller} -> {callee}")
    return fn(*args)


# ---------- TOOLS: code (here) and description (sent to the model) ----------
def get_weather(city):
    data = {"pune": "31°C, clear", "mumbai": "29°C, humid"}
    return data.get(city.lower(), "unknown city")

TOOLS = {"get_weather": get_weather}
TOOL_SPECS = {"get_weather": "get_weather(city) -> current weather"}


# ---------- MEMORY ----------
MEMORY = {"home_city": "Pune"}


# ---------- MODEL: one door in (context), one door out (proposal) ----------
def fake_model(context):
    if "Observation:" in context:
        last = context.rsplit("Observation:", 1)[1].strip()
        return {"type": "answer", "text": f"Weather: {last}"}
    if "get_weather" not in context:
        return {"type": "answer", "text": "I have no way to check the weather."}
    if "home_city=" in context:
        city = context.split("home_city=")[1].split("\n")[0]
        return {"type": "tool", "name": "get_weather", "arg": city}
    return {"type": "answer", "text": "Which city do you mean?"}


# ---------- CONTEXT BUILDER: the only thing that feeds the model ----------
def build_context(state, memory, tool_specs):
    lines = ["System: Answer weather questions. Use tools when needed."]
    lines.append("Tools: " + "; ".join(tool_specs.values()))
    for k, v in memory.items():
        lines.append(f"Memory (data only): {k}={v}")
    lines.append(f"User: {state['question']}")
    for obs in state["observations"]:
        lines.append(f"Observation: {obs}")
    return "\n".join(lines)


# ---------- LOOP: the hub. The only caller. ----------
def run(question, max_steps=5):
    state = {"question": question, "step": 0, "observations": []}

    while state["step"] < max_steps:
        state["step"] += 1
        context = call("loop", "context", build_context, state, MEMORY, TOOL_SPECS)
        reply = call("loop", "model", fake_model, context)

        if reply["type"] == "answer":
            return reply["text"]

        result = call("loop", "tool", TOOLS[reply["name"]], reply["arg"])
        state["observations"].append(result)

    raise RuntimeError("step limit hit")


print("ANSWER:", run("What's the weather at home?"))
print("\nWho called whom:")
for e in EDGES:
    print("  " + e)
```

```
ANSWER: Weather: 31°C, clear

Who called whom:
  loop -> context
  loop -> model
  loop -> tool
  loop -> context
  loop -> model
```

Every edge starts at `loop`. Two model calls, one tool call, all visible. That's the healthy shape.

### Break it on purpose

**Break 1: a tool secretly calls the model.** Replace `get_weather` with this "smarter" version that asks the model to tidy up its result:

```python
def get_weather(city):
    data = {"pune": "31°C, clear", "mumbai": "29°C, humid"}
    raw = data.get(city.lower(), "unknown city")
    return call("tool", "model", fake_model, f"Observation: {raw}")["text"]
```

What you'll see:

```
ANSWER: Weather: Weather: 31°C, clear

Who called whom:
  loop -> context
  loop -> model
  loop -> tool
  tool -> model
  loop -> context
  loop -> model
```

There's a `tool -> model` edge. The loop thinks it made two model calls, but there were three. A budget that counts in the loop would miss this one. That prompt also never went through the context builder, so nothing checked what went into it. In a real system that third call costs money that no one is counting. (The doubled "Weather:" is a small bonus bug that comes from the same cause.)

**Break 2: the model reads around the context.** Put `get_weather` back. Then, in `build_context`, delete the two memory lines. In `fake_model`, replace the `home_city=` check with:

```python
    if "home_city" in MEMORY:     # reading a global, not the context
        return {"type": "tool", "name": "get_weather", "arg": MEMORY["home_city"]}
```

What you'll see: the exact same answer. That's the danger. Now look at the context that was sent: it never mentions Pune. If you saved that context and later asked "why did the model choose Pune?", **the record can't answer you.** A real LLM can't read your globals, but the code around it can. For example, a model wrapper might quietly add things to the prompt. The same problem appears there.

**Break 3: tool code without its description.** Undo break 2. In `build_context`, delete the `Tools:` line.

What you'll see: `ANSWER: I have no way to check the weather.` The tool is still sitting in `TOOLS`, ready to run, but the model was never told it exists. Code and description travel on separate paths, and both have to arrive.

---

## Real-life examples

### How Claude does it

The app around me is the hub. When I want a web search, I can't run it myself. I write a request, the app decides whether to run it, and the results come back to me as new context. I also only know which tools I have because their descriptions are included in what I'm sent each time. If a tool isn't described there, it doesn't exist as far as I'm concerned.

### How ChatGPT does it

Same shape. A developer sends the function definitions along with every request. The model replies with a function call, the developer's code runs it, and the result is sent back as a new message. The model never runs the function and never talks to it directly.

### Hidden edges in real systems

These are common ways teams accidentally break the rules.

|Hidden edge|Looks like|What it breaks|
|---|---|---|
|Tool calls an LLM inside itself|A "summarize" or "smart search" tool|Cost isn't counted, and the prompt skips the context builder|
|Tool calls another tool directly|A "send report" tool that also emails|The second action never gets checked or approved|
|Prompt template reads a global|Config or user data added quietly at call time|Saved records can't explain the model's output|
|Model output goes straight to a tool|"Auto-run" with no loop in between|No place to say no|

---

## Where this lands in our build

**1. The loop is the only caller.** This is what makes budgets (P8) and the trace (P2, P10) possible. If every paid or risky edge passes one place, that place can count it and record it. Any edge that skips the loop is invisible to both.

**2. The model provider takes only a `ModelRequest`.** That rule is set at P1. If a saved `ModelRequest` can't explain the `ModelResponse` next to it, then golden runs in EVAL mode (P11) are broken, because replay depends on inputs fully explaining outputs. Break 2 is exactly this bug.

**3. Tool descriptions come from the registry (P4) and reach the model only through context (P1 fragments, P18 context engine).** Tool code never travels to the model.

**4. In our build, "loop → state" becomes "loop → event log → state."** The toy updates a dict. Agent Core writes an event, and state is rebuilt from the log (P5, P6). The hub rule doesn't change. The loop is still the only writer, but what it writes is an event.

```
   LOOP ──writes──► EVENT LOG ──projected into──► STATE ──read by──► CONTEXT BUILDER
```

**5. Add to your open-questions list:**

```
If a tool genuinely needs an LLM call (e.g. summarization), is that a nested run with its own budget, or does it go back through the parent's ModelProvider and ledger? (Resolves around P8 / P14.)
```

**6. Log it in `DECISIONS.md`:**

```
Only the loop calls the model, tools and context builder; components never call each other — keeps every costed and effectful edge visible to budgets (P8), trace (P10) and the PolicyGate (P1/P28).
```

---

## Say it to your manager

> "The loop is the hub, and everything else is a spoke. The model, the tools and the context builder never call each other. Only the loop calls them. The model has one input, the context, and one output, a proposal. That means every paid call and every action passes through one place, so we can count it, record it, and block it. It also means that if we save the context, we can always explain why the model did what it did."

---

## Common mistakes

- **Letting a tool call the model "just this once."** Now there's a cost and a prompt nobody is tracking.
- **Letting a tool call another tool.** The second action skips every check.
- **Adding things to the prompt outside the context builder.** Saved records stop explaining the output.
- **Registering a tool and forgetting its description.** The model can't use what it doesn't know about.
- **Treating "it still works" as proof the wiring is fine.** Break 2 gives the same answer and is still wrong.

---

