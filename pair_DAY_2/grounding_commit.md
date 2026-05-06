# Grounding Commit — Day 1

**Asker:** Nahom  
**Artifact edited:** `KNOWLEDGE.md` — §Compose stage  
**Type of edit:** Added a defended paragraph explaining the text-completion vs. function-calling trade-off with a mechanistic argument.

---

## What changed

Added the following paragraph to the Compose stage section of `KNOWLEDGE.md`:

> **Why text-completion, not function-calling.** `compose_email` sends a plain chat-completion request with no `tools` parameter. The decision is deliberate: the output schema (Subject line + HTML body under 120 words) is simple and stable enough that a string parser is more reliable than a tool-selection round-trip. Function-calling would be the right choice if the compose stage needed to *select between* multiple actions based on brief content — for example, choosing between `write_email`, `write_sms`, or `request_clarification`. It is not the right choice for a single, fixed output format. Mechanistically, tool schemas are serialized into the context window and attend into every generated token during prefill; the description and parameter fields of each schema consume context and add latency with no benefit when only one output shape is ever needed. If the compose stage is extended to support multiple output actions, the switch to function-calling should happen at that point, and tool descriptions should be written to be precise about the action boundary — not the implementation — because description strings are causally upstream of the tool-name logit.

## Why this edit

Before today's explainer I could describe this choice at the API level ("we don't pass a `tools` array") but not defend it mechanistically. The explainer revealed that tool schemas consume context and attend into generation even for the tokens that precede the tool name. That made the existing choice *more* correct than I knew — there is a real cost to adding schemas when you do not need tool selection — and gave me the language to explain it in a way that would hold up under engineering review.
