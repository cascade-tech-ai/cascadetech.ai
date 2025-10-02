---
layout: post
title: "VLLM Predicted Outputs"
date: 2025-09-19
hero_image: false
---

The primary focus of Cascade Technologies is to enable the development of realtime llm agents, and one of the most
important requirements for llm agents is to be low latency, to process prompts quickly and return their results. The
biggest challenge to improving latency in llm generations is their auto-regressive nature, each token generated needs to 
know its preceding token before it can be generated, so all output tokens must be generated serially.

Llm prompts are generally processed in two phases:
- First the input tokens are processed <strong>in parallel</strong> very quickly, thousands of tokens may all be
processed in hundreds of milliseconds.  They can be processed so quickly because they can be processed in parallel.  
They can be processed in parallel because, unlike output tokens, each token already knows all the tokens that precede it.
- Next the output tokens are processed one at a time often at 10 or more milliseconds per token, 100x or more times
slower than their input tokens, even though the actual math and weights that process them are identical to input.

<strong>tldr:</strong> Predicted Outputs uses a prediction about the contents of the model's output to allow 
output tokens to be processed in parallel.  When the predictions are even partially correct it can dramatically reduce
the latency of llm generations.

How it works:
* The user provides a prediction for the output of their llm request.  This can be many things, but an
easy example to understand is the case of code modification, in which case you would provide the
original code as the prediction.
* As long as the prediction is aligned with the output, we are able to process our llm generation
in parallel, which can make it orders of magnitude faster.
* When the prediction diverges from the output, we use standard diff algorithms to realign the prediction and continue
processing the output in parallel.

Predicted Outputs is already supported in the OpenAI API so you can drop it in and use it with very few changes to your
client code.

Here is a simple example:

<div class="diff-block">
  <strong>Prediction</strong>
  <pre><code>Predicted Outputs
Predicted outputs are great because they
allow you to use knowledge about the likely
output to speed up generation.</code></pre>
</div>

<div class="diff-block">
  <strong>Model output with Predicted Outputs</strong>
  <pre><code><span class="diff-line"><span class="diff-match">Predicted Outputs</span><span class="diff-miss">:</span></span>
<span class="diff-line diff-miss">Predicted outputs are great because they</span>
<span class="diff-line diff-match">allow you to use knowledge about the likely</span>
<span class="diff-line diff-match">output to speed up generation.</span></code></pre>
</div>

As you can see, Predicted Outputs needs one identical line to realign, but then carries on with the prediction for
the rest of the generation.  The tokens in green are generated nearly "for free".

Because running LLM model passes is so slow, and computing text diffs are so fast, matching just a single line can often
provide speed benefits over the no-prediction base case, and there are many use cases where you can frequently match
much more:

* Coding agents modifying sections of code. This is the easiest win for Predicted Outputs, given the ease of
  prediction and frequency of prediction matching.
* Implementing true Structured Outputs can often be very complicated, but passing an example of your output as a
  prediction can often get many of the speed benefits with much less difficulty.
* Modifying documents.
* Updating state agent state dicts / memory.

## Implementation

VLLM provides an easy mechanism to integrate Predicted Outputs in their speculative decoding system. Instead of a
draft model, we simply use a static text prediction and the diff algorithm mentioned before. We keep a cursor to the
position in the static text where we currently are, and as long as the prediction matches, we propose chunks of text to
the system. When the generation diverges from the prediction we use the Myers diff algorithm to attempt to realign.

User's provide predictions via the <a href="https://platform.openai.com/docs/guides/predicted-outputs">standard OpenAI
API</a>, and the rest is automatic.

You can use Predicted Outputs yourself today via Cascade
Technology's <a href="https://github.com/cascade-tech-ai/vllmx">vllm fork</a>, or you can try it yourself on our servers
here: <a href="http://app.cascadetech.ai">http://app.cascadetech.ai</a>

<style>
.diff-block {
    margin: 4vh 0;
    font-family: 'JetBrains Mono', monospace;
    font-size: 0.8rem;
}
.diff-block pre {
    background: rgba(255, 255, 255, 0.03);
    padding: 1.5vh;
    border: 1px solid rgba(255, 255, 255, 0.08);
    overflow-x: auto;
    margin: 0 0 2vh;
}
.diff-block strong {
    display: block;
    margin-bottom: 1vh;
    letter-spacing: 0.05em;
    color: rgba(255, 255, 255, 0.6);
}
.diff-block code {
    display: block;
}
.diff-line {
    display: inline-block;
    line-height: 1.5;
    white-space: pre;
}
.diff-match {
    color: #7cd992;
}
.diff-miss {
    color: #ff6b6b;
}
</style>
