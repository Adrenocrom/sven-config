---
name: llm_repetition_loops
description: Why LLMs get stuck repeating themselves (neural text degeneration, autoregressive feedback, sampling/attention causes) and how to mitigate with nucleus sampling, penalties, and agent loop caps
tags:
- llm
- repetition
- decoding
- sampling
- nucleus-sampling
- text-generation
- agents
created_at: '2026-10-01T13:35:04.824886404+02:00'
---

# Why LLMs get stuck in repetition loops

## Root cause
Autoregressive generation: each emitted token is fed back into the context, so the model conditions on its own output. Repetition is **self-reinforcing** — once a pattern appears, continuing it becomes the locally most probable action. Repetition is a stable attractor of the decoding loop, which is why it appears suddenly after long healthy output.

## Canonical reference
Holtzman et al. 2019, *"The Curious Case of Neural Text Degeneration"*:
- Greedy/beam search (maximizing likelihood) reliably degenerates into bland loops.
- Next-token distribution is heavy-tailed; the argmax is often degenerate.
- **Nucleus (top-p) sampling** samples from the smallest token set with cumulative prob > p, avoiding both degenerate argmax and incoherent tail.

## Triggers
- **Decoding**: temperature too low (greedy) or too high; missing/mis-tuned repetition penalty.
- **Attention/context**: attention sinks & recency bias (repeated tail dominates); long contexts dilute attention; RoPE extrapolation beyond trained length; fp16/bf16 logit ties.
- **Training data**: repetitive boilerplate/code/HTML teaches repetition as valid continuation. Code especially prone.
- **Reasoning models**: "doom loops" — re-deriving the same step or oscillating between options in chain-of-thought.
- **Agents**: separate control-loop failure — same tool called with same args repeatedly because belief state doesn't change. Fix with loop detection + iteration caps, not temperature.

## Mitigations
| Layer | Fix |
|---|---|
| Sampling | top-p ~0.9–0.95, top-k, temp ~0.7; avoid pure greedy for open-ended text |
| Penalties | repetition penalty, presence/frequency penalty (tune carefully) |
| Decoding | no-repeat n-gram blocking, min-length constraints |
| Prompting | explicit stopping condition / output format; decompose tasks |
| Agents | max-iteration caps, detect repeated tool calls, inject "already tried X" |
| Serving | correct RoPE scaling, avoid precision-induced ties |

## One-liner
The model is a next-token predictor feeding on its own output; repetition is a stable attractor of that loop, and sampling strategies exist to keep it out.
