---
layout: post
title: "AI Wrote the Code. A Human Made It Clean."
categories: [scala, tech, ai]
---

How tuple pattern matching replaced nested if-else — and why code review still matters in the age of AI.

<!--more-->

I recently asked GitHub Copilot to add a new validation step to an existing Scala controller. The AI delivered working code in seconds. It passed tests. It compiled. It did exactly what I asked.

But when I reviewed what it actually *did* to the code, I realised: working isn't the same as good.

This is the story of a small refactoring that AI couldn't see — and why human code review remains essential when you're pairing with an AI assistant.

## The Starting Point

We had an API controller that validates incoming requests. Before processing, it checks that the domain and process capability in the URL path match what's in the request payload:

```scala
processRequest(context, json) { payload =>
  if (payload.domain == domain) {
    if (payload.processCapability == processCapability) {
      Right(())
    } else {
      Left(
        s"Path and payload process-capability mismatch: ($processCapability vs ${payload.processCapability})"
      )
    }
  } else {
    Left(
      s"Path and payload domain mismatch: ($domain vs ${payload.domain})"
    )
  }
}
```

Classic nested if-else. Two boolean conditions, three possible outcomes. It worked, it was readable enough, and nobody had a reason to touch it — until now.

---

## The Task: Add a New Validator

The requirement was simple: after the path/payload check passes, run an additional validation (a blocklist check) for a specific domain. I gave the task to Copilot.

Here's what the AI produced:

```scala
processRequest(context, json) { payload =>
  val pathPayloadValidation = if (payload.domain == domain) {
    if (payload.processCapability == processCapability) {
      Right(())
    } else {
      Left(
        s"Path and payload process-capability mismatch: ($processCapability vs ${payload.processCapability})"
      )
    }
  } else {
    Left(
      s"Path and payload domain mismatch: ($domain vs ${payload.domain})"
    )
  }

  pathPayloadValidation.flatMap(_ =>
    BlocklistValidator.validate(payload, domain, processCapability)
  )
}
```

It works. Tests pass. The AI took the existing nested if-else block, assigned it to a `val`, then chained the new validator with `.flatMap`. Minimal diff, minimal risk.

But look at what happened: the AI *preserved the existing structure*. It treated the nested if-else as a fixed block and bolted new logic around it. The AI didn't ask: "is this the right shape for this code now that it's growing?"

---

## The Human Refactoring

When I reviewed this, I saw something the AI missed: two boolean conditions with three outcomes is a textbook case for **tuple pattern matching**.

Here's the refactored version:

```scala
processRequest(context, json) { payload =>
  validatePathPayload(domain, processCapability, payload)
    .flatMap(_ => BlocklistValidator.validate(payload, domain, processCapability))
}

private def validatePathPayload(
  domain: String,
  processCapability: String,
  payload: CreateRequestInput
): Either[String, Unit] = {
  (payload.domain == domain, payload.processCapability == processCapability) match {
    case (false, _) =>
      Left(s"Path and payload domain mismatch: ($domain vs ${payload.domain})")
    case (true, false) =>
      Left(
        s"Path and payload process-capability mismatch: ($processCapability vs ${payload.processCapability})"
      )
    case (true, true) =>
      Right(())
  }
}
```

### What Changed

1. **Flat instead of nested.** The tuple `(domainMatches, capabilityMatches)` gives us a flat structure. Each case is at the same indentation level. No mental stack required.

2. **Extracted into a named method.** The validation logic now has a name (`validatePathPayload`) that communicates its purpose. The call site becomes a clean pipeline: validate path, then validate blocklist.

3. **Exhaustive pattern match.** The compiler ensures we've handled all combinations of `(Boolean, Boolean)`. With nested if-else, it's easy to miss a branch or accidentally flip the logic.

4. **Composable.** Adding a third condition? Just expand the tuple to a triple. Each case remains one line. With nested if-else, a third condition means another level of nesting — three levels deep for three booleans.

---

## The Pattern: When to Reach for Tuple Matching

Tuple pattern matching shines when you have **multiple independent boolean conditions** that together determine the outcome. The recipe:

```scala
(conditionA, conditionB, conditionC) match {
  case (false, _, _)     => // handle A failure (short-circuit)
  case (true, false, _)  => // handle B failure
  case (true, true, false) => // handle C failure
  case (true, true, true)  => // all good
}
```

Compare the nested equivalent:

```scala
if (conditionA) {
  if (conditionB) {
    if (conditionC) {
      // all good
    } else {
      // handle C failure
    }
  } else {
    // handle B failure
  }
} else {
  // handle A failure
}
```

The nested version buries the happy path at maximum depth. The pattern match puts all outcomes at the same level, with the happy path at the bottom — or wherever makes semantic sense.

### When NOT to use it

- **Single condition** — a plain `if/else` is fine.
- **Dependent conditions** — if condition B only makes sense when A is true, and this dependency is important to communicate, nesting might actually be clearer.
- **Many conditions** — beyond 3-4 booleans, a tuple match gets unwieldy too. Consider a validation pipeline (chain of `Either` with `.flatMap`) instead.

---

## Why the AI Didn't Do This

AI coding assistants are excellent at **local transformations** — they see the code in front of them and modify it to achieve the stated goal. Copilot saw nested if-else, needed to add a step after it, and did the smallest thing that works: wrap the existing block in a `val` and chain a `.flatMap`.

What the AI didn't do:

- **Question the existing structure.** It didn't ask "should this code look different now?"
- **Recognise a refactoring opportunity.** Adding new logic to existing code is exactly the moment to ask whether the existing code still has the right shape.
- **Apply taste.** Knowing that tuple pattern matching is *idiomatic Scala* for this situation requires judgment about readability, team conventions, and long-term maintainability.

This isn't a criticism of the tool. Copilot did exactly what was asked: add a blocklist validator. It delivered correct code. But correctness is table stakes — the human's job is to ensure the code is *good*.

---

## The Takeaway

AI accelerates the obvious work. It handles boilerplate, it generates tests, it wires up new functionality. But it operates within the shape of what already exists.

Refactoring — recognising that the code's structure should evolve as requirements grow — is still a human skill. It requires stepping back and asking: "now that this code does more, should it look different?"

So here's my workflow now:

1. **Let AI write the first pass.** It's fast, it's usually correct, it handles the boring parts.
2. **Review as if a junior wrote it.** The AI's code works, but does it follow the idioms of your language? Does it compose well with what's already there? Would you approve this in a code review?
3. **Refactor the shape, not just the logic.** When new code gets bolted onto old code, that's your signal. The structure might need to change.

The code that ships should look like a *human* wrote it — because a human reviewed it.

---

*The examples in this post are from a real PR. The AI generated functional code, and a human turned it into clean code. Both contributions were valuable — but they're different kinds of value.*
