---
name: ollama_decision_models
description: 'Ollama decision models (Nimble 9B from Bespoke Labs, Tev1 4B/0.8B from Together AI): fast typed classification via the /v1/systemone endpoint with choice/noul/score question types, probabilities, and the TypeSafe Python SDK.'
tags:
- ollama
- decision-models
- nimble
- tev1
- systemone
- classification
- llm
created_at: '2026-10-12T07:37:31.286225502+02:00'
---

# Ollama Decision Models: Nimble & Tev1 (`/v1/systemone`)

Decision models are a separate model class on Ollama (requires **Ollama 0.35+**). They are NOT chat models: you send text plus a set of questions, and they return a typed answer with probabilities for every question. No reasoning step — the model reads the prompt once per question and scores answer tokens directly (Nimble: <100ms on an M5 Max).

- **Nimble** (`nimble`, `nimble:9b`) — 9B from Bespoke Labs, fine-tuned from Qwen3.5-9B. 9.3–9.5GB. Apache 2.0. ~75.7% mean accuracy on 13 public datasets. Trained on contrastive pairs.
- **Tev1** (`tev1` / `tev1:4b`, `tev1:0.8b`) — experimental, from Together AI, fine-tuned from Qwen3.5. 4B = 4.4–4.5GB (73.3%), 0.8B = ~812MB (63.5%). MIT-licensed dataset builders/training scripts.

Model choice: Nimble 9B = most accurate local option; Tev1 4B = smaller; Tev1 0.8B = tight memory budgets. (Reference: Jev 1.13 via TypeSafe API = 76.0%.)

## API: `POST /v1/systemone`

Follows TypeSafe's Jev API. Ollama builds the prompt for you.

```bash
curl http://localhost:11434/v1/systemone \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nimble",
    "state": {"ticket": "I was charged twice. Please refund the extra payment."},
    "questions": {
      "team": {
        "type": "choice",
        "instructions": "Which team should handle this ticket?",
        "criteria": {
          "billing": "Payments and refunds",
          "technical": "Bugs and integrations",
          "other": "None of the above"
        }
      },
      "refund": {
        "type": "noul",
        "instructions": "Does the customer explicitly ask for a refund?"
      },
      "urgency": {
        "type": "score",
        "instructions": "How urgent is this ticket?",
        "criteria": ["Routine", "Soon", "Urgent"]
      }
    }
  }'
```

### Request fields

| Field | Description |
|---|---|
| `model` | `nimble` or `tev1` (4B) / `tev1:0.8b` |
| `state` | Text to judge: string, or JSON object/array for structured input |
| `questions` | 1–64 named questions; answers come back in the same order |
| `keep_alive` | Optional: how long the model stays loaded |

### Question types

| Type | `criteria` | Answer fields |
|---|---|---|
| `choice` | Object mapping option → description; `null` lets the option name describe itself | `choice`, `probabilities`, `confidence` |
| `noul` | Optional `{"true": "...", "false": "..."}` | `noul` = probability the answer is true |
| `score` | Array of level descriptions, lowest first (levels numbered from 0) | `score` (probability-weighted level), `legend`, `probabilities`, `confidence` |

- Choice/score take 2–26 options; **Tev1 was trained on 2–24, stay in that range**.
- `confidence` (0–1) = how concentrated the probabilities are. It is NOT the chance the answer is right.

### Response shape

```json
{
  "model": "nimble",
  "answers": {
    "team": {
      "type": "choice",
      "choice": "billing",
      "probabilities": {"billing": 0.984, "technical": 0.011, "other": 0.005},
      "confidence": 0.918
    },
    "refund": {"type": "noul", "noul": 0.997},
    "urgency": {
      "type": "score",
      "score": 0.693,
      "legend": {"0": "Routine", "1": "Soon", "2": "Urgent"},
      "probabilities": {"0": 0.482, "1": 0.342, "2": 0.175},
      "confidence": 0.068
    }
  },
  "usage": {"input_tokens": 841, "output_tokens": 3}
}
```

## Python: TypeSafe SDK

Decision models are NOT in the Ollama CLI or the Ollama Python/JS libraries yet. Use the raw API or TypeSafe's SDK (Ollama ignores the API key — any value works):

```bash
pip install typesafe-sdk
export TYPESAFE_BASE_URL=http://localhost:11434
export TYPESAFE_API_KEY=ollama
export TYPESAFE_DEFAULT_MODEL=nimble
```

```python
from typesafe_sdk import Choice, Noul, Score, TypeSafeClient

questions = {
    "team": Choice(instructions="Which team?", criteria={"billing": "Payments", "technical": "Bugs"}),
    "refund": Noul(instructions="Does the customer explicitly ask for a refund?"),
    "urgency": Score(instructions="How urgent?", criteria=["Routine", "Soon", "Urgent"]),
}

with TypeSafeClient(timeout=120) as client:
    result = client.system_one(state={"ticket": "..."}, questions=questions)

print(result.choices["team"].choice)
print(result.nouls["refund"].noul)
print(result.scores["urgency"].score)
```

## Use cases

Routing (pick destination/model), condition checks (yes/no + evidence), policy application (rules → typed decision), rubric rating (ordered levels). Good for agent guardrails, e.g. `noul` question "Could this tool call cause harm?" on a tool call before running it.

## Caveats & gotchas

- **Text only.** Models only pick from answers you supply — no explanations, nested JSON, or quotes.
- **Questions are scored independently** — enforce cross-question consistency in your own code.
- **Probabilities are not calibrated to your data.** A 0.9 probability ≠ 90% chance of being right. Test thresholds on your own data.
- **Add a "none"/"no match" option** if none of the answers might fit.
- **Context limits:** every question is scored with the full state + question set in its prompt. Nimble prompt must fit ~8,192 tokens; Tev1 runs ~2,000 tokens (longest training example ~1,500) — keep Tev1 inputs short.
- Request bodies up to 64 KiB.
- Tev1 in a regular chat tends to reply in prose — always use `/v1/systemone`.
- Don't let these be the only check on high-stakes decisions. Tev1 is not fully tested for prompt injection, non-English, or calibration.

## References

- https://ollama.com/library/nimble · https://ollama.com/library/tev1
- Ollama blog: "Ollama now supports decision models"
- GitHub: bespokelabsai/nimble, togethercomputer/tev1
- HF: bespokelabs/Bespoke-Nimble-9B, togethercomputer/Tev1-4B-experimental, Tev1-0.8B-experimental
- Together AI: "How to train your own Jev for $17"
