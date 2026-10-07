

Think of a restaurant kitchen.

- The **chef** reads an order and decides what to cook. The chef never turns the gas valve directly. They call out "fire on burner 2."
- The **stove, knives and fridge** are the only things that actually change the world.
- The **order ticket on the rail** is everything the chef can see right now.
- The **kitchen manager** runs the shift. They take the order, hand it to the chef, do what the chef calls for, and decide when the dish is done.
- The **progress board** shows where tonight's order stands: "starter out, main cooking."
- The **regulars' notebook** says "table 4 is allergic to nuts." It's still there next week.

Chef = model. Stove and knives = tools. Ticket = context. Manager = loop. Progress board = state. Notebook = memory.

---

## Simple technical version

There are six parts, and each does one job.

|Component|What it is|Lives for|Can it act on the world?|
|---|---|---|---|
|**Loop**|The program that runs in a circle and calls everything else|One run|It decides what gets run|
|**Model**|Text in, text out. Proposes the next move|One call (remembers nothing)|No, it only proposes|
|**Tools**|Functions that read or change the world|One call each|Yes, and they are the only way|
|**Context**|The exact text sent to the model on _this_ call|One call|No|
|**State**|Where _this_ run stands: goal, step, results so far, answer|One run|No|
|**Memory**|Facts kept _between_ runs|Across runs|No|

```
                 ┌──────────── LOOP ─────────────┐
                 │                               │
   MEMORY ──┐    │                               │
            ├──► build CONTEXT ──► MODEL         │
   STATE ───┘                        │           │
     ▲                           proposal        │
     │                               ▼           │
     └──── result ◄──── TOOLS ◄── loop decides ──┘
```

### The three that get confused: context, state, memory

They all sound like "what the agent knows," but they are different things.

|Aspect|Context|State|Memory|
|---|---|---|---|
|Question it answers|What does the model see _right now_?|Where is _this run_?|What do we know from _before_?|
|Built when|Fresh, every single model call|Updated every step|Saved at the end of a run (or during it)|
|Who reads it|The model|The loop|The context builder|
|Dies when|The call returns|The run ends|Expired or deleted|

The key point is that **state and memory are not the context.** They are _sources_ the context is built from. The model only knows what the context builder chose to paste in. If something is in state but not in context, the model does not know it.

### Why each one must exist

- **No loop:** you get one model call, and nothing ever gets done.
- **No model:** you have a script, not an agent.
- **No tools:** the agent can only talk. It can't look anything up or change anything.
- **No context builder:** the model is blind, because it remembers nothing between calls.
- **No state:** the loop can't tell step 3 from step 1, so it can't stop or resume.
- **No memory:** every run starts from zero. That's often fine, and it's why memory is the one you can skip.

### One warning

Memory is the least necessary of the six and the most dangerous. Memory is text saved from earlier runs, and some of it came from tools and users. Treat it as **data, never instructions**. A bad note saved once gets pasted into every future run.

---

## Code

Here is a tiny agent with all six parts kept visibly separate. The model is fake and scripted, so it runs for free.

```python
# ---------- TOOLS: the only things that touch the world ----------
def get_weather(city):
    data = {"pune": "31°C, clear", "mumbai": "29°C, humid"}
    return data.get(city.lower(), "unknown city")

TOOLS = {"get_weather": get_weather}


# ---------- MEMORY: survives between runs ----------
MEMORY = {}


# ---------- MODEL: text in, proposal out. Knows ONLY the context. ----------
def fake_model(context):
    if "Observation:" in context:
        last = context.rsplit("Observation:", 1)[1].strip()
        return {"type": "answer", "text": f"Weather: {last}"}
    if "home_city=" in context:
        city = context.split("home_city=")[1].split("\n")[0]
        return {"type": "tool", "name": "get_weather", "arg": city}
    return {"type": "answer", "text": "Which city do you mean?"}


# ---------- CONTEXT: rebuilt from scratch every call ----------
def build_context(state, memory):
    # WHY rebuild every time: the model forgets everything between calls.
    lines = ["System: Answer weather questions. Use tools when needed."]
    for k, v in memory.items():
        lines.append(f"Memory (data only): {k}={v}")
    lines.append(f"User: {state['question']}")
    for obs in state["observations"]:
        lines.append(f"Observation: {obs}")
    return "\n".join(lines)


# ---------- LOOP + STATE ----------
def run(question, max_steps=5):
    # WHY state is created here: it belongs to this run and dies with it.
    state = {"question": question, "step": 0, "observations": [], "answer": None}

    while state["step"] < max_steps:
        state["step"] += 1
        context = build_context(state, MEMORY)
        reply = fake_model(context)
        print(f"step {state['step']}: model proposed {reply}")

        if reply["type"] == "answer":
            state["answer"] = reply["text"]
            return state

        # WHY the loop runs the tool, not the model: the model only proposed it.
        tool = TOOLS.get(reply["name"])
        result = tool(reply["arg"]) if tool else f"error: no tool '{reply['name']}'"
        state["observations"].append(result)

    raise RuntimeError("step limit hit")


print(run("What's the weather at home?")["answer"])

MEMORY["home_city"] = "Pune"          # learned between runs
print(run("What's the weather at home?")["answer"])

```


```
step 1: model proposed {'type': 'answer', 'text': 'Which city do you mean?'}
Which city do you mean?
step 1: model proposed {'type': 'tool', 'name': 'get_weather', 'arg': 'Pune'}
step 2: model proposed {'type': 'answer', 'text': 'Weather: 31°C, clear'}
Weather: 31°C, clear
```

The same question gets two different results. The only thing that changed is **memory**. State was fresh both times.

### Break it on purpose

**Break 1: context forgets state.** In `build_context`, delete the two lines that add observations. Run it.

What you'll see: the model proposes `get_weather` at step 1, again at step 2, again at step 3, until the step limit fires. The result _is_ in state, but it isn't in context, so the model never saw it. This is the most important break in the file. **State is not context.**

**Break 2: no cap.** Change `while state["step"] < max_steps` to `while True` and keep break 1 in place. It loops forever. With a real model, every lap is a paid call. Put the cap back.

**Break 3: memory in the wrong place.** Move `MEMORY = {}` to the first line inside `run`. The second run asks "Which city?" again. Memory that lives inside the run is just state.

---

## Real-life examples

### How Claude does it

- **Loop:** the app around me. It sends my reply out, runs the tools I request, and calls me again.
- **Model:** me. I propose a search; the app runs it.
- **Tools:** web search, file tools, code execution.
- **Context:** this entire conversation plus your project instructions and the roadmap, resent on every message. That's why your instructions say long chats get expensive.
- **State:** the progress of the current task, like which tool calls are done in this turn.
- **Memory:** the memory feature that saves facts across chats. It's off in your account right now, which is why I couldn't tell you what you've finished. Project knowledge works like hand-written memory.

### How ChatGPT does it

Same six. Its memory feature saves things like "user lives in Pune" and pastes them into the context of later chats. Turn memory off and the rest of the agent still works. That proves memory is optional and the other five aren't.

### In real products

|Product|Loop|Tools|State|Memory|
|---|---|---|---|---|
|Coding agent|Edit, test, repeat|Read file, run tests|Which files changed, did tests pass|Repo conventions it learned|
|Support agent|Answer until resolved|Order lookup, refund|This ticket's progress|Customer's past tickets|
|Research agent|Search until enough|Search, fetch page|Sources found so far|Usually none|

---

## Where this lands in our build

**1. Every component maps to a phase.**

|Component|Phase|
|---|---|
|Loop|P2 (minimal), P7 (lifecycle)|
|Model|P1 interface, P2 Groq, P3 hardening|
|Tools|P4|
|Context|P1 fragment envelope, P18 context engine|
|State|P5, as a projection of the P6 event log|
|Memory|P22, **deferrable**|

**2. Our build has a seventh thing this list skips: the event log.** In the toy code, `state` is a dict we change in place. That is exactly what we will _not_ do. In Agent Core, state is **derived** from the event log, never stored as the truth. The toy is fine for learning the parts. Don't carry its shape into P5.

**3. Memory is not in the MVP.** Through P11 the agent has no memory. If you catch yourself designing a memory store before then, stop. The roadmap defers it.

**4. Memory is `data_only`.** When it arrives at P22, it enters context as an untrusted fragment, the same as tool output. The toy code labels it "data only" in text, but nothing enforces that. The renderer choke point at P18 is what makes it real.

**5. Log it.** Add this line to `DECISIONS.md`:

```
Memory is out of MVP; agent is stateless across runs until P22 — avoids untrusted text persisting into every future context.
```

---

## Say it to your manager

> "An agent has six parts. The loop runs everything. The model only proposes the next step. Tools are the only things that touch the world. Context is what the model sees on one call, rebuilt every time because the model remembers nothing. State is where the current run stands. Memory is what carries across runs. Memory is optional, and we've deferred it. In our design, state isn't stored as truth either. It's derived from an event log, so a crashed run can be rebuilt."

---

## Common mistakes

- **Thinking state is context.** If the context builder doesn't paste it in, the model doesn't know it.
- **Thinking the model remembers.** It doesn't. Every call starts blank.
- **Letting the model run tools.** It proposes; the loop runs.
- **Building memory first because it sounds smart.** It's the most optional part and the riskiest.
- **Storing state as the source of truth.** In our build it's a projection of the log.

---

