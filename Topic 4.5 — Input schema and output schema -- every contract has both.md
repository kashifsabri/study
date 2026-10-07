

Think of a currency-exchange counter.

**What goes in is checked.** The clerk only takes certain currencies, in whole notes, not torn, not counterfeit. If you push through a bus ticket, it's refused at the window, before anything happens.

**What comes out has a fixed shape.** You always get the same receipt: the amount handed over, the rate, the time. The accountant in the back office posts those receipts by reading fixed boxes. If the counter sometimes prints a receipt without the amount, the accountant can't post it, and the error is found weeks later.

Two rules, one on each side of the window:

- **Rule on the way in:** "This is what I accept."
- **Rule on the way out:** "This is what I promise to give back."

A contract with only the first rule protects the counter. A contract with both protects everyone who depends on it.

---

## Simple technical version

### What a schema is

A **schema** is a machine-checkable description of what valid data looks like: which fields exist, which are required, what type each one is, and any limits (like "at least 1").

Your roadmap uses **JSON Schema** for this. It's a standard format, and libraries can check data against it and list exactly what's wrong.

### Two schemas per contract

|Aspect|Input schema|Output schema|
|---|---|---|
|Says|What I accept|What I promise to return|
|Checked|**Before** running|**After** running, before the result is used|
|Protects|The thing being called, from bad requests|Everything downstream, from bad results|
|Who produces the data being checked|The model (unreliable)|The tool, i.e. the outside world (untrusted)|
|In your schemas|`ToolCall.args` is "validated against input_schema"|`ToolResult.output` is "validated against output_schema"|

Most people only write the first. The second matters just as much: a tool's output comes from outside your system (3.5), so it is **untrusted data that is about to enter your system**. You check it at the door like anything else.

### Where this applies: every contract

The title says "every contract has both." Here it is on the contracts you've already met:

|Contract|Input|Output|
|---|---|---|
|A tool (`ToolDefinition`)|Arguments the model supplies|The result the tool returns|
|`ModelProvider.generate_structured()`|The request, plus the shape you want back|An object that actually matches that shape|
|`PolicyGate`|An `ActionProposal`|A `PolicyDecision`|
|The public embedding API (P1)|A task from the caller|A run result|

If a contract has no output schema, then the code that uses its result is guessing.

### What a good schema checks, and what it doesn't

Look at the input schema used in the code below:

- `order_id` must be a non-empty string.
- `amount` must be an **integer** of at least **1**.
- Both are **required**.
- **No extra fields** (`additionalProperties: false`).

That last line catches typos. Without it, `{"ammount": 50}` could slip through with the real field missing.

A schema checks **shape**, not **permission.** "Is `amount` an integer ≥ 1?" is a schema question. "Is a refund of 5000 allowed for this user?" is a policy question (3.4, 3.7). Don't push business limits into the schema. They'll end up in two places and drift.

### Errors are written for the model

When input validation fails, the tool never runs, and the problems go back to the model as an observation. So they should be specific:

```
amount: 'fifty' is not of type 'integer'
```

That's something a model can fix on its next try. "Invalid input" is not.

### One honest limit of output validation

An output check happens **after** the tool ran. If the tool already moved the money and then returned garbage, the check can tell you the result is unusable. It **cannot undo the effect.** The code below shows exactly this.

---

## Code

You need the `jsonschema` library: `pip install jsonschema`. Save as `schemas.py`.

```python
from jsonschema import Draft202012Validator

BANK = {"balance": 1000}


# ---------- the two schemas for ONE tool: what goes in, what comes out ----------
REFUND_INPUT = {
    "type": "object",
    "properties": {
        "order_id": {"type": "string", "minLength": 1},
        "amount":   {"type": "integer", "minimum": 1},
    },
    "required": ["order_id", "amount"],
    "additionalProperties": False,
}

REFUND_OUTPUT = {
    "type": "object",
    "properties": {
        "order_id":        {"type": "string"},
        "refunded_amount": {"type": "integer", "minimum": 1},
    },
    "required": ["order_id", "refunded_amount"],
    "additionalProperties": False,
}


# ---------- the tools themselves ----------
def refund(order_id, amount):
    BANK["balance"] -= amount
    return {"order_id": order_id, "refunded_amount": amount}

def buggy_refund(order_id, amount):
    BANK["balance"] -= amount
    return {"status": "ok"}                      # forgot to return the amount

REFUND = {"name": "refund", "fn": refund,
          "input_schema": REFUND_INPUT, "output_schema": REFUND_OUTPUT}
BUGGY  = {"name": "refund", "fn": buggy_refund,
          "input_schema": REFUND_INPUT, "output_schema": REFUND_OUTPUT}


# ---------- validation: returns a list of plain-language problems ----------
def problems(schema, data):
    v = Draft202012Validator(schema)
    out = []
    for e in sorted(v.iter_errors(data), key=lambda e: str(list(e.absolute_path))):
        where = ".".join(str(p) for p in e.absolute_path) or "(whole object)"
        out.append(f"{where}: {e.message}")
    return out


# ---------- the boundary: check going in, check coming out ----------
def call_tool(tool, args):
    bad = problems(tool["input_schema"], args)
    if bad:
        return {"status": "INPUT_INVALID", "problems": bad}      # tool never runs

    output = tool["fn"](**args)

    bad = problems(tool["output_schema"], output)
    if bad:
        return {"status": "OUTPUT_INVALID", "problems": bad}     # bad data never enters the system
    return {"status": "SUCCESS", "output": output}


if __name__ == "__main__":
    cases = [
        ("1. good call",            REFUND, {"order_id": "A1", "amount": 50}),
        ("2. wrong type",           REFUND, {"order_id": "A1", "amount": "fifty"}),
        ("3. missing + typo field", REFUND, {"order_id": "A1", "ammount": 50}),
        ("4. negative amount",      REFUND, {"order_id": "A1", "amount": -500}),
        ("5. tool returns junk",    BUGGY,  {"order_id": "A1", "amount": 50}),
    ]
    for label, tool, args in cases:
        r = call_tool(tool, args)
        print(f"{label}\n  args  : {args}\n  status: {r['status']}")
        for p in r.get("problems", []):
            print(f"    - {p}")
        if "output" in r:
            print("  output:", r["output"])
        print()
    print("balance:", BANK["balance"])
```

Run `python schemas.py`:

```
1. good call
  args  : {'order_id': 'A1', 'amount': 50}
  status: SUCCESS
  output: {'order_id': 'A1', 'refunded_amount': 50}

2. wrong type
  args  : {'order_id': 'A1', 'amount': 'fifty'}
  status: INPUT_INVALID
    - amount: 'fifty' is not of type 'integer'

3. missing + typo field
  args  : {'order_id': 'A1', 'ammount': 50}
  status: INPUT_INVALID
    - (whole object): 'amount' is a required property
    - (whole object): Additional properties are not allowed ('ammount' was unexpected)

4. negative amount
  args  : {'order_id': 'A1', 'amount': -500}
  status: INPUT_INVALID
    - amount: -500 is less than the minimum of 1

5. tool returns junk
  args  : {'order_id': 'A1', 'amount': 50}
  status: OUTPUT_INVALID
    - (whole object): 'order_id' is a required property
    - (whole object): 'refunded_amount' is a required property
    - (whole object): Additional properties are not allowed ('status' was unexpected)

balance: 900
```

How to read it:

- **Cases 2, 3 and 4** are rejected on the way in. The tool never ran, and each message names the exact problem. Case 3 shows both problems at once: the real field is missing, and the typo was caught.
- **Case 5** is rejected on the way out. The tool ran, then returned `{"status": "ok"}`, which doesn't match what it promised.
- **Look at the balance.** It is **900**. Case 1 moved 50, and **case 5 moved another 50**. The output check caught the bad result, but the money had already gone. This is the limit mentioned above: an output failure on an effectful tool does _not_ mean "nothing happened."

### Break it on purpose

**Break 1: skip the input check.** Call the tool directly with a negative amount:

```python
import schemas as s
s.BANK["balance"] = 1000
args = {"order_id": "A1", "amount": -500}
out = s.REFUND["fn"](**args)                 # no input validation
print("tool returned:", out)
print("balance      :", s.BANK["balance"], "(a 'refund' that ADDED money)")
```

```
tool returned: {'order_id': 'A1', 'refunded_amount': -500}
balance      : 1500 (a 'refund' that ADDED money)
```

Nothing crashed. A "refund" of -500 took the balance **up** by 500. The tool's code never looked at the sign, because it trusted its caller. The schema's `"minimum": 1` is the one line that stops it.

**Break 2: skip the output check.** Use the buggy tool and trust whatever it returns:

```python
import schemas as s
out = s.BUGGY["fn"](order_id="A1", amount=50)    # no output validation
print("tool returned:", out)
message = f"Refund complete: {out.get('refunded_amount')} paid to order A1"
print("customer is told:", message)
```

```
tool returned: {'status': 'ok'}
customer is told: Refund complete: None paid to order A1
```

The bad result flowed straight into the next step, and the customer is told that `None` was paid. In a real system, the wrong value might be quietly saved, fed back to the model as a "fact", or used in a calculation. The output schema stops that at the door.

---

## Real-life examples

### How Claude and ChatGPT do it

When developers give a model a tool, they describe its arguments with a JSON Schema, and the model's tool request has to fit that shape. Many providers can also constrain a model's reply to a schema. Even then, a careful runtime checks again itself, because the contract has to hold no matter which provider is plugged in (3.6).

### Web APIs

Every serious web API publishes what a request must look like and what a response will look like (OpenAPI is the common format for this). Teams that skip the response half find out when a field quietly changes and a client breaks.

### Where real systems go wrong

|Gap|What happens|
|---|---|
|No input schema|A bad value reaches the tool, which does something silently wrong (Break 1)|
|No output schema|Garbage flows into the next step as if it were fact (Break 2)|
|Extra fields allowed|A typo'd field name is ignored, and the real one is missing|
|Silent type conversion (`"50"` becomes `50`)|Model mistakes get hidden, and behavior drifts|
|Vague error messages|The model can't fix its mistake and retries blindly|
|Business limits written into schemas|The same rule lives in two places and drifts|

---

## Where this lands in our build

**1. This is P1 and P4.** P1 defines the contracts. P4 gives each tool a `ToolDefinition` with both schemas, and `ToolValidator` does JSON Schema validation. Your schema notes already say it: `ToolCall.args` against `input_schema`, `ToolResult.output` against `output_schema`.

**2. Input failure is `ToolValidationError`.** It's in the P9 failure list. Its recovery should be "retry with repair": give the model the specific problems and let it fix its arguments. Cases 2, 3 and 4 are that path.

**3. `generate_structured()` is the same idea for the model.** The roadmap's P3 "Understand" line is that structured output from a text model is a reliability problem and not a parsing problem. The model's reply has an output schema too, and the provider checks it.

**4. Output validation protects the context.** A tool's output becomes a `data_only` fragment (4.4). Validating it first means malformed data never reaches the next prompt.

**5. Limits stay out of the schema.** Consistent with 3.4: the schema checks shape. The refund limit stays in policy. If you find a business rule in a schema, move it.

**6. Schemas change over time.** When a tool's schema changes, that's a new version. `ToolCall` records `tool_version` for this reason, and 4.7 covers how versioning works.

**7. Add to your open-questions list:**

```
When an effectful tool's output fails its output schema (case 5), which ToolResult status do we record: ERROR or UNKNOWN? The effect may have happened. The recovery path depends on the tool's idempotency. (Resolves at P9.)
```

**8. Log it in `DECISIONS.md`:**

```
Every contract carries an input schema and an output schema; tool args are validated before execution and tool output before it enters context; schemas check shape only, limits and permissions live in policy; no silent type coercion.
```

---

## Say it to your manager

> "Every contract has two schemas. The input schema says what we accept, and the output schema says what we promise to return. We check the input before running anything, so a bad request never reaches the tool. We check the output before using it, because tool results come from outside our system and are untrusted. Errors say exactly what's wrong so the model can fix it. Schemas check shape only. Business limits live in policy, so each rule has one home. And we know an output check can't undo something the tool already did, so that case has its own recovery path."

---

## Common mistakes

- **Writing only an input schema.** Results then enter the system unchecked (Break 2).
- **Trusting the caller.** The tool should be protected by its schema, not by hoping (Break 1).
- **Allowing extra fields.** Typos then pass silently.
- **Converting types quietly.** It hides the model's mistakes.
- **Putting limits in the schema.** They belong in policy.
- **Assuming "output invalid" means "nothing happened."** For an effectful tool, the effect may have already occurred.
- **Returning vague errors.** The model can't repair what it can't read.

---

**Next:** 4.6 — Why contracts you persist are different from classes you instantiate.