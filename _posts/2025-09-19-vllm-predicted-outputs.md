---
layout: post
title: "VLLM Predicted Outputs"
date: 2025-09-19
hero_image: false
---

Have you ever asked an AI agent to make an simple change to a large piece of code, only to find yourself sitting idly by
while the llm regurgitates pages and pages of code you've already wrote, with just a few small changes made?  Did you 
wonder 'WHY does it have to regenerate all of this code token by token? Can't it just regenerate the pieces that have
changed?'

The answer is YES, with Predicted Outputs. Predicted outputs is a technique in LLM generation which uses a prediction
about what the LLM will output to allow the llm to functionally skip generation of the parts that it already 'knows 
about', and only generate the truly new tokens.  The prediction only needs to match partially: if a little bit matches
then the generation will speed up a little bit, if a lot matches then the speedup will be dramatic.

Predicted outputs is not a common feature of most llm platforms - one of the only implementations of it is the original
in OpenAI's api, but as we will see below it is certainly not optimal, often causing the generation to be SLOWER rather
than speeding it up. Perhaps this is the reason that it hasn't seen more uptake, but we believe that this technique has
the ability to dramatically speed up many llm applications, not the least of which is coding agents.

Cascade Technologies is proud to release a new implementation of predicted outputs in the excellent VLLM platform. Our
implementation scales nearly linearly with prediction accuracy, meaning that a 50% accurate prediction will result in
generations in roughly half the time, and 100% accurate predictions will be nearly instantaneous.  Truly, the llm only
needs to generate the new content in an output.

## How does it work?

Llm prompts are generally processed in two phases:
- First the input tokens are processed <strong>in parallel</strong> very quickly, thousands of tokens may all be
processed in hundreds of milliseconds.  They can be processed so quickly because they can be processed in parallel.  
They can be processed in parallel because, unlike output tokens, each token already knows all the tokens that precede it.
- Next the output tokens are processed one at a time often at 10 or more milliseconds per token, 100x or more times
slower than their input tokens, even though the actual math and weights that process them are identical to input.

<strong>tldr:</strong> Predicted Outputs uses a prediction about the contents of the model's output to aid in the
generation of outputs tokens. When predictions match, output tokens can be processed in parallel, effectively as if 
they were input tokens.  This speedup is achieved with <strong>NO reduction in accuracy</strong>, the output of the 
llm is identical.

* The user provides a prediction for the output of their llm request.  This can be many things, but an
easy example to understand is the case of code modification, in which case you would provide the
original code as the prediction.
* As long as the prediction is aligned with the output, we are able to process our llm generation
in parallel, which can make it orders of magnitude faster.
* When the prediction diverges from the output, we use standard diff algorithms to realign the prediction and continue
processing the output in parallel.
* Predicted Outputs is already supported in the standard OpenAI API so you can drop it in and use it with very few changes 
to your client code.

## Technical Details

Cascade Technology's implementation works as a form of speculative decoding, but instead of using a draft model to 
generate predictions, we use the static text 'prediction' sent via the api as a basis for our speculative proposals.
When the prediction and the output diverge, we use standard diff algorithms to realign the prediction. 
- The prediction is entirely on the cpu, with no additional GPU resources required.
- The alignment algorithm's execution can be hidden with a one frame delay on alignment, eliminating any latency overhead.
- Only exact prediction matches are accepted, so there is no loss in accuracy.

## Example

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

Notice the colon on the first line of the second block!  As you can see, Predicted Outputs needs one identical line to realign, 
but then carries on with the prediction for the rest of the generation.  The tokens in green are generated nearly "for free".

Because running LLM model passes is so slow, and computing text diffs are so fast, matching just a single line can often
provide speed benefits over the no-prediction base case, and there are many use cases where you can frequently match
much more:

* Coding agents modifying sections of code. This is the easiest win for Predicted Outputs, given the ease of
  prediction and frequency of prediction matching.
* Implementing true Structured Outputs can often be very complicated, but passing an example of your output as a
  prediction can often get many of the speed benefits with much less difficulty.
* Modifying documents.
* Updating state agent state dicts / memory.

## Comparison with OpenAI

<div id="cascade-comparison-root" style="width: 100%; min-height: 720px;"></div>
<script type="module">
  import React, { useMemo } from "https://esm.sh/react@18.3.1";
  import { createRoot } from "https://esm.sh/react-dom@18.3.1/client";
  import {
    ResponsiveContainer,
    BarChart,
    Bar,
    XAxis,
    YAxis,
    CartesianGrid,
    Tooltip,
    Cell
  } from "https://esm.sh/recharts@2.12.7";

  const CascadeComparison = () => {
    const verbatimData = useMemo(
      () => [
        { name: "Baseline", speedup: 100, tokens: 56.9, color: "#94a3b8" },
        { name: "OpenAI", speedup: (386.434 / 56.9) * 100, tokens: 386.434, color: "#3b82f6" },
        { name: "Cascade", speedup: (1488.944 / 129.843) * 100, tokens: 1488.944, color: "#22c55e" }
      ],
      []
    );

    const multiplayerData = useMemo(
      () => [
        { name: "Baseline", speedup: 100, tokens: 71.674, color: "#94a3b8" },
        { name: "OpenAI", speedup: (57.774 / 71.674) * 100, tokens: 57.774, color: "#3b82f6" },
        { name: "Cascade", speedup: (178.454 / 121.17) * 100, tokens: 178.454, color: "#22c55e" }
      ],
      []
    );

    const CustomTooltip = ({ active, payload }) => {
      if (active && payload && payload.length) {
        const data = payload[0].payload;
        return (
          React.createElement(
            "div",
            { className: "cascade-tooltip" },
            React.createElement("p", { className: "title" }, data.name),
            React.createElement("p", null, `${data.speedup.toFixed(1)}% of baseline speed`),
            React.createElement("p", { className: "muted" }, `${data.tokens.toFixed(1)} tokens/sec`)
          )
        );
      }
      return null;
    };

    return React.createElement(
      "div",
      { className: "cascade-chart" },
      React.createElement(
        "div",
        { className: "cascade-chart__intro" },
        React.createElement(
          "h2",
          null,
          "Cascade vs OpenAI: Predicted Outputs Performance"
        ),
        React.createElement(
          "p",
          null,
          "Testing predicted outputs on a ~800 token Python snake game. ",
          React.createElement("span", { className: "strong" }, "Verbatim"),
          " repeats the code with perfect prediction (93-97% acceptance). ",
          React.createElement("span", { className: "strong" }, "Multiplayer"),
          " uses the original code as prediction with the modification prompt \"make it multiplayer\" (26-40% acceptance)."
        )
      ),
      React.createElement(
        "div",
        { className: "cascade-chart__stack" },
        React.createElement(
          "div",
          { className: "panel" },
          React.createElement("h3", null, "Verbatim Repeat"),
          React.createElement("p", null, "Perfect prediction scenario"),
          React.createElement(
            "div",
            { className: "chart-wrapper" },
            React.createElement(
              ResponsiveContainer,
              { width: "100%", height: 320 },
              React.createElement(
                BarChart,
                {
                  data: verbatimData,
                  margin: { top: 20, right: 30, left: 20, bottom: 24 }
                },
                React.createElement(CartesianGrid, { strokeDasharray: "3 3", stroke: "#374151" }),
                React.createElement(XAxis, { dataKey: "name", tick: { fill: "#9ca3af" } }),
                React.createElement(
                  YAxis,
                  {
                    label: {
                      value: "% of Baseline Speed",
                      angle: -90,
                      position: "insideLeft",
                      style: { fill: "#9ca3af" }
                    },
                    tick: { fill: "#9ca3af" }
                  }
                ),
                React.createElement(Tooltip, { content: React.createElement(CustomTooltip, null) }),
                React.createElement(Bar, { dataKey: "speedup", radius: [8, 8, 0, 0] }, verbatimData.map((entry, index) => React.createElement(Cell, { key: `verbatim-${index}`, fill: entry.color })))
              )
            )
          ),
          React.createElement(
            "ul",
            { className: "legend" },
            React.createElement(
              "li",
              null,
              React.createElement("span", { className: "legend-label" }, "OpenAI speedup:"),
              React.createElement(
                "strong",
                { className: "legend-value positive" },
                `${((386.434 / 56.9 - 1) * 100).toFixed(0)}% faster`
              )
            ),
            React.createElement(
              "li",
              null,
              React.createElement("span", { className: "legend-label" }, "Cascade speedup:"),
              React.createElement(
                "strong",
                { className: "legend-value accent" },
                `${((1488.944 / 129.843 - 1) * 100).toFixed(0)}% faster`
              )
            ),
            React.createElement(
              "li",
              { className: "muted acceptance" },
              React.createElement("span", { className: "legend-label" }, "Acceptance rate:"),
              React.createElement(
                "span",
                { className: "legend-value acceptance-values" },
                React.createElement("span", { className: "acceptance-openai" }, "OpenAI 93.15%"),
                React.createElement("span", { className: "acceptance-divider" }, "|"),
                React.createElement("span", { className: "acceptance-cascade" }, "96.67%")
              )
            )
          )
        ),
        React.createElement(
          "div",
          { className: "panel" },
          React.createElement("h3", null, "Code Modification"),
          React.createElement("p", null, "Partial prediction scenario"),
          React.createElement(
            "div",
            { className: "chart-wrapper" },
            React.createElement(
              ResponsiveContainer,
              { width: "100%", height: 320 },
              React.createElement(
                BarChart,
                {
                  data: multiplayerData,
                  margin: { top: 20, right: 30, left: 20, bottom: 24 }
                },
                React.createElement(CartesianGrid, { strokeDasharray: "3 3", stroke: "#4a5568" }),
                React.createElement(XAxis, { dataKey: "name", tick: { fill: "#475569" } }),
                React.createElement(
                  YAxis,
                  {
                    label: {
                      value: "% of Baseline Speed",
                      angle: -90,
                      position: "insideLeft",
                      style: { fill: "#475569" }
                    },
                    tick: { fill: "#475569" }
                  }
                ),
                React.createElement(Tooltip, { content: React.createElement(CustomTooltip, null) }),
                React.createElement(Bar, { dataKey: "speedup", radius: [8, 8, 0, 0] }, multiplayerData.map((entry, index) => React.createElement(Cell, { key: `multiplayer-${index}`, fill: entry.color })))
              )
            )
          ),
          React.createElement(
            "ul",
            { className: "legend" },
            React.createElement(
              "li",
              null,
              React.createElement("span", { className: "legend-label" }, "OpenAI speedup:"),
              React.createElement(
                "strong",
                { className: "legend-value negative" },
                `${((57.774 / 71.674 - 1) * 100).toFixed(0)}% slower`
              )
            ),
            React.createElement(
              "li",
              null,
              React.createElement("span", { className: "legend-label" }, "Cascade speedup:"),
              React.createElement(
                "strong",
                { className: "legend-value accent" },
                `${((178.454 / 121.17 - 1) * 100).toFixed(0)}% faster`
              )
            ),
            React.createElement(
              "li",
              { className: "muted acceptance" },
              React.createElement("span", { className: "legend-label" }, "Acceptance rate:"),
              React.createElement(
                "span",
                { className: "legend-value acceptance-values" },
                React.createElement("span", { className: "acceptance-openai" }, "OpenAI 25.64%"),
                React.createElement("span", { className: "acceptance-divider" }, "|"),
                React.createElement("span", { className: "acceptance-cascade" }, "Cascade 40.03%")
              )
            )
          )
        )
      )
    );
  };

  const mount = () => {
    const container = document.getElementById("cascade-comparison-root");
    if (!container) return;

    const root = createRoot(container);
    root.render(React.createElement(CascadeComparison));
  };

  if (document.readyState === "loading") {
    document.addEventListener("DOMContentLoaded", mount);
  } else {
    mount();
  }
</script>

<style>
  #cascade-comparison-root {
    position: relative;
    background: #000;
    color: #e5e7eb;
    border-radius: 20px;
    padding: clamp(1.5rem, 4vw, 3rem);
    box-shadow: inset 0 0 0 1px rgba(148, 163, 184, 0.15);
  }

  #cascade-comparison-root .cascade-chart {
    display: flex;
    flex-direction: column;
    gap: clamp(1.5rem, 3vw, 2.5rem);
  }

  #cascade-comparison-root .cascade-chart__intro h2 {
    font-size: clamp(1.65rem, 3vw, 2.5rem);
    margin-bottom: 0.5rem;
  }

  #cascade-comparison-root .cascade-chart__intro p {
    color: rgba(226, 232, 240, 0.7);
    max-width: 60ch;
    line-height: 1.6;
  }

  #cascade-comparison-root .cascade-chart__intro .strong {
    color: #f8fafc;
    font-weight: 600;
  }

  #cascade-comparison-root .cascade-chart__stack {
    display: flex;
    flex-direction: column;
    gap: clamp(1.5rem, 3vw, 2.5rem);
  }

  #cascade-comparison-root .panel {
    background: rgba(15, 23, 42, 0.75);
    border-radius: 16px;
    padding: clamp(1.15rem, 2.3vw, 1.5rem) clamp(1.25rem, 2.5vw, 1.75rem) clamp(0.6rem, 1.5vw, 0.9rem);
    border: 1px solid rgba(148, 163, 184, 0.1);
    backdrop-filter: blur(8px);
  }

  #cascade-comparison-root .panel h3 {
    font-size: clamp(1.1rem, 2.4vw, 1.5rem);
    margin-bottom: 0.25rem;
    color: #e2e8f0;
  }

  #cascade-comparison-root .panel p {
    color: rgba(203, 213, 225, 0.7);
    margin-bottom: 1rem;
    font-size: 0.9rem;
  }

  #cascade-comparison-root .panel .chart-wrapper {
    width: 100%;
    height: 320px;
  }

  #cascade-comparison-root .panel .legend {
    margin: 0.4rem auto 0;
    padding: 0;
    list-style: none;
    display: flex;
    flex-direction: column;
    gap: 0.2rem;
    color: rgba(203, 213, 225, 0.8);
    font-size: 0.9rem;
    line-height: 1.15;
    align-items: center;
    width: min(100%, 22rem);
  }

  #cascade-comparison-root .panel .legend li {
    display: grid;
    grid-template-columns: clamp(6rem, 38%, 9.5rem) minmax(0, 1fr);
    align-items: baseline;
    column-gap: 0.6rem;
    margin: 0;
    padding: 0;
    position: static;
    width: 100%;
  }

  #cascade-comparison-root .panel .legend li + li {
    margin-top: 0.15rem;
  }

  #cascade-comparison-root .panel .legend .legend-label {
    color: rgba(148, 163, 184, 0.9);
  }

  #cascade-comparison-root .panel .legend .legend-value {
    justify-self: stretch;
    text-align: left;
    display: flex;
    align-items: baseline;
    gap: 0.35rem;
    color: #e2e8f0;
    font-weight: 600;
    flex-wrap: nowrap;
  }

  #cascade-comparison-root .panel .legend .legend-value.acceptance-values {
    gap: 0.4rem;
    white-space: nowrap;
  }

  #cascade-comparison-root .panel .legend strong {
    font-weight: 600;
  }

  #cascade-comparison-root .panel .legend .positive {
    color: #3b82f6;
  }

  #cascade-comparison-root .panel .legend .negative {
    color: #f97316;
  }

  #cascade-comparison-root .panel .legend .accent {
    color: #22c55e;
  }

  #cascade-comparison-root .panel .legend .muted {
    color: rgba(148, 163, 184, 0.65);
    font-size: 0.85rem;
  }

  #cascade-comparison-root .panel .legend .acceptance-openai {
    color: #f87171;
    font-weight: 600;
    margin-left: 0;
  }

  #cascade-comparison-root .panel .legend .acceptance-cascade {
    color: #22c55e;
    font-weight: 600;
    margin-left: 0;
  }

  #cascade-comparison-root .panel .legend .acceptance-divider {
    margin: 0;
    color: rgba(148, 163, 184, 0.5);
    font-weight: 400;
  }

  #cascade-comparison-root .panel .legend .acceptance {
    align-items: center;
  }

  #cascade-comparison-root .panel .legend li::before {
    display: none;
  }

  #cascade-comparison-root .cascade-tooltip {
    background: rgba(15, 23, 42, 0.95);
    border: 1px solid rgba(129, 140, 248, 0.35);
    padding: 0.75rem 1rem;
    border-radius: 12px;
    box-shadow: 0 8px 24px rgba(15, 23, 42, 0.45);
  }

  #cascade-comparison-root .cascade-tooltip .title {
    font-weight: 600;
    margin: 0 0 0.25rem;
    color: #f8fafc;
  }

  #cascade-comparison-root .cascade-tooltip .muted {
    color: rgba(203, 213, 225, 0.7);
    margin: 0;
    font-size: 0.85rem;
  }

  @media (max-width: 640px) {
    #cascade-comparison-root {
      padding: 1.25rem;
    }

    #cascade-comparison-root .panel .chart-wrapper {
      height: 260px;
    }
  }
</style>

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
