---
layout: default
title: Smart Predicted Outputs Demo
hero_image: false
---
<div class="section" id="demo">
    <h1>SMART PREDICTED OUTPUTS DEMO</h1>
    <p>Try the feature with this minimal curl script against your local vLLMX OpenAI endpoint:</p>
<pre><code class="language-bash">curl https://localhost:8000/v1/chat/completions \
  -H "content-type: application/json" \
  -d '{
    "model": "llama-1b",
    "messages": [
      {"role": "user", "content": "Write a Python function called average"}
    ],
    "prediction": {
      "type": "content",
      "content": "def average(values):\n    total = sum(values)\n    count = max(len(values), 1)\n    return total / count\n"
    },
    "max_completion_tokens": 64,
    "temperature": 0
  }'</code></pre>
    <p>The server resumes proposing even if your local edit tweaks the mid-section of the function. Check your logs for alignment hits and accepted token bursts.</p>
</div>
