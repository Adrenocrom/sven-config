---
name: research_methodology
description: Best practices and structured approach for conducting technical research,
  including web searches, documentation review, codebase exploration, and synthesizing
  findings.
tags:
- research
- methodology
- best-practices
- web-search
created_at: '2026-07-12T18:56:15.125707+00:00'
---

# Technical Research Methodology

## Structured Approach

### 1. Define the Research Question
- Be specific: "How does X work?" → "What is the authentication flow in library X for OAuth2 PKCE?"
- Identify what you already know vs. what you need to find out
- Determine the scope and depth needed (quick lookup vs. deep dive)

### 2. Formulate Search Strategy
- **Web search**: Use DuckDuckGo with targeted queries. Prefer site-specific searches (`site:docs.python.org`) for documentation.
- **Codebase exploration**: Use `find` to locate relevant files, `grep` to find patterns/usage, and read surrounding context.
- **Man pages / docs**: Use `manpage` for CLI tools; fetch official docs via `webfetch`.

### 3. Execute Searches (Iterative)
- Start broad, then narrow based on initial findings.
- Try multiple query variations — the first result is rarely the best.
- Cross-reference at least 2 sources before trusting a claim.

### 4. Evaluate Sources
| Priority | Source Type | Reliability |
|----------|------------|-------------|
| 1st | Official documentation / source code | Highest |
| 2nd | Reputable blogs, RFCs, spec docs | High |
| 3rd | Stack Overflow, forum posts | Medium — verify against primary sources |
| 4th | AI-generated summaries | Low — always verify |

### 5. Synthesize Findings
- Extract the key facts, not the narrative.
- Note version-specific behavior (APIs change).
- Flag any contradictions between sources and investigate further.

## Search Tips
- Use quotes for exact phrases: `"async context manager"`
- Combine operators: `site:github.com "dependency injection" python`
- For APIs: search `<language> <library> <method>` directly
- When stuck, try searching the error message itself — often leads to root cause

## Output Format
When reporting research findings, structure as:
1. **Answer**: Direct response to the question (1-2 sentences)
2. **Details**: Key supporting information, code examples if relevant
3. **Sources**: Links or file paths consulted
4. **Caveats**: Version constraints, edge cases, uncertainties

