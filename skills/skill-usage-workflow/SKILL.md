---
name: skill_usage_workflow
description: Best practices for proactively discovering and applying stored skills
  during task execution. Ensures skills are used as reference material before answering
  domain-relevant questions.
tags:
- workflow
- skills
- best-practices
created_at: '2026-07-12T19:43:52.676538+00:00'
---

## Skill Usage Workflow

### When to Check Skills
1. **At task start**: List available skills to understand what's at your disposal
2. **When topic matches**: Search skills by keyword when the user's request touches a known domain (weather, research, coding patterns, etc.)
3. **Before complex execution**: If a task could benefit from a stored methodology or pattern, search and apply it

### How to Apply Skills
- Load relevant skill via `get_skill`
- Use its content as reference/inspiration — not rigid rules
- Adapt the skill's patterns to the specific context
- Mention when you're using a skill if it meaningfully changes your approach

### Trigger Examples
| User Input | Search Query | Relevant Skill |
|------------|-------------|----------------|
| "weather in Berlin" | weather | weather_api_wttr |
| "research this topic" | research | research_methodology |
| "how should I structure..." | methodology, pattern | (search broadly) |

### Anti-Patterns to Avoid
- Don't force a skill if it doesn't fit the context
- Don't skip skills just because you could answer without them — they encode learned patterns worth applying
