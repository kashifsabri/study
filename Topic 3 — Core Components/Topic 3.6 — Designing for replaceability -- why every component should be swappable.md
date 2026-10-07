

Think of a wall socket and a kettle.

The kettle has a plug. The wall has a socket. The two agree on a **shape**. Because of that, the kettle works in any house. It doesn't know or care whether the power comes from coal, solar or a generator. And the power company can change how it makes electricity without anyone touching their kettle.

The socket shape is the **contract**. Everything on either side is replaceable, as long as it fits the shape.

Now imagine a kettle hard-wired into the wall of one specific house. It works fine, until the house is sold, or the wiring changes, or you want to test the kettle somewhere else. Then you're cutting into the wall.

That's replaceability. **Depend on the shape of the socket, never on what's behind the wall.**

---

## Simple technical version

A replaceable component has three parts.

|Part|What it is|Who owns it|
|---|---|---|
|**Contract**|A small interface plus the data shapes that cross it|You|
|**Implementations**|Real code that fits the contract. There are at least two|You, or a vendor behind an adapter|
|**Conformance checks**|The _same_ tests run against every implementation|You|

```
                      THE LOOP
                         │
                         │  knows ONLY this:
                         ▼
               ┌───────────────────┐
               │  ModelProvider    │   ◄── the contract (the socket)
               │  generate(prompt) │
               │  → ModelResponse  │
               └─────────┬─────────┘
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     Groq adapter   Vendor B adapter   Fake provider      ◄── implementations (the plugs)
```

The loop never learns which one is plugged in. Swapping means changing one line of config.

### What should be swappable in Agent Core

|Component|Contract|Implementation 1|Implementation 2|Phase|
|---|---|---|---|---|
|Model|`ModelProvider`|Groq|A scripted fake (free tests)|P1, P2|
|Event log|`EventLog`|SQLite|Postgres, or in-memory for tests|P6, P26|
|Policy|`PolicyGate`|No-op (allows everything)|A real rule-based gate|P1|
|Tools|`ToolRegistry` entries|Real tools|Fixture tools|P4, P11|
|Clock and ids|Boundary ports (3.5)|Real|Fixed or recorded|P6|
|Context strategy|`ContextBuilder`|Simple builder|Budget-allocating builder|P2, P18|
|Reasoning strategy|Pluggable policy|ReAct|Plan-and-execute, if P11 proves it|P16|

### Why bother: five concrete reasons

1. **Vendors change.** Prices move, models get retired, a better one appears. If the vendor is behind a contract, a model swap is a config change.
2. **Tests need fakes.** A scripted provider costs nothing and behaves the same every run. You couldn't test the loop cheaply without one.
3. **EVAL needs stand-ins.** Replaying recorded responses (P11) is just one more implementation of the contract: one that reads from the log instead of calling out.
4. **Some parts arrive late.** A real policy engine isn't needed on day one. A no-op `PolicyGate` from P1 means the seam already exists, and the real one plugs in later.
5. **Scale comes later.** SQLite today, Postgres at P26, with the same `EventLog` contract.

Your roadmap says it directly about P3: hardening is "an implementation upgrade behind the P1 interface. Not a refactor, because P2 never depended on the vendor." That sentence is replaceability paying off.

### What makes a contract good

|Rule|Why|
|---|---|
|**Small.** A few methods, not twenty|Fewer things for each implementation to get wrong|
|**You own the data shapes.** Return your `ModelResponse`, never the vendor's object|Vendor shapes can't leak into the loop|
|**Translate inside the adapter.** The vendor's weird format dies there|The adapter's whole job is translation|
|**Required fields are required.** Tokens, model id, cost|Budgets (P8) and traces (P10) depend on them|
|**Same tests for every implementation**|Proves the swap is safe before you do it|

### The warning: don't swap what has only one possible form

Replaceability costs a little effort per contract. Spend it where you can name at least **two real implementations**: a real one and a fake, or today's and tomorrow's. Don't wrap things that will never change. The loop itself, for example, has one job, and "a pluggable loop" has no second implementation worth writing.

---

## Code

Save as `replaceable.py`. Two fake "vendors" return completely different raw shapes. Each adapter translates its own shape into one `ModelResponse`. The loop only knows `ModelResponse`.

```python
from dataclasses import dataclass
from typing import Protocol


# ---------- THE CONTRACT: the only thing the loop knows about models ----------
@dataclass
class ModelResponse:
    text: str
    tokens_in: int
    tokens_out: int
    model_id: str

class ModelProvider(Protocol):
    def generate(self, prompt: str) -> ModelResponse: ...


# ---------- IMPLEMENTATION 1: vendor A (its SDK returns an OpenAI-style dict) ----------
class VendorAProvider:
    def _raw(self, prompt):
        return {"choices": [{"message": {"content": f"A says: {prompt}"}}],
                "usage": {"prompt_tokens": len(prompt.split()), "completion_tokens": 4}}

    def generate(self, prompt):
        raw = self._raw(prompt)
        # WHY here: translating the vendor's shape is this class's whole job.
        return ModelResponse(raw["choices"][0]["message"]["content"],
                             raw["usage"]["prompt_tokens"],
                             raw["usage"]["completion_tokens"],
                             "vendor-a-1")


# ---------- IMPLEMENTATION 2: vendor B (a completely different shape) ----------
class VendorBProvider:
    def _raw(self, prompt):
        return {"completion": f"B says: {prompt}", "in": len(prompt.split()), "out": 5}

    def generate(self, prompt):
        raw = self._raw(prompt)
        return ModelResponse(raw["completion"], raw["in"], raw["out"], "vendor-b-2")


# ---------- IMPLEMENTATION 3: scripted fake, costs nothing ----------
class FakeProvider:
    def generate(self, prompt):
        return ModelResponse("fake answer", 0, 0, "fake-0")


PROVIDERS = {"a": VendorAProvider, "b": VendorBProvider, "fake": FakeProvider}

def make_provider(name):
    return PROVIDERS[name]()


# ---------- CONFORMANCE: the same checks, run against every implementation ----------
def conformance(provider):
    r = provider.generate("hello world")
    assert isinstance(r, ModelResponse), "must return ModelResponse"
    assert isinstance(r.text, str) and r.text, "text must be a non-empty string"
    assert isinstance(r.tokens_in, int) and isinstance(r.tokens_out, int), "token counts must be ints"
    assert r.model_id, "model_id must be set"
    return "ok"


# ---------- THE LOOP: knows only the contract ----------
def run(provider, question):
    r = provider.generate(question)
    return f"{r.text} [{r.model_id}, {r.tokens_in}+{r.tokens_out} tokens]"


if __name__ == "__main__":
    for name in ["a", "b", "fake"]:
        p = make_provider(name)
        print(f"{name:<5} conformance={conformance(p)}   {run(p, 'weather in Pune')}")
```

Run `python replaceable.py`:

```
a     conformance=ok   A says: weather in Pune [vendor-a-1, 3+4 tokens]
b     conformance=ok   B says: weather in Pune [vendor-b-2, 3+5 tokens]
fake  conformance=ok   fake answer [fake-0, 0+0 tokens]
```

Three completely different back ends and the same `run()` function. To swap, you change the name passed to `make_provider`. The loop code is identical.

### Break it on purpose

**Break 1: the loop reaches into one vendor's shape.** Someone is in a hurry and reads the raw vendor response inside the loop:

```python
import replaceable as r

def run_bad(provider, question):
    raw = provider._raw(question)
    return raw["choices"][0]["message"]["content"]       # vendor A's shape, hard-coded

for name in ["a", "b"]:
    try:
        print(name, "->", run_bad(r.make_provider(name), "weather in Pune"))
    except Exception as e:
        print(name, "-> CRASH:", type(e).__name__, e)
```

```
a -> A says: weather in Pune
b -> CRASH: KeyError 'choices'
```

It works perfectly with vendor A, so you'd never notice. The day you swap to B, the loop crashes. Replaceability isn't lost when you swap. It's lost the moment vendor details **leak** into the loop, because from then on the swap is a rewrite.

**Break 2: a new implementation that skips the checks.** You write vendor C and forget the token counts:

```python
import replaceable as r

class VendorCProvider:
    def generate(self, prompt):
        return r.ModelResponse("C says hi", None, None, "vendor-c-1")

p = VendorCProvider()
try:
    print(r.conformance(p))
except AssertionError as e:
    print("conformance FAILED:", e)
print("loop output anyway:", r.run(p, "weather in Pune"))
```

```
conformance FAILED: token counts must be ints
loop output anyway: C says hi [vendor-c-1, None+None tokens]
```

The loop doesn't complain. It happily prints `None+None tokens`. At P8 that `None` would either crash the budget ledger or quietly count as zero spend, and the budget would stop protecting you. The conformance check catches it **at the door**, before the loop ever sees it. That's why you run the same checks against every implementation.

---

## Real-life examples

### How Claude does it

In this very conversation the model was switched partway through. The conversation, your project files and your notes all carried on unchanged. That works because the model is a replaceable part sitting behind a contract: text in, text out. Everything around it stayed the same.

### How ChatGPT and the API world do it

Many model providers, Groq among them, offer endpoints that copy the same request and response shape, so a developer can often switch vendor by changing a base URL and a model name. That is a contract that grew up by common agreement. It's also a warning: "compatible" is not "identical," and the small differences are exactly where vendor details leak (Break 1).

### Where real systems get locked in

|Lock-in|How it happens|Cost later|
|---|---|---|
|Vendor objects used throughout the code|`response.choices[0]...` in ten files|A model swap becomes a big refactor|
|Vendor-only features used everywhere|Special tool-call format assumed|Can't change vendor without redesign|
|One storage engine's SQL in the loop|Queries scattered everywhere|Moving database means touching the runtime|
|A framework's classes as the core types|Your data shapes are the framework's|Leaving the framework means rewriting the core|

The last row is why the roadmap holds frameworks until P27. Adopting one early makes its types your contract.

---

## Where this lands in our build

**1. P1 defines the contract, P2 supplies the first implementation.** `ModelProvider` has `generate()` and `generate_structured()`. `groq_provider.py` is the only file that imports the Groq SDK, and `fake_provider.py` is the second implementation from day one.

**2. A cheap leak detector.** Search your repo for the vendor name: `grep -rn "groq" src/`. Every hit should be inside `model/groq_provider.py`. Any other hit is Break 1 waiting to happen.

**3. `ModelResponse` has required fields.** Tokens, cost, latency and model identity are first-class fields at P3. Break 2 shows why they must be non-optional: the budget ledger at P8 relies on them.

**4. EVAL is another implementation.** At P11, replaying recorded responses means plugging in a provider that reads from the log. The loop is unchanged. This is why replaceability and the boundary from 3.5 are really one idea.

**5. Retry and repair can wrap a provider.** One option for P3 is a wrapper that implements the same `ModelProvider` interface and adds retries around another provider. The loop can't tell the difference. Budget reserve and settle stay in the runtime, per the hub rule from 3.2.

**6. A model swap is a new agent version.** The roadmap's P19 says versions are immutable and every run is reproducible against one. So changing the model isn't editing an agent. It's creating a new `AgentVersion` with a different model config.

**7. Capabilities differ, so declare them.** Vendors don't all support structured output or tool calling the same way. The contract can include a small capabilities description, and the provider handles the fallback, so the loop doesn't branch on vendor names.

**8. Add to your open-questions list:**

```
Does the ModelProvider contract expose capabilities (structured output, tool calling, max context) or hide them behind generate_structured()? (Resolves around P3.)
```

**9. Log it in `DECISIONS.md`:**

```
Every external dependency (model, event log, policy, clock) sits behind a small contract with at least two implementations from day one, checked by the same conformance tests — vendor shapes never leak past their adapter.
```

---

## Say it to your manager

> "We build every external dependency behind a small contract that we own: the model, the event log, the policy gate, even the clock. The loop only knows the contract. Each contract has at least two implementations from the start, a real one and a fake one, and we run the same conformance tests against both. That gives us free tests, cheap replays, and a model swap that's a config change."

---

## Common mistakes

- **Letting a vendor's response object travel into the loop.** That is Break 1.
- **Skipping conformance tests for the second implementation.** That is Break 2.
- **Making fields optional so every implementation "fits."** The loop then has to guard against `None` everywhere.
- **Abstracting something with only one possible form.** Spend the effort where you can name two real implementations.
- **Adopting a framework's types as your core data shapes.** You lose the right to leave.
- **Treating "compatible API" as "same API."** Test it.

---

