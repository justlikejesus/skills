---
name: quiz-content-lint
description: Lint an existing chapter quiz (KO + EN) against the 15 mechanical rules, per-type rules, and advisor rubric encoded in `seed-quiz-from-srt`. Read-only — produces a markdown report with two layers: a six-perspective scorecard (Item Mechanics, Fairness/Bloom, Transcript Grounding, Coverage & Rotation, Bilingual Parity, Pedagogical Value — each 0–10 with a publish gate at min ≥ 7) and a per-question violation list (rule id, severity, snippet, suggested fix). Does not modify the database. Use when the user says "lint quiz <course>/<chapter>", "audit this quiz", "score this quiz", "check this quiz against the rules", or wants to spot drift after admin-UI edits.
user-invocable: true
argument-hint: "<course-slug> <chapter-position-0indexed> [--srt <path>] [--rule <ID>[,ID,...]] [--locale ko|en] [--snapshot] [--severity error|warn|info]"
---

# Lint an existing chapter quiz against the seed-quiz ruleset

A read-only auditor that runs the 15 mechanical rules + per-type rules + advisor rubric defined in `.claude/skills/seed-quiz-from-srt/SKILL.md` against a quiz already in the Neon database. Produces a six-perspective scorecard plus a per-question violation list. **Does not write to the database, ever.**

## Why this skill exists

`seed-quiz-from-srt` enforces the rules at *author* time. After that, content drifts:

- An admin edits an MC question in the UI and adds a fourth option that's a synonym of the key.
- A fill_blank stem gets a `{{ blank }}` and a `{{1}}` mixed together — `buildQuizSnapshots` will silently freeze whichever form it detects first.
- A T/F statement picks up a "usually" qualifier in translation and is no longer categorical.
- `questions_per_attempt` gets bumped to a value larger than the pool, defeating `shuffleQuestions`.
- Re-publishing freezes drifted items into the snapshot students attempt.

Lint runs the same rule corpus in reverse: read the live data, score it, surface violations. Same rules, different direction. Catches drift before publish — and surfaces unsupported question types (`matching`, `ordering`) that the student app silently skips.

## What it produces

A markdown report printed to the chat with two layers:

1. **Scorecard** — six perspectives, each scored **0–10** by `advisor()`, with a one-sentence headline per perspective and a **publish gate** (PASS if every perspective ≥ 7, else FAIL). The lowest-scoring perspective is the leverage point: fix it first to move the gate.
2. **Per-question violations** — quiz-level findings (pool size, type mix, time limit) at the top, then one block per question with that question's violations: rule id, severity, locale, snippet, suggested fix.

**Does not auto-apply fixes** — the user picks how to act.

Severity levels for individual violations:

- **ERROR**: deterministic, unambiguous violation (e.g., MC has 5 options, fill_blank mixes `{{ blank }}` and `{{1}}`, type is `matching`/`ordering`). Must fix before re-publish.
- **WARN**: judgment-based violation flagged by the perspective scoring. Usually worth fixing; author may have a reason.
- **INFO**: heads-up — boundary case (e.g., a 4-option MC vs the rule's preferred 3) where the author may have intended the choice.

The scorecard and severity counts are independent: a perspective can score 8/10 even with WARN violations under it (the score is holistic), and a perspective with no ERRORs can still score < 7 if items read as drift.

## Protocol (do these in order)

### 1. Preflight

- Confirm `<course-slug>` exists in `classes`. If not, list valid slugs and exit.
- Confirm `class_chapters WHERE class_id = $id AND position = $position` resolves to a chapter row. If not, list the chapter `position + KO title` for that course and exit. (Position is 0-indexed — the first chapter is `position = 0`.)
- Confirm `quizzes WHERE chapter_id = $chapterId AND position = 0` resolves. If no quiz exists for this chapter, exit with "no quiz seeded yet — use `seed-quiz-from-srt` first."
- Optional flags:
  - `--srt <path>` — enables P3 (Transcript Grounding) full scoring; without it, P3 falls back to "data module receipts only" mode (see step 4 P3 below).
  - `--rule ID[,ID,...]` — restrict to specific rule IDs (M1–M15 mechanical, plus per-type rules — see Rule registry).
  - `--locale ko|en` — restrict to one locale; P5 reports `N/A — single-locale lint`.
  - `--snapshot` — lint `classes.published_snapshot.quizzes[<quizId>]` (what students see) instead of the live draft tables.
  - `--severity error|warn|info` — minimum severity in the per-question list (does not affect the scorecard).

### 2. Pull the live shape

Read every row the rules touch:

```sql
SELECT q.id, q.position, q.questions_per_attempt, q.passing_threshold,
       q.is_required, q.shuffle_questions, q.time_limit_minutes
  FROM quizzes q
  JOIN class_chapters cc ON cc.id = q.chapter_id
  JOIN classes c ON c.id = cc.class_id
 WHERE c.slug = $1 AND cc.position = $2 AND q.position = 0;

SELECT * FROM quiz_translations WHERE quiz_id = $quizId;

SELECT qq.id, qq.question_type, qq.points, qq.position
  FROM quiz_questions qq
 WHERE qq.quiz_id = $quizId
 ORDER BY qq.position;

SELECT * FROM quiz_question_translations
 WHERE question_id = ANY($questionIds);

SELECT qqi.id, qqi.question_id, qqi.role, qqi.position, qqi.is_mc_correct
  FROM quiz_question_items qqi
 WHERE qqi.question_id = ANY($questionIds)
 ORDER BY qqi.question_id, qqi.position;

SELECT * FROM quiz_question_item_translations
 WHERE item_id = ANY($itemIds);
```

If `--snapshot`, read `classes.published_snapshot->'quizzes'->><quizId>` and parse it as the snapshot shape from `packages/utils/src/course-snapshot.ts`. The snapshot has `fillBlankMode: "ordered" | "any_order"` per question — surface this in the report; it's authoritative for grading.

If `--srt <path>` is set, read the SRT and split into cues with `(startSeconds, endSeconds, text)`. Used by P3.

### 3. Run deterministic checks

Cheap, definitive. Severity defaults to ERROR unless noted. Encode each as a small predicate over the data pulled in step 2.

| ID | Rule | Check |
|---|---|---|
| M1a | Type is supported by student app | `question_type ∈ {multiple_choice, true_false, fill_blank}`. `matching` and `ordering` are silently skipped by `apps/school/src/features/quiz/actions.ts:230`; treat as ERROR — those items are invisible to students. |
| M1b | Type matches DB enum | `question_type ∈ {multiple_choice, true_false, fill_blank, matching, ordering}` per the `0001_initial.ts` CHECK constraint. (Anything else is impossible from a sane insert path; flag if seen.) |
| M2 | No negative stems | Stem text contains no standalone `NOT`, `EXCEPT`, `LEAST`, or `only` as the negation device (regex on word boundaries; allow within phrases like "the only correct…"). |
| M3 | No "all/none of the above" / "I don't know" | Item `primary_text` contains none of: `all of the above`, `none of the above`, `I don't know`, `위 보기 모두`, `없음`, `모름`. |
| M4 | No absolute determiners (unless source is absolute — judgment) | Item `primary_text` contains: `always`, `never`, `completely`, `absolutely`, `항상`, `절대`, `완전히` → INFO (judgment-needed; advisor decides). |
| M7 | Fill-blank marker form consistency | Question stem in each locale uses **either** `{{ blank }}` **or** `{{1}}/{{2}}` — never both. Mixed → ERROR (snapshot freezes whichever it sees first; grading silently corrupts). Detect via regex: `/\{\{\s*blank\s*\}\}/` and `/\{\{\s*\d+\s*\}\}/` — if both match in the same stem, fail. |
| M7b | Fill-blank slot count matches `role: "blank"` row count | Count `{{ blank }}` / `{{N}}` markers in stem; must equal number of `quiz_question_items WHERE role = 'blank' AND question_id = ?`. Mismatch → ERROR (runtime treats every blank row as a required slot). |
| M9 | No grammatical giveaways (article agreement) | Heuristic: scan EN options — if exactly one option begins with "an" and the others with "a" (or vice versa), flag WARN. KO has no equivalent. |
| M10 | Option lengths within ±25% | For MC items, compute char length per `primary_text`; flag any option whose length deviates more than ±25% from the median → INFO. |
| M14a | T/F has exactly 2 choice items | `quiz_question_items WHERE question_id = ? AND role = 'choice'` count = 2 for `question_type = 'true_false'`. |
| M14b | T/F stem has no negation | T/F `primary_text` doesn't contain `not`, `never`, `cannot`, `won't`, `아니다`, `없다`, `못한다` (heuristic; flag for review → INFO). |
| M14c | T/F stem has no qualifier | T/F `primary_text` doesn't contain `usually`, `often`, `most`, `대개`, `보통`, `대부분` → ERROR (rule M14: T/F only when categorical). |
| M15 | Pool size > `questions_per_attempt` | `COUNT(quiz_questions) > quizzes.questions_per_attempt`. Equal or less → ERROR (`shuffleQuestions` becomes a no-op; every retake selects the entire pool). |
| MC-1 | MC has 1 correct answer | For each MC question: `quiz_question_items WHERE role = 'choice' AND is_mc_correct = TRUE` count = 1. Multi-correct → INFO (rule "don't use it for attention checks — ambiguous to students"). |
| MC-2 | MC has 3 options | `quiz_question_items WHERE role = 'choice'` count = 3. 4+ → INFO (skill prefers 3 per Rodriguez 2005). |
| FB-1 | Fill-blank has at least one `role: 'blank'` item | `quiz_question_items WHERE role = 'blank'` count ≥ 1 per fill_blank question. |
| BL-1 | Both KO and EN translations present | Each `quiz_questions.id` has rows in `quiz_question_translations` for both `language_code = 'ko'` and `language_code = 'en'`. |
| BL-2 | Explanation present in both locales | `quiz_question_translations.explanation IS NOT NULL` for both ko and en. WARN (skill says "explanation is required on every item"; some legacy items may lack one). |
| QZ-1 | `passing_threshold ∈ [50, 100]` | Sanity: 0% or > 100% means broken admin save. |
| QZ-2 | `time_limit_minutes IS NULL OR > 0` | Zero is invalid. |

For each violation: capture rule id, severity, surface (`q<position>` for question-level, `q<position>.item<position>` for item-level), locale (if applicable), snippet (≤ 120 chars), one-line suggested fix.

### 4. Score against six perspectives via `advisor()`

Group judgment-heavy rules under **six perspectives**. One `advisor()` call per perspective. Each call sees how the rules interact in the actual content, not just whether they're satisfied independently.

#### Perspective rubric

| # | Perspective | Asks | Rules covered |
|---|---|---|---|
| **P1** | Item Mechanics | Are individual items technically valid? Stems clear, distractors functional, no rule violations. | M2, M4, M5, M6 (synonym distractors), M8 (no humor), M9, M10, MC option quality, T/F categorical |
| **P2** | Fairness & Bloom | Would an attentive viewer reliably answer correctly? Bloom 1–2 only, no traps, p ≈ 0.80–0.90. | Bloom level rubric, fairness rubric from seed-quiz §6 |
| **P3** | Transcript Grounding | Every item traces to a ≥20s emphasized transcript passage. | M11 + grounding-receipt convention. With `--srt`: full check against cue text. Without `--srt`: scan for `// [hh:mm:ss]` comments in the data module if accessible, else P3 = "receipts only — full grounding check skipped, --srt to enable." |
| **P4** | Coverage & Rotation | Pool size meaningful for retakes, type mix balanced, sections proportional, no fact duplication. | M12 (no fact overlap), M15, type mix targets (~60% MC / 15–20% T/F / 20–25% FB), pool sizing heuristics |
| **P5** | Bilingual Parity | KO native KO, EN native EN, both locales test the same fact, no idioms/puns/double negatives in either. | M13, bilingual policy |
| **P6** | Pedagogical Value | Explanations teach (one line, grounded), paraphrase ≥ verbatim balance (~3:2), retrieval not recognition. | Explanation quality rubric, paraphrase/verbatim balance |

Each perspective is scored on a **0–10 scale**:

| Score | Meaning |
|---|---|
| 10 | Exemplary — model quiz for the platform |
| 8–9 | Strong; would publish today |
| 7 | Passes the bar; not flashy but no blockers |
| 5–6 | Drift; publishable only with the named fixes |
| 3–4 | Material problems; would mislead, confuse, or false-fail attentive viewers |
| 0–2 | Absent or actively harmful (e.g., student app can't even render the items) |

#### Per-perspective advisor call

For each perspective, send:

- The **relevant content slice** from step 2:
  - P1, P2, P5, P6: every question's translations + items
  - P3: every question's stem + the SRT cues if `--srt` provided, or the data module receipt comments if available
  - P4: the full question list (for fact-overlap detection) + quiz settings (`questions_per_attempt`, type counts)
- The **deterministic violations** found in step 3 that fall under this perspective.
- The **rule list** for this perspective with one-line wording.
- This rubric verbatim:

> Score this perspective 0–10. Use the scale: 10 exemplary · 8–9 strong · 7 passes · 5–6 drift · 3–4 material problems · 0–2 absent.
> Score each individual rule 0–10 with the same scale.
> Return a **one-sentence headline** naming the single most impactful issue (the change that would lift this perspective the most), and 1–3 specific violations with quoted snippets from the question(s).

Cap: **6 advisor calls per lint** — one per perspective. If `--rule N` narrows the set, run only the perspectives whose rule list includes N. If `--locale ko|en` is given, P5 is reported as `N/A — single-locale lint`.

### 5. Compute the publish gate

A quiz **passes the lint** if every perspective scores ≥ 7. The minimum across the six perspectives is the gate value — a single 5 in P3 fails the gate even if every other perspective is 9. This is a notch lower than the seed-quiz skill's author-time gate (≥ 8 per Section 6 of that skill); lint runs on already-shipped content where the bar is "no longer drifting" rather than "fresh exemplar."

### 6. Render the report

```markdown
# Quiz lint: <course-slug> · ch<N>
**source**: draft   |   snapshot
**quiz id**: 42  ·  position 0  ·  passing 70%
**pool**: 8 questions  ·  questions_per_attempt: 5  ·  rotation ~37%
**type mix**: 5 MC  ·  1 T/F  ·  2 fill_blank   (target ~60/15-20/20-25)
**translations present**: ko ✓ · en ✓
**publish gate**: PASS  (or)  FAIL — minimum 7/10 per perspective; current min: 5 (P2)
**deterministic count**: 2 ERROR · 4 WARN · 3 INFO

## Scorecard

| # | Perspective              | Score | Headline |
|---|--------------------------|-------|----------|
| P1 | Item Mechanics           | 8/10  | one MC has near-synonym distractors (피 vs 보혈) |
| P2 | Fairness & Bloom         | **5/10** | q3 requires Apply-level reasoning ("which step would you use if…") — out-of-scope for an attention check |
| P3 | Transcript Grounding     | 7/10  | q5 has no receipt comment; cannot trace |
| P4 | Coverage & Rotation      | 9/10  | ✅ pool > attempts, sections proportional |
| P5 | Bilingual Parity         | 8/10  | EN q2 uses "literally" idiom that KO doesn't carry |
| P6 | Pedagogical Value        | 7/10  | q1 explanation is generic ("see lesson"); make it teach |

**Lowest score**: P2 (Fairness & Bloom, 5/10). Rewrite q3 from "which step would you use if X" to "which step does the speaker name first" — the Bloom drop is the single change that lifts the gate.

---

## Quiz-level findings

- ❌ **M15**: pool size 5 = `questions_per_attempt`. Add at least 3 more items so retakes meaningfully sample (target: pool ≥ attempts + 3).
- ℹ️  type mix is 100% MC; rule targets ~60% MC / 15–20% T/F / 20–25% fill_blank. Consider one T/F if any categorical statements exist in the lesson.

## q1 (multiple_choice · 10pts)
- ✅ no findings.

## q2 (multiple_choice · 10pts)
- ⚠️  **M6** ko: distractor "보혈" is a near-synonym of the key "피" in this lesson's context. Reroll the distractor from a different conceptual slot.
  - current key: "피"
  - current distractor: "보혈"
  - suggested: replace "보혈" with a non-equivalent term from elsewhere in the transcript (e.g., "기름부음").

## q3 (multiple_choice · 10pts)
- ❌ **Bloom-3** ko/en: stem requires Apply-level reasoning ("어떤 단계를 사용하시겠습니까?" / "Which step would you use if you encounter X?"). Rewrite as Remember-level recall.
  - current ko: "다음 상황에서 어떤 단계를 적용하시겠습니까?"
  - suggested: "강사가 첫 번째 단계로 명시한 것은 무엇인가요?"

## q5 (fill_blank · 10pts)
- ❌ **M7** ko: stem mixes `{{ blank }}` and `{{1}}` in the same question. The snapshot freezes one form silently; grading will corrupt.
  - current ko: "첫 번째 단계는 {{ blank }}이고, 두 번째 단계는 {{1}}이다."
  - suggested: pick ONE form. If order matters, use `{{1}}/{{2}}`; if not, use `{{ blank }}` × 2.
- ℹ️  no transcript receipt comment. Cannot verify ≥20s grounding without `--srt`.

(...etc per question. Questions with zero violations get `✅ no findings.`)
```

Per-violation format:

```
- <icon> **<rule-id>** <locale>: <one-line description>
  - current:   "<snippet>"
  - suggested: "<short fix>"
```

Icons: `❌` ERROR · `⚠️ ` WARN · `ℹ️ ` INFO · `✅` clean question.

Show every question in numeric order, even those with no violations — omitting reads as "didn't check."

### 7. End with the action menu

After the report, lead with the perspective holding the gate down:

> **Lowest perspective**: P2 (Fairness & Bloom, 5/10). The single change that lifts the publish gate is rewriting q3 from Apply-level to Remember-level.
>
> Found N ERRORs / M WARNs / K INFOs across 6 perspectives. Options:
>   1. Fix in the admin UI by hand at `/courses/<slug>/syllabus/quizzes/<quizId>` (best for one-off prose tweaks).
>   2. Edit the data module at `packages/database/scripts/seed-data/<slug>-ch<N>-quiz.data.mjs` if one exists, then re-run `node packages/database/scripts/seed-<slug>-ch<N>-quiz.mjs`. Note: re-running wipes `quiz_attempts` + `quiz_attempt_answers` (CASCADE) — confirm student-data impact first.
>   3. If only one perspective is failing, narrow with `--rule ID[,ID,...]` and re-run lint to confirm.
>   4. Tell me which questions to fix here and I'll patch the data module.

Do NOT auto-apply fixes. Lint is read-only. The user picks the path.

## Rule registry (canonical source)

The 15 mechanical rules and the advisor rubric live in `.claude/skills/seed-quiz-from-srt/SKILL.md`:

- **M1–M15**: Section 5 "Mechanical self-check (15 rules)" of that skill.
- **Per-type rules** (MC-*, T/F, FB-*): Section "Item-type rules (per type)".
- **Bilingual policy**: Section "Bilingual policy".
- **Bloom level + Fairness rubric**: Section 6 "Advisor revision loop" rubric table.

This skill does not duplicate rule wording — it cites by ID. When the seed-quiz skill's rules change, the lint skill picks them up by reference. If a rule is renumbered in seed-quiz, update both files in the same diff.

The rule ID prefixes:
- **M1–M15**: mechanical self-check (deterministic-friendly)
- **MC-N / FB-N**: per-type rules
- **QZ-N**: quiz-level settings (pool size, threshold, time limit)
- **BL-N**: bilingual presence (deterministic)
- **Bloom-N**: Bloom level (judgment, advisor)

## When NOT to use

- Quiz doesn't exist yet → use `seed-quiz-from-srt` to author it.
- User wants the lint to *fix* violations → no, lint is read-only. Fix in the admin UI or by editing the data module + re-seeding.
- User wants the published version checked → pass `--snapshot` to lint `classes.published_snapshot.quizzes[<quizId>]` instead of the draft tables.
- One-off check ("does this question have an explanation?") → run a single `SELECT explanation FROM quiz_question_translations`; lint is for full audits.
- Course content (title, FAQs, instructor bio, etc.) → out of scope; use `course-content-lint`.
- Quiz authoring quality before insert → out of scope; that's the seed-quiz skill's own advisor pass (Section 6 of that skill).

## Files this skill creates

None. Output is printed to the chat. If the user wants persistence:

```bash
# user-side, not the skill
> claude lint quiz <slug> <chapter> > lint-reports/<slug>-ch<N>-quiz-$(date +%Y%m%d).md
```

No data module, no executor, no DB writes — by design.
