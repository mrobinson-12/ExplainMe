# ExplainMe — “I Don’t Get It” Translator (Rubrics + Criteria MVP)

## Goal
Build a small web app where a student pastes **rubric criteria** (or a rubric row / teacher criteria list) and gets a structured translation into:
- Plain English
- A small, generic example (format-only, not a full student answer)
- A checklist of what to fix / include
- “Teacher probably wants…” (grading intent)
- Clarifying questions + uncertainty notes (to avoid hallucinated certainty)

**Non-goals (for this phase)**
- Not solving problems or writing full essays.
- Not taking full assignment prompts or student drafts (rubrics/criteria only).
- No accounts, no classrooms, no file uploads.

---

## MVP UX
### Single page
**Inputs**
- Text area: “Paste rubric criteria / feedback criteria”
- Optional dropdown: Subject (Unknown / Writing / History / Science Lab / Math / Coding / Languages / Other)
- Optional dropdown: Grade band (Unknown / Middle / High / College)
- Button: Translate

**Outputs (fixed order)**
1. Plain English translation
2. Example (short, generic)
3. Checklist
4. Teacher probably wants…
5. Uncertainty + what to ask

**UX constraints**
- Keep it fast: one screen, no extra flows.
- Always show the uncertainty section (even if short).

---

## Input Handling
### Accepted input shapes
- Single rubric row: criterion + descriptor
- Bulleted criteria list
- Multi-row rubric pasted from LMS (messy formatting)

### Parsing strategy (MVP)
- Heuristic parse to identify rows/criteria:
  - Split by blank lines and common separators (`:` `–` `-` `•` tabs)
  - Detect rubric levels (e.g., “Exceeds / Meets / Approaches”)
- If parsing is uncertain, keep raw chunks and translate chunk-by-chunk.

### Data model (internal)
- `rawText: string`
- `subject: "unknown" | ...`
- `gradeBand: "unknown" | ...`
- `chunks: Array<{ label?: string; text: string }>`

---

## LLM Output Contract (important for reliability)
Require **structured JSON** from the model so the UI is stable.

Example schema:
```json
{
  "subject": "writing",
  "overallSummary": "...",
  "items": [
    {
      "original": "Use stronger evidence to support your claim",
      "plainEnglish": "...",
      "example": "...",
      "checklist": ["..."],
      "teacherWants": ["..."],
      "questions": ["..."],
      "assumptions": ["..."],
      "confidence": {
        "plainEnglish": "medium",
        "example": "low",
        "checklist": "medium",
        "teacherWants": "low"
      }
    }
  ]
}
```

UI renders each `item` as a card with those sections.

---

## Prompt + Guardrails (the “flavourtime”)
### Core behavior
- Translate **only what is present** in the rubric text.
- Prefer “likely interpretations” over “definitive meaning.”
- If criteria is vague (“be clearer”), output:
  - 2–3 plausible interpretations
  - questions to disambiguate
  - a checklist that is safe across interpretations

### Anti-hallucination rules
- Never invent specific rubric requirements (page count, citations, formatting) unless explicitly in the input.
- If the rubric seems incomplete, say so and ask what’s missing.
- Use calibrated language: “probably,” “often,” “might,” and explicit assumptions.

### Anti-cheating constraints
- Examples must be **generic and minimal**:
  - Demonstrate *form* (e.g., one sentence with a claim + one piece of evidence)
  - Avoid matching the student’s topic (since we’re not collecting their prompt)

---

## Subject Scaffolds (lightweight)
Use subject-specific checklists/examples without overfitting.

- Writing/ELA: claim-evidence-explanation, clarity, organization, audience, mechanics.
- History/Social Studies: thesis, sourcing, context, evidence integration, counterargument.
- Science Lab: variables, method clarity, data tables, units, error analysis, conclusion tied to data.
- Math: show work, define variables, units, justification, check reasonableness.
- Coding: correctness, readability, edge cases, tests, complexity, style.

If subject = Unknown: default to general academic writing clarity + evidence + completeness.

---

## Tech Stack (pragmatic MVP)
- Frontend: Next.js (React) + basic styling (keep minimal)
- Backend: Next.js route handler that calls the LLM
- Config: env var for API key

**Why this**: simple deploy, one repo, minimal plumbing.

---

## Implementation Steps
1. Scaffold Next.js app with a single page and basic layout.
2. Build form inputs + output renderer (cards per parsed chunk).
3. Implement parser for rubric chunks (best-effort).
4. Add API route:
   - validate input length
   - call LLM with strict JSON schema instruction
   - parse/validate JSON before returning
5. Add safety/quality features:
   - character limit + friendly errors
   - “uncertainty/questions” always required in output
6. Add lightweight evaluation harness:
   - a small set of pasted rubric examples in a JSON file
   - manual review checklist (hallucinations? too confident? useful checklist?)

---

## Acceptance Criteria (MVP)
- Given a pasted rubric row, the app returns all 5 output sections.
- Output is consistently structured (no broken formatting).
- The app explicitly flags uncertainty when criteria is vague.
- The app does not fabricate missing requirements.

---

## Future Enhancements (later)
- Allow “paste full prompt” mode.
- Allow “paste my draft” mode with feedback-only (no full rewrite).
- Save/share results, history, classroom mode.
- Add more rubric-format parsers (Canvas/Google Classroom exports).
