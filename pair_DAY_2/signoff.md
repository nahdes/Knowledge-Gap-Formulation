# Signoff — Day 1

**Asker:** Nahom  
**Question:** At the token level, when a model using function-calling selects a tool, is the description string part of the selection computation — and if so, how — or is it processed only after the tool name token is already committed?  
**Explainer written by:** Mikias Dagem  
**Gap closure judgment:** Closed

---

Before this explainer I understood function-calling at the API level — you pass a `tools` array, the model returns a `tool_use` block. I could not say anything about what happens between those two events. The explainer closed the gap by naming the mechanism: tool schemas are serialized into the context window during prefill, and because the transformer's attention is computed across the full context before any token is generated, the description strings are causally upstream of the tool-name token's logit distribution. The description is not "read after the name is chosen" — it is part of the key-value representations that shape the probability of choosing that name in the first place.

The practically load-bearing consequence: description quality is not a UX nicety — it directly affects which tool gets selected. A vague description (`"calls HubSpot"`) produces a different logit distribution than a precise one (`"creates or updates a HubSpot contact given an email and a property dict; use this when you have a verified prospect email to write to CRM"`). I could not have made that argument before today because I did not know the mechanism that made it true.

The constrained-decoding section also answered a follow-on question I did not know I had: some inference engines apply a token mask so only valid tool names are possible at the name position, which means the model's learned preference is filtered by a hard constraint. That changes the calculus for when description quality matters most (open-ended generation) vs. when the name will be selected correctly regardless (constrained decoding on a small tool set).
