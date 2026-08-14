---
name: tutor
description: >
  Interactive quiz tutor for Obsidian StudyVault learning. Use when the user wants to:
  (1) Take a diagnostic assessment of their knowledge,
  (2) Study or review specific sections/topics,
  (3) Drill weak areas identified in previous sessions,
  (4) Check their learning progress or dashboard,
  or says things like "quiz me", "test me", "let's study", "/tutor", "학습", "퀴즈", "평가".
---

# Tutor Skill

Quiz-based tutor that tracks what the user knows and doesn't know at the **concept level**. The goal is helping users discover their blind spots through questions.

## File Structure

```
StudyVault/
├── *dashboard*              ← Compact overview: proficiency table + stats
├── *question*log*           ← User-raised questions + answers ONLY (standalone review page)
└── concepts/
    ├── {area-name}.md       ← Per-area concept tracking (attempts, status, error notes)
    └── ...
```

- **Dashboard**: Only aggregated numbers. Links to concept files. Stays small forever.
- **Concept files**: One per area. Tracks each concept with attempts, correct count, date, status, and error notes. Grows proportionally to unique concepts tested (bounded).
- **Question log**: Every question the *user* asked, with its answer. Newest first. The user reads this page alone to revisit their own doubts without wading through concept files. Sits next to the dashboard.

## Workflow

### Phase 0: Detect Language

Detect user's language from their message → `{LANG}`. All output and file content in `{LANG}`.

### Phase 1: Discover Vault

1. Glob `**/StudyVault/` in project
2. List section directories
3. Glob `**/StudyVault/*dashboard*` to find dashboard
4. If found, read it. Preserve existing file path regardless of language.
5. If not found, create from template (see Dashboard Template below)

If no StudyVault exists, inform user and stop.

### Phase 2: Ask Session Type

**MANDATORY**: Use AskUserQuestion to let the user choose what to do. Analyze the dashboard to build context-aware options, then present them.

Read the dashboard proficiency table and build options based on current state:

1. If unmeasured areas (⬜) exist → include "Diagnostic" option targeting those areas
2. If weak areas (🟥/🟨) exist → include "Drill weak areas" option naming the weakest area(s)
3. Always include "Choose a section" option so the user can pick any area
4. If all areas are 🟩/🟦 → include "Hard-mode review" option

Present these as an AskUserQuestion with header "Session" and concise descriptions showing which areas each option targets. The user MUST select before proceeding.

### Phase 3: Build Questions

1. Read markdown files in target section(s)
2. If drilling weak area: also read `concepts/{area}.md` to find 🔴 unresolved concepts — rephrase these in new contexts (don't repeat the same question)
3. Craft exactly 4 questions following `references/quiz-rules.md`

**CRITICAL**: Read `references/quiz-rules.md` before crafting ANY question. Zero hints allowed.

### Phase 4: Present Quiz

Use AskUserQuestion:
- 4 questions, 4 options each, single-select
- Header: "Q1. Topic" (max 12 chars)
- Descriptions: neutral, no hints

### Phase 5: Grade & Explain

1. Show results table (question / correct answer / user answer / result)
2. Wrong answers: concise explanation
3. Map each question to its area

### Phase 6: Update Files

#### 1. Update concept file (`concepts/{area}.md`)

For each question answered:
- **New concept**: Add row to table + if wrong, add error note under `### 오답 메모` (or localized equivalent)
- **Existing 🔴 concept answered correctly**: Increment attempts & correct, change status to 🟢, keep error note (learning history)
- **Existing 🟢 concept answered wrong again**: Increment attempts, change status back to 🔴, update error note

Table format:
```markdown
| Concept | Attempts | Correct | Last Tested | Status |
|---------|----------|---------|-------------|--------|
| concept name | 2 | 1 | 2026-02-24 | 🔴 |
```

Error notes format (only for wrong answers):
```markdown
### Error Notes

**concept name**
- Confusion: what the user mixed up
- Key point: the correct understanding
```

#### 2. Update dashboard

- Recalculate per-area stats from concept files (sum attempts/correct across all concepts in that area)
- Update proficiency badges: 🟥 0-39% · 🟨 40-69% · 🟩 70-89% · 🟦 90-100% · ⬜ no data
- Update stats: total questions, cumulative rate, unresolved/resolved counts, weakest/strongest

Dashboard stays compact — no session logs, no per-question details.

#### 3. Update question log (`*question*log*`)

**Trigger**: the user asked a question of their own — anything beyond answering the quiz. This includes:
- An explicit question ("why is X?", "do I have to do Y?", "what is Z?")
- "I don't know what ___ is" / "I'm not sure about ___" inside a quiz answer
- Pushback on an explanation, or a request to dig deeper

**Do NOT log**: meta requests about session format (question count, difficulty, file layout). Those are preferences, not learning questions.

Steps:
1. Glob `**/StudyVault/*question*log*`. If missing, create from the template below.
2. Prepend a new entry — **newest first**, numbered ascending (`Q7` goes above `Q6`).
3. Add the row to the table of contents at the top, also newest first.
4. If a new question refines an existing entry, add a cross-link (`후속 질문 → [[#Q6]]`) rather than duplicating.

Each entry must stay scannable. This page exists because the user finds full documents tiring — keep it tight:
- Restate the question in the user's own words as the heading
- **One-line answer in bold immediately after** — the user should be able to stop reading there
- Then the supporting detail: tables, short code, a diagram if it earns its place
- Flag any wrong reasoning the user used, with a concrete counter-example — this is the highest-value part
- End with `📎 [[concepts/{area}]]` and cross-links

Answer quality bar: cite concrete evidence (a file path and line, a real class name, an exact rule) rather than restating the principle. If the vault has example source code, check it — an empirical finding beats a paraphrase.

## Dashboard Template

Create when no dashboard exists. Filename localized to `{LANG}`. Example in English:

```markdown
# Learning Dashboard

> Concept-based metacognition tracking. See linked files for details.

---

## Proficiency by Area

| Area | Correct | Wrong | Rate | Level | Details |
|------|---------|-------|------|-------|---------|
(one row per section, last column = [[concepts/{area}]] link)
| **Total** | **0** | **0** | **-** | ⬜ Unmeasured | |

> 🟥 Weak (0-39%) · 🟨 Fair (40-69%) · 🟩 Good (70-89%) · 🟦 Mastered (90-100%) · ⬜ Unmeasured

---

## Stats

- **Total Questions**: 0
- **Cumulative Rate**: -
- **Unresolved Concepts**: 0
- **Resolved Concepts**: 0
- **Weakest Area**: -
- **Strongest Area**: -
```

## Concept File Template

Create per area when first question is asked. Example:

```markdown
# {Area Name} — Concept Tracker

| Concept | Attempts | Correct | Last Tested | Status |
|---------|----------|---------|-------------|--------|

### Error Notes

(added as concepts are missed)
```

## Question Log Template

Create when the user asks their first question. Filename localized to `{LANG}`, placed beside the dashboard.

```markdown
# My Question Log

> Only the questions I raised during study, with their answers.
> Read this page alone — no need to dig through the concept files.
> New questions are appended below.

---

## Index

| # | Question | Area | Date |
|---|----------|------|------|
| [Q1](#q1) | ... | ... | YYYY-MM-DD |

---

<a id="q1"></a>
## Q1. {question in the user's own words}

> **One-line answer: ...**

{supporting detail — tables, short code, counter-examples}

### ⚠️ {wrong reasoning the user used, if any}

{concrete counter-example}

📎 [[concepts/{area}]]

---

## Related

- [[{dashboard}]]
```

## Important Reminders

- ALWAYS read `references/quiz-rules.md` before creating questions
- NEVER include hints in option labels or descriptions
- NEVER use "(Recommended)" on any option
- Randomize correct answer position
- After grading, ALWAYS update the concept file AND dashboard
- **Whenever the user asks a question of their own, ALWAYS append it to the question log** — this applies mid-session, not only at grading time. A question answered but not logged is a bug.
- Communicate in user's language
