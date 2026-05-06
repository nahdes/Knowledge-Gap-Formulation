# Day 1 Question — Agent and Tool-Use Internals

**Asker:** Nahom  
**Topic:** Agent and tool-use internals  
**Subtopic:** How function-calling works at the token level

---

## The Question

At the token level, when a model using function-calling selects a tool, is the description string part of the selection computation — and if so, how — or is it processed only after the tool name token is already committed?

---

## Artifact connection

`agent/hubspot_mcp.py` calls HubSpot tools by bare name strings (`hubspot-create-contact`, `hubspot-update-contact`) with no description exposed to the model — because in my Week 10 system the LLM never makes the selection; Python calls the tool deterministically.

If I refactored to let the model choose tools, I would not know whether to invest effort in writing good descriptions or just get the tool name right, because I don't know which one actually drives the selection token. Closing this gap produces a concrete edit to `KNOWLEDGE.md §Compose` explaining the text-completion vs. function-calling trade-off with a mechanistic argument, not just an API-level one.

---

## Why it generalizes

Every FDE writing tool descriptions for an agent is implicitly betting on whether descriptions influence tool selection or only argument generation. Getting that wrong means either wasted effort on descriptions that don't affect routing, or under-specified names that cause silent misroutes.
