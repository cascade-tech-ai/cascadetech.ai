---
layout: post
title: "Introducing Smart Predicted Outputs"
date: 2025-09-19
hero_image: false
---
OpenAI predicted outputs are great until they hit a typo or an extra clause—then the model forgets they ever existed.<!--more--> Think about asking for a JSON stub:

```
predicted: {"status":"ok","data":{"count":1}}
model:     {"status":"ok","data":{}}
```

The first divergence (`count` vs `{}`) causes the vanilla fast path to throw away the rest of the prediction even though the closing braces still match.

We fix that with the same Myers diff realignment every source-control nerd relies on. After a mismatch we walk both strings line by line, line up the next common anchor, and keep proposing tokens from there. It costs a tiny bit of CPU, but even three aligned lines are worth the price.

Code edits are the killer use case. Here’s a tiny Python tweak that stays mostly the same:

```diff
@@
-def average(values):
-    total = sum(values)
-    return total / len(values)
+def average(values):
+    total = sum(values)
+    count = max(len(values), 1)
+    return total / count
```

Traditional predicted outputs bail on line two. Smart Predicted Outputs snaps back at the `return` line and keeps accepting tokens, so speculative decoding still flies through the body.

Under the hood we reuse vLLM’s existing speculative decoding plumbing. Instead of a draft model, the proposer feeds the static prediction through Myers alignment, offers batches of tokens, and the target model accepts or rejects them exactly like any other speculative pass. No extra models, just better reuse of the text you already know.

API-wise nothing changes. The OpenAI-compatible server still takes `prediction`, `SamplingParams.predicted_outputs` still works, and you just get more hits before we fall back to normal decoding.

Want to see it in action? [Hit the live Smart Predicted Outputs sample]({{ '/smart-predicted-outputs-demo/' | relative_url }}).
