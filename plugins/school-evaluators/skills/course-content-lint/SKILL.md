---
name: course-content-lint
description: Lint an existing course (KO + EN) against the 18 enrollment-conversion rules and structural conventions encoded in `seed-course-from-srt`. Read-only — produces a markdown report with two layers: a six-perspective scorecard (Conversion, Pedagogy, Theological Integrity, Bilingual Parity, Trust, Cohesion — each 0–10 with a publish gate at min ≥ 7) and a per-surface violation list (rule id, severity, snippet, suggested fix). Does not modify the database. Use when the user says "lint <course-slug>", "audit course content", "score this course", "check this course against the rules", or wants to spot drift after admin-UI edits.
user-invocable: true
argument-hint: "<course-slug> [--rule <N>[,N,...]] [--locale ko|en] [--snapshot] [--severity error|warn|info]"
---

# Lint an existing course against the enrollment ruleset

A read-only auditor that runs the 18 conversion rules + structural conventions defined in `.claude/skills/seed-course-from-srt/SKILL.md` against a course in the Neon database. Produces a markdown report grouped by surface (title, description, introduction, detail_page_json, features, FAQs, chapters, lessons, instructor bio, video chapters, bible references). **Does not write to the database, ever.**

## Why this skill exists

`seed-course-from-srt` enforces the rules at *author* time. After that, content drifts:

- Admin renames a title and adds the speaker's name back.
- A new instructor uploads a bio that leans on adjectives instead of a credential.
- FAQs balloon to 12 entries.
- "Zoom" sneaks into the description.
- A re-publish freezes the drifted state into the snapshot students read.

Lint runs the same rule corpus in reverse: read the live data, score it, surface violations. Same rules, different direction. Catches drift before publish.

## What it produces

A markdown report printed to the chat with two layers:

1. **Scorecard** — six perspectives, each scored **0–10** by `advisor()`, with a one-sentence headline per perspective and a **publish gate** (PASS if every perspective ≥ 7, else FAIL). The lowest-scoring perspective is the leverage point: fix it first to move the gate.
2. **Per-surface violations** — title, description, introduction, detail_page_json, features, FAQs, chapters, lessons, instructor bio, video chapters, bible references. Each violation lists rule id, severity, locale, current snippet, and a suggested fix.

**Does not auto-apply fixes** — the user picks how to act.

Severity levels for individual violations:

- **ERROR**: deterministic, unambiguous violation. Must fix before re-publish.
- **WARN**: judgment-based violation; the perspective scoring noticed it but author may have a reason.
- **INFO**: heads-up — boundary case (e.g., 4 FAQs vs the 5–8 rule) where the author may genuinely want the choice.

The scorecard and severity counts are independent: a perspective can score 8/10 even with WARN violations under it (because the score is holistic), and a perspective with no ERRORs can still score < 7 if the content reads as drift.

## Protocol (do these in order)

### 1. Preflight

- Confirm `<course-slug>` exists in `classes`. If not, `SELECT slug FROM classes ORDER BY slug` and exit with the list.
- Default source: live draft tables (`class_translations`, `class_features`, etc.). Pass `--snapshot` to lint `classes.published_snapshot` instead — that's what students currently see; useful when the user asks "is the published version OK to leave alone?"
- Optional flags:
  - `--rule N[,M,...]` — run only specific rule numbers (1–18 from seed-course; structural rules are S1–S6 — see Rule registry below).
  - `--locale ko|en` — restrict to one locale.
  - `--severity error|warn|info` — minimum severity in the report.

### 2. Pull the live shape

Read every surface the rules touch — one query per table, scoped to `class_id`:

```sql
SELECT id, slug, status, language, format, level, price, original_price, thumbnail_id, published_snapshot
  FROM classes WHERE slug = $1;

SELECT * FROM class_translations          WHERE class_id = $id;
SELECT cc.position, cct.* FROM class_chapters cc
  JOIN class_chapter_translations cct ON cct.chapter_id = cc.id
 WHERE cc.class_id = $id ORDER BY cc.position, cct.language_code;
SELECT cl.position, cl.content_json, cl.mux_video_id, clt.*
  FROM class_lessons cl
  JOIN class_chapters cc ON cc.id = cl.chapter_id
  LEFT JOIN class_lesson_translations clt ON clt.lesson_id = cl.id
 WHERE cc.class_id = $id ORDER BY cc.position, cl.position;
SELECT cf.icon, cf.position, cft.*
  FROM class_features cf
  JOIN class_feature_translations cft ON cft.feature_id = cf.id
 WHERE cf.class_id = $id ORDER BY cf.position, cft.language_code;
SELECT cf.position, cft.*
  FROM class_faqs cf
  JOIN class_faq_translations cft ON cft.faq_id = cf.id
 WHERE cf.class_id = $id ORDER BY cf.position, cft.language_code;
SELECT i.slug, it.*
  FROM class_instructors ci
  JOIN instructors i ON i.id = ci.instructor_id
  JOIN instructor_translations it ON it.instructor_id = i.id
 WHERE ci.class_id = $id;
SELECT mv.id, mv.duration_seconds, mv.chapters, mv.bible_references
  FROM mux_videos mv
  JOIN class_lessons cl ON cl.mux_video_id = mv.id
  JOIN class_chapters cc ON cc.id = cl.chapter_id
 WHERE cc.class_id = $id;
```

Parse `class_translations.detail_page_json` and `class_translations.introduction` (TipTap docs) into a flat block sequence so structural rules can walk them.

### 3. Run deterministic checks

Cheap, definitive. Severity defaults to ERROR unless noted. Encode each as a small predicate over the data pulled in step 2.

| ID | Rule | Check |
|---|---|---|
| 2a | No speaker name in title | Substring match against every `instructor_translations.name` (KO and EN forms) for instructors linked to this course. |
| 2b | No platform name in title | Substring match against `["줌", "Zoom", "YouTube", "유튜브", "온라인", "Online"]`. |
| 3a | Description ≤ 22 words | Word count per locale on `class_translations.description`. KO: whitespace-split tokens; EN: same. |
| 3b | No speaker / platform name in description | Same lists as 2a/2b. |
| 4a | Introduction has exactly 3 paragraphs | Walk `class_translations.introduction`; count `paragraph` nodes at the top level. |
| 5a | Exactly 4 features | `COUNT(*)` on `class_features` for this class. |
| 5b | Icon ∈ `case|book|check|message` | Each `class_features.icon` row. (Note: column is unconstrained `text`; only this rule catches typos.) |
| 7 | Each chapter has at least 1 lesson | `class_lessons` non-empty per `class_chapters.id`. |
| 8a | `duration_text` not raw minutes alone | Reject `^\s*\d+\s*분\s*$` (KO) or `^\s*\d+\s*minutes?\s*$` (EN). |
| 8b | `duration_text` excludes session/cadence count | Reject substrings like `세션`, `sessions`, `주`, `weeks`, `하루`, `per day`. |
| 11 | At most one `twoColumn` in `detail_page_json` | Walk; count nodes with `type === "twoColumn"`. |
| 14a | FAQ count ∈ [5, 8] | `COUNT(*)` on `class_faqs`. Outside band → ERROR; inside but at the edge (5 or 8) → INFO. |
| 16 | Scripture appears at most once on the entire detail page | Walk `detail_page_json` + `introduction`; count nodes whose text matches any `BOOK_NAME_TO_OSIS` key followed by chapter:verse pattern. |
| 18a | No surface-level discount theatre | `original_price IS NOT NULL AND price > 0` and `original_price > price` → INFO (intentional?). |
| S1 | No `heading(1)` in `detail_page_json` | TipTap walk; flag `heading` with `attrs.level === 1`. |
| S2 | No instructor section duplicated in `detail_page_json` | Heuristic: any `heading` whose text matches `/instructor|강사|소개/i` followed by a paragraph naming a linked instructor. WARN. |
| S3 | Bilingual independence — no parenthetical glosses | KO rows: regex `\([A-Za-z][^)]*\)` → WARN unless the parenthetical equals a registered org name in a small whitelist (e.g., `Destiny Encounters International`). EN rows: regex `\([ㄱ-힝][^)]*\)` → WARN. |
| S4 | Slug shape | `^[a-z][a-z0-9-]*[a-z0-9]$`, lowercased, hyphenated, English-derived. |
| S5 | First videoChapter at 0:00 | For each lesson with `mux_video_id`, confirm `mux_videos.chapters[0].startSeconds === 0`. (The seed executor enforces this; admin UI does not — see seed-course-from-srt skill, gap C2.) |
| S6 | Bible refs resolve | Every `bible_references[].bookOsisId` ∈ `bible_books.osis_id`. The admin UI validates on save, but historical refs may predate validation. |

For each violation: capture rule id, severity, surface (table + row + column), snippet (≤ 120 chars), one-line suggested fix.

### 4. Score against six perspectives via `advisor()`

Instead of scoring rules in isolation, group the judgment-heavy rules under **six evaluation perspectives** and let the advisor score each one *holistically*. One `advisor()` call per perspective. Each call sees how the rules interact in the actual content, not just whether they're satisfied independently.

#### Perspective rubric

| # | Perspective | Asks | Rules covered |
|---|---|---|---|
| **P1** | Conversion psychology | Would a hesitant browser actually enroll? Hook strength, objection handling, CTA discipline. | 1, 2c, 3c, 13, 14a, 15, 18a, 18b |
| **P2** | Pedagogical clarity | Would a prospective student know what they're getting and what changes after? | 4b, 6, 7, 8a, 8b, 9, 10a, 10b |
| **P3** | Theological integrity | Does it hold up to a discerning reader from this tradition? No guarantee theatre, scripture used soberly. | 16, 17, **O1** ("tease the outcome, don't teach it") |
| **P4** | Bilingual parity | Does KO read as native KO and EN as native EN, equivalent depth, zero cross-locale contamination? | S3 + every per-locale rule (1–18 evaluated against both rows) |
| **P5** | Trust & credibility | Is the instructor's authority concrete and verifiable, not adjective-driven? | 12, plus instructor name/headline checks |
| **P6** | Cohesion | If a user reads only the title, do they get the same message as the full page? Title → description → introduction → features → chapter list reinforce one outcome. | cross-cutting (draws on 2c, 3c, 4b, 5, 6) |

Each perspective is scored on a **0–10 scale**:

| Score | Meaning |
|---|---|
| 10 | Exemplary — model for other courses on the platform |
| 8–9 | Strong; would publish today |
| 7 | Passes the bar; not flashy but no blockers |
| 5–6 | Drift; publishable only with the named fixes |
| 3–4 | Material problems; would mislead, confuse, or under-sell |
| 0–2 | Absent or actively harmful |

#### Per-perspective advisor call

For each perspective, send:

- The **relevant content slice** from step 2 (e.g., P1 sees title + description + introduction + FAQs + closing paragraphs; P3 sees full `detail_page_json` + `introduction` + chapter descriptions; P5 sees instructor rows + the introduction's instructor link).
- The **deterministic violations** found in step 3 that fall under this perspective (the advisor needs to see what's already been counted).
- The **rule list** with one-line wording for each rule under this perspective.
- This rubric verbatim:

> Score this perspective 0–10. Use the scale: 10 exemplary · 8–9 strong · 7 passes · 5–6 drift · 3–4 material problems · 0–2 absent.
> Score each individual rule 0–10 with the same scale.
> Return a **one-sentence headline** naming the single most impactful issue (the change that would lift this perspective the most), and 1–3 specific violations with quoted snippets from the content.

Cap: **6 advisor calls per lint** — one per perspective. If `--rule N` narrows the set, run only the perspectives whose rule list includes N. If `--locale ko|en` is given, P4 is reported as `N/A — single-locale lint`.

### 5. Compute the publish gate

A course **passes the lint** if every perspective scores ≥ 7. The minimum across the six perspectives is the gate value — a single 5 in P3 fails the gate even if every other perspective is 9. This matches the seed-course skill's `advisor: ≥ 8` author-time gate (Section 7 of that skill), but lowered to 7 since lint runs on already-shipped content where the bar is "no longer drifting" rather than "fresh exemplar."

### 6. Render the report

```markdown
# Course lint: <slug>
**source**: draft  (or  snapshot)
**status**: published  (or  draft)
**translations present**: ko ✓ · en ✓     (note any missing)
**publish gate**: PASS  (or)  FAIL — minimum 7/10 per perspective; current min: 5 (P3)
**deterministic count**: 3 ERROR · 5 WARN · 2 INFO

## Scorecard

| # | Perspective              | Score | Headline |
|---|--------------------------|-------|----------|
| P1 | Conversion psychology    | 7/10  | description runs 28 words; FAQs miss refund/access |
| P2 | Pedagogical clarity      | 9/10  | ✅ scope and outcomes clear |
| P3 | Theological integrity    | **5/10** | guarantee language in introduction ¶2 ("you will hear God") |
| P4 | Bilingual parity         | 8/10  | EN bio contains hangul parenthetical "(벤 림)" |
| P5 | Trust & credibility      | 6/10  | bio leads with adjectives, no named org/year |
| P6 | Cohesion                 | 8/10  | ✅ surfaces reinforce one outcome |

**Lowest score**: P3 (theological integrity, 5/10) — fix the guarantee language in `introduction` ¶2 first; that single change moves the gate.

---

## Per-surface violations

## Title
- ❌ **2a** ko: title contains speaker name "벤 림". Instructors render via `class_instructors`; remove the name and let the `introduction` link carry credibility.
  - current:  "벤 림 · 척 피어스의 한국을 향한 줌예배"
  - suggested: "한국을 향한 중보 — 6주 가이드 실전"
- ❌ **2b** ko: title contains platform word "줌". Move to `classes.format`.

## Description
- ⚠️  **3a** en: 28 words (rule: ≤22). Cut adjectives.
  - current: "An online course for any believer who feels stuck in their prayer life and wants more…"

## Introduction
- ❌ **4a** ko: 5 top-level paragraphs (PAS expects exactly 3).
- ⚠️  **4b** ko: ¶3 ends with a CTA ("지금 등록하세요"). Replace with a forward-leaning question.

## Features
- ❌ **5a**: 6 features defined; rule requires exactly 4. Likely two redundant outcome cards — drop down to 1 outcome.

## FAQs
- ✅ count and grouping look fine (6 FAQs covering fit, format, refund).

## Detail page (TipTap)
- ⚠️  **17**: paragraph 4 contains guarantee language ("you *will* hear God speak"). Reframe as access-to-training.
- ℹ️   **11**: two `twoColumn` blocks present — rule caps at 1. Confirm the second is intentional.

## Instructor bio (Ben Lim)
- ⚠️  **12** en: leads with adjectives ("a powerful and prophetic voice…") instead of a verifiable credential. Move named org/role/year-count to sentence 1.

## Video chapters
- ✅ every lesson with a mux_video_id has `chapters[0]` starting at 0:00.

## Bible references
- ❌ **S6**: lesson `cc=2/cl=1` references `bookOsisId="ENO"` (Enoch — pseudepigrapha, not in `bible_books`). Drop or replace.
```

Per-violation format:

```
- <icon> **<rule-id>** <locale>: <one-line description>
  - current:   "<snippet>"
  - suggested: "<short fix>"
```

Icons: `❌` ERROR · `⚠️ ` WARN · `ℹ️ ` INFO · `✅` clean section.

Surfaces with zero violations get a single `✅ clean` line — don't omit them, omitting reads as "didn't check".

### 7. End with the action menu

After the report, surface what the user can do — and stop. Lead with the perspective that's holding the gate down:

> **Lowest perspective**: P3 (theological integrity, 5/10). The single change that lifts the publish gate is fixing the guarantee language in `introduction` ¶2 ("you will hear God speak" → reframe as access-to-training).
>
> Found N ERRORs / M WARNs / K INFOs across 6 perspectives. Options:
>   1. Fix in the admin UI by hand (best for one-off prose tweaks).
>   2. Run `seed-course-from-srt` evaluate-and-feedback mode (Section 9 of that skill) to regenerate specific surfaces from the original SRT.
>   3. Tell me which violations to fix here and I'll patch the data module if one exists.
>   4. If only one perspective is failing, narrow with `--rule N[,M,...]` and re-run lint to confirm.

Do NOT auto-apply fixes. Lint is read-only. The user picks the path.

## Rule registry (canonical source)

The 18 enrollment rules and the structural conventions live in `.claude/skills/seed-course-from-srt/SKILL.md`:

- **1–18**: Section 4 "Enrollment conversion ruleset" (lines ~171–223 of that skill at the time of writing).
- **S1–S6**: derived from Section 3 "Design the data module" + Section 3a "video chapters" + Section 3b "bible references".
- **O1** ("tease the outcome, don't teach it"): the overarching principle at the top of Section 4.

This skill does not duplicate the rule text. When the seed-course skill's rules change, the lint skill picks them up by reference. If a numbered rule in seed-course is renumbered, update both files in the same diff.

## When NOT to use

- Course doesn't exist yet → use `seed-course-from-srt` to author it.
- User wants the lint to *fix* violations → no, lint is read-only. Use the seed-course evaluate branch (Section 9) or the admin UI for fixes.
- User wants draft-vs-published diff → that's a separate question; lint can be run twice (once with `--snapshot`, once without) but it isn't a diff tool.
- One-off check ("is the title too long?") → run a single `SELECT title FROM class_translations`; lint is for full audits.
- Quiz content → out of scope. Quiz quality is enforced at author time by `seed-quiz-from-srt`'s own rule set; a separate `quiz-content-lint` would be needed.

## Files this skill creates

None. Output is printed to the chat. If the user wants persistence:

```bash
# user-side, not the skill
> claude lint <slug> > lint-reports/<slug>-$(date +%Y%m%d).md
```

No data module, no executor, no DB writes — by design.
