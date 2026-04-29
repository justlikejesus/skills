---
name: seed-quiz-from-srt
description: Seed a bilingual (KO primary + EN secondary) low-stakes comprehension quiz onto an existing course chapter from one or more SRT transcripts. Generates a pool of items — as many as the lesson's emphasized content honestly supports, no fixed cap — using only the types the student app currently supports (multiple_choice, true_false, fill_blank), grounded in emphasized transcript passages. Pool size always exceeds `questionsPerAttempt` so retakes meaningfully sample different items. Designed as an attention check (attentive viewer p ≈ .80–.90), never a summative exam. Use when the user says "make a quiz for chapter N of <course>", "seed a quiz from this SRT", or anything equivalent.
user-invocable: true
argument-hint: "<course-slug> <chapter-position-0indexed> <srt-path> [more-srt-paths...]"
---

# Seed a chapter quiz from SRT transcripts

A one-shot, bilingual (KO + EN), **low-stakes attention-check** quiz attached to an existing chapter of an existing course. The point of this quiz is to flag students who did not watch the lesson — **not to fail attentive students**. Every design choice inside this skill is biased toward "fair, clearly answerable after watching" over "discriminating."

## Platform constraints (as of this writing)

- **Only `multiple_choice`, `true_false`, and `fill_blank` render on the student app.** `matching` and `ordering` are authorable in the admin UI and have DB columns, but `apps/school/src/features/quiz/actions.ts` explicitly skips them (`"student UI doesn't support these yet — skip"` at line 177, and `"doesn't grade these; mark as 0"` at line 617). **This skill does not emit matching or ordering items.** When the student UI adds support, revisit this.
- **Re-seeding destroys historical attempt data.** `quiz_attempt_answers.question_id` has `ON DELETE CASCADE` against `quiz_questions.id`, so wiping the questions wipes every student's answer rows. `quiz_attempts.attempted_questions` is a JSON array of question IDs with **no FK constraint** — leaving the attempt rows behind would orphan them with stale IDs and break the next submission with a foreign-key error. The executor therefore wipes both `quiz_attempts` and `quiz_questions` for the quiz on every re-run, and the skill surfaces all three wipe counts before running so the user can decide.
- **`class_chapters.position` is 0-indexed.** The first chapter is `position = 0`, not `1`. `<chapter-position-0indexed>` in the argument hint is that raw DB value. If the user says "chapter 1" colloquially, clarify whether they mean position 0 (first chapter, Korean "1장") or position 1 (second chapter); do not guess.

Two files per invocation:

- `packages/database/scripts/seed-data/<course-slug>-ch<N>-quiz.data.mjs` — authored KO + EN content
- `packages/database/scripts/seed-<course-slug>-ch<N>-quiz.mjs` — idempotent executor, single transaction, preflight-guarded

Re-running finds the same quiz row by `(chapterId, position=0)` and wipes + reinserts all questions, and also wipes `quiz_attempts` for the same quiz (their `attempted_questions` JSON would otherwise hold stale IDs and break the next submit). **Wiping cascades through `quiz_attempt_answers` for the answer rows.** The Claude Code session must warn the user of all three counts (N questions + M answer rows + K attempts deleted) before running.

## Guiding philosophy (what the research says for *this kind* of quiz)

Evidence base: Rodriguez & Albano (2017) 26-rule condensation of Haladyna/Downing; Rodriguez (2005) on option count; Roediger & Karpicke on retrieval practice; Agarwal field work on low-stakes quizzing; Cathy Moore on e-learning knowledge checks; International Test Commission on translation adaptation.

- **Target p-value 0.80–0.90** for an attentive viewer. Summative exams aim at .50–.70 to discriminate; an attention check aims high — the quiz should reward watching, not trap viewers.
- **Bloom levels 1–2 only** (Remember, Understand). Never Apply / Analyze / Evaluate — those produce false negatives (attentive viewer, still wrong).
- **Ground every item** in ≥20 seconds of emphasized transcript (repetition, explicit definition, long dwell, "the key point is…"). No passing-trivia items.
- **Paraphrase ≥ verbatim.** Aim for roughly 3 paraphrased : 2 verbatim across the pool. Verbatim-only items reward surface pattern matching; paraphrased items force meaningful retrieval.
- **Explanation is required on every item** — mitigates the negative-suggestion effect (Roediger & Marsh, 2005): without the correct answer shown on submit, students can actually learn the lure.
- **3 options for MC, not 4.** Rodriguez (2005) meta-analysis: 3 ≥ 4 ≥ 5 on reliability and 10–15% faster. Two-thirds of 4-option items already have only 1–2 functioning distractors in the wild.
- **Bilingual cleanliness** — no idioms, no puns, no double negatives in KO *or* EN. Items written to translate cleanly per the International Test Commission guidelines.

## Protocol (do these in order)

### 1. Preflight — hard-fail, never infer

All of the following are hard errors. **Never guess, never infer, never hallucinate** a course / chapter / file.

- `classes.slug = <course-slug>` → if no row, print the list of valid slugs and exit.
- `class_chapters WHERE class_id = ? AND position = ?` → if no row, print the chapter count and each chapter's `position + KO title` for that course, then exit.
- `<srt-path>` readable → ENOENT or non-`.srt` extension → exit with the path shown.

The executor performs these same checks at runtime. The skill body performs them at authoring time (via a quick read of the DB + a `ls`) so it never drafts a quiz for a nonexistent target.

### 2. Read the SRT(s) in full

No skim. A good comprehension check comes from knowing the *shape* of the lesson: where the instructor pauses, repeats, defines, and emphasizes. Identify while reading:

- **Transcript segments.** 3–5 major sections by topic-shift cue or long pause. Mark their start/end timestamps.
- **Emphasized passages.** ≥20 seconds of repetition, definition, explicit marker words ("the most important thing is", "remember this"), or long dwell on a single point. Every item you write must come from one of these.
- **Explicit categories.** Lists of parallel items ("three steps…", "four signs…"), chronological sequences, cause→effect chains. Since the skill doesn't emit matching/ordering, use these to author MC questions like "which step comes right after X?" or "how many stages does the speaker name?"
- **Explicit binaries.** Statements the instructor frames as categorically true or false with no qualifier — these are the only candidates for a T/F item.

### 3. Build the blueprint

Before writing any question, produce a small blueprint (kept in a comment block at the top of the data module):

```
// Blueprint
// Section 1 (00:00–08:42, ~9m): <topic>  → 2 items
// Section 2 (08:42–22:15, ~14m): <topic> → 3 items
// Section 3 (22:15–36:00, ~14m): <topic> → 3 items
//
// Pool size: 8 items · questionsPerAttempt: 5 · rotation: each attempt samples 5 of 8
// Type mix: ~5 MC · 1 T/F · 2 fill_blank (≈ 60% MC, 15% T/F, 25% fill_blank)
```

**Pool sizing (no fixed total — quality drives count):**

- Author every item the lesson honestly supports under the quality bar. Per-section heuristic: roughly **one item per ~5–7 minutes** of emphasized content. A 30-minute lesson typically yields 6–10 quality items; a 60-minute lesson, 10–15.
- **Hard floor: pool size > `questionsPerAttempt`.** If pool ≤ `questionsPerAttempt`, every retake selects the entire pool — defeats `shuffleQuestions`. Aim for pool ≥ `questionsPerAttempt + 3` so retakes feel materially different (≥ 60% chance any given item rotates out across two attempts).
- **`questionsPerAttempt` default: 5.** Reasonable for an attention check; raise for deeper chapters, lower for short ones. Always less than the pool.
- **No upper cap.** If the lesson genuinely has 12 distinct, well-grounded points worth checking, author 12 — don't truncate to hit a number. The quality rules below (especially #11 ≥20s grounding and #12 no fact overlap) are the real ceiling.

**Type mix rules (only student-supported types are allowed — see Platform Constraints):**

- **Target ratio**: ~60% `multiple_choice`, ~15–20% `true_false`, ~20–25% `fill_blank`. Round to whole items at the pool size you actually have. For a pool of 8: e.g., 5 MC · 1 T/F · 2 fill_blank. For a pool of 12: e.g., 7 MC · 2 T/F · 3 fill_blank.
- **Drop T/F when the lesson has no categorical statement** (every factual claim qualified with "usually / often / most"). Reallocate that slot to MC. T/F can be 0 across the entire pool — it's not required.
- **Never emit `matching` or `ordering`.** The student app doesn't render or grade them — the attempt UI silently skips them and the grader marks them 0. Shipping these types would mean every attentive student fails those items.
- No two items test the same fact (rule 12 below).

### 4. Draft each item

Below each item in the data module, add a **comment with the transcript quote + timestamp** you used as the source. This is the grounding receipt — the advisor and future-you check this.

```js
// [00:14:32] "... and the three stages are preparation, then the offering, then the sealing."
{
  type: "multiple_choice",
  ...
}
```

Per-item rules are in the **Item-type rules** section below. The **13 forbidden rules** section is the mechanical self-check after the draft.

### 5. Mechanical self-check (15 rules)

Run every item against this list. Any failure → revise or drop the item.

1. **Only supported types.** Every item is `multiple_choice`, `true_false`, or `fill_blank`. Zero `matching`, zero `ordering`.
2. **No negative stems.** No "NOT", "EXCEPT", "LEAST", "only" as the negation device. Rephrase positively.
3. **No "all of the above" / "none of the above" / "I don't know"** options.
4. **No absolute determiners in options** — "always", "never", "completely", "absolutely" — unless the source is genuinely absolute.
5. **No technically-true distractors.** A distractor must be unambiguously wrong as an answer to the stem.
6. **No synonym or near-equivalent distractors.** If a distractor means substantially the same thing as the correct answer in this lesson's context, it is invalid — the student is forced into a coin flip between two right answers. Examples to watch for in Korean theology vocabulary: `피` vs `보혈` (both = blood / Christ's blood), `회개` vs `회개하기` (repentance vs to repent), `선포` vs `고백` (declaration vs confession when used interchangeably), `시작` vs `출발`. Same check across locales — a KO distractor whose EN gloss is a synonym of the EN key is also invalid. When in doubt, ask: "could a careful student argue this option is also correct?" If yes, drop it. This applies to MC `choice` options AND `fill_blank` `wordbank` distractors.
7. **Fill-blank: write placeholders as `{{ blank }}` for any-order grading, `{{1}}`/`{{2}}` for ordered grading.** The marker form selects the **grading rule** at publish time — they are NOT interchangeable syntaxes. Pick the form that matches what the question actually tests:
   - **`{{ blank }}` — any-order (multiset match).** All correct words must be present in the bag of submissions, but slot identity is ignored. If correct = `[정의, 평화]` and the student fills `(평화, 정의)`, they get full credit. Use for stems like "강사가 언급한 세 가지는 `{{ blank }}`, `{{ blank }}`, `{{ blank }}`이다." where the answers are a set, not a sequence.
   - **`{{1}}` / `{{2}}` — ordered (per-slot match).** Each numbered slot must hold its specific answer. Slot mismatch = wrong. Use for stems where the surrounding context (Korean particles, semantic role) fixes which word goes where: "마음을 다하고 `{{1}}`(으)로 `{{2}}`를 사랑하라."
   - **Single-blank questions**: both forms grade identically. Default to `{{ blank }}` for visual consistency with the rest of the seed-skill output.
   - **Never mix the two forms in one question.** The admin UI's form toggle auto-converts on switch and the substituter accepts both, but a single saved question commits to one mode — `buildQuizSnapshots` detects the form from the draft text before substitution and freezes `fillBlankMode: "ordered" | "any_order"` on the snapshot. Mixed text would resolve a sequential `{{ blank }}` and a positional `{{1}}` against the same blank item.

   Other shorthands (underscores `____`, `<blank>`, etc.) are NOT recognized and will render literally to the student. The runtime treats every `role: "blank"` row as its own required slot — adding a "synonym" row creates a second hidden slot the student can never see, costing them points. Multiple distinct blanks per question are fine (one marker per slot, each with its own answer); synonym alternates for the *same* slot are not. If two synonyms feel equally correct (`피` vs `보혈`, `회개` vs `회개하기`), **rephrase the stem** so only one fits — e.g. switch the surrounding particle/article, or use a more specific cue from the transcript that admits only the canonical word.
8. **No humor / absurd distractors.**
9. **No grammatical giveaways** — article agreement ("a/an"), singular/plural.
10. **Option lengths within ±25%** of each other. The key is not systematically longest.
11. **Source passage ≥20 seconds OR explicitly emphasized.** Trivia mentioned once in passing is banned.
12. **Coverage:** no two items in the pool test the same fact. Ranking the items by transcript timestamp, each consecutive pair should be testing distinct beats — if a draft has two items on the same idea, drop one or rewrite it onto a different fact from the lesson.
13. **Bilingual clean:** no idioms, puns, double negatives in KO *or* EN.
14. **T/F only when categorical.** No "usually / often / most" in the statement; no negation; no double negatives.
15. **Pool size > `questionsPerAttempt`.** `QUESTIONS.length` must strictly exceed `QUIZ.questionsPerAttempt`; otherwise `shuffleQuestions` becomes a no-op (every retake selects the entire pool) and students see the same items every attempt. Aim for `QUESTIONS.length ≥ questionsPerAttempt + 3` for meaningful rotation.

### 6. Advisor revision loop (≥ 8 / 10 to proceed)

**This step is mandatory.** After the draft + mechanical self-check, call `advisor()` and iterate until the score is ≥ 8 on a 1–10 scale.

```
iteration = 1
MAX_ITERATIONS = 4

loop:
    1. Write the data module to disk (always durable before the advisor call — if
       the session dies mid-call, the latest draft survives).
    2. Call advisor() with the full draft + the SRT excerpts each item cites.
    3. Parse score + per-item critique.
    4. If score >= 8 AND zero rule-list violations → break, proceed to step 7.
    5. Else apply the advisor's specific fixes to the flagged items only; re-run
       the mechanical self-check; increment iteration.
    6. If iteration > MAX_ITERATIONS → stop; show the user the advisor's
       critique + current score and ask: "advisor stuck at <N>/10 after 4
       passes — proceed anyway, regenerate from scratch, or abandon?"
```

What the advisor is asked to score (tell it this rubric verbatim):

| Dimension | Weight | Fails if |
|---|---|---|
| Transcript grounding (≥20s, emphasized, traceable quote) | heavy | any item traces to passing trivia |
| Bloom level (Remember / Understand only) | heavy | any item requires Apply / Analyze / Evaluate |
| Fairness (p ≈ .80–.90 attentive viewer) | heavy | ambiguous key · technically-true distractor · twist wording |
| 14-rule compliance | hard gate | any single violation caps the score at 7 |
| Coverage proportional, no duplicate facts | medium | ≥2 items test the same concept |
| Paraphrase / verbatim balance (~3:2 of 5) | light | all verbatim or all paraphrased |
| Bilingual cleanliness | medium | one locale leans on wording the other can't carry |
| Explanation quality (one line, grounded, teaches) | medium | missing or generic |

### 7. Present what's going in, then run

Per the user's meta-instruction, after the advisor clears you:

1. **Pre-flight impact check.** Before telling the user anything else, run a single `psql` (or use `@repo/database/pool`) to count what will be wiped:
   - existing question count for this `(chapter_id, position)` quiz
   - existing `quiz_attempt_answers` count that CASCADEs through those questions
   Do not skip this — the CASCADE can silently erase a large amount of student data if students have already been taking the quiz.
2. Tell the user:
   - advisor score + iteration count (`advisor: 9/10 on pass 2`)
   - pool size and `questionsPerAttempt` (`pool: 8 items · 5 per attempt — retakes rotate ~37% of items on average`)
   - the type mix (`5 MC · 1 T/F · 2 fill_blank`)
   - KO stem short-form of every item in the pool
   - EN stem short-form of every item in the pool
   - **destructive impact**, formatted explicitly: `will delete N existing questions AND M student answer rows on quiz "<title>" (CASCADE)` — if M > 0, mark it so the user sees it and has a chance to cancel
3. On "go" / "now do it" / "proceed", run `node packages/database/scripts/seed-<course-slug>-ch<N>-quiz.mjs`.
4. Report the resulting quiz id and admin-UI next step (`open /courses/<slug>/syllabus/quizzes/<id> to tune thresholds / enable requirement`).

## Data module shape

Follow this structure exactly. The executor imports named exports from this file.

```js
// packages/database/scripts/seed-data/<course-slug>-ch<N>-quiz.data.mjs
//
// Blueprint (comment block — keep up to date)
// Section 1 (00:00–08:42): legal grounds          → 1 item (MC)
// Section 2 (08:42–22:15): covenant + altars      → 2 items (T/F + MC)
// Section 3 (22:15–36:00): ignorance + freedom    → 2 items (fill_blank + MC)
//
// Pool size: 5 · questionsPerAttempt: 4 · ~20% of items rotate per retake
// Type mix: 3 MC · 1 T/F · 1 fill_blank  (≈ 60% MC / 20% T/F / 20% fill_blank)
// (Larger lessons should yield 8–12 items — see "Pool sizing" rule.)

/** Identifies the target. Executor hard-fails if not found. */
export const TARGET = {
  courseSlug: "invisible-altars",   // classes.slug
  chapterPosition: 1,                // class_chapters.position (0-indexed as stored; pass 0,1,2,...)
};

/** Quiz-level metadata. Re-synced on every run (wipe + reinsert semantics). */
export const QUIZ = {
  position: 0,                       // stable identity within the chapter (single quiz per chapter)
  isRequired: false,
  shuffleQuestions: true,            // server picks `questionsPerAttempt` of `QUESTIONS.length` per attempt
  questionsPerAttempt: 4,            // MUST be < QUESTIONS.length (rule #15) — otherwise every attempt sees the full pool
  passingThreshold: 70,              // 70% → 3-of-4 to pass, fair for attentive viewers
  timeLimitMinutes: null,            // no time pressure for a comprehension check
  translations: {
    ko: {
      title: "1장 · 듣고 있었나요? 복습 퀴즈",
      description: "영상의 핵심 내용을 다시 확인하는 복습 퀴즈입니다. 매 시도마다 문항이 무작위로 선택됩니다.",
    },
    en: {
      title: "Chapter 1 — Attention Check",
      description: "A short comprehension review on the main points of this session. Questions are sampled at random each attempt.",
    },
  },
};

/**
 * Question array. Order here = final position order in the DB.
 * Every entry MUST include a transcript-quote comment above it.
 */
export const QUESTIONS = [
  // [00:03:12] "... there are three legal grounds Satan uses: ignorance, disobedience, and covenant."
  {
    type: "multiple_choice",          // one of: multiple_choice | true_false | fill_blank  (matching/ordering exist in the DB enum but are not emitted by this skill)
    points: 10,
    translations: {
      ko: {
        text: "강사가 사탄이 삶에 접근하는 ‘합법적 통로(legal grounds)’의 개수를 몇 개로 말했나요?",
        explanation: "강의 초반 0:03:12 부근에서 ‘세 가지 합법적 통로’라고 명시했습니다.",
      },
      en: {
        text: "How many ‘legal grounds’ does the speaker name as Satan's access points?",
        explanation: "The speaker names three legal grounds early in the session (around 00:03:12).",
      },
    },
    items: [
      // role: 'choice' for multiple_choice + true_false
      { role: "choice", isCorrect: true,  translations: { ko: { primary: "3개" }, en: { primary: "Three" } } },
      { role: "choice", isCorrect: false, translations: { ko: { primary: "2개" }, en: { primary: "Two" } } },
      { role: "choice", isCorrect: false, translations: { ko: { primary: "5개" }, en: { primary: "Five" } } },
    ],
  },

  // [00:09:45] "... and whenever we talk about a covenant, it is always a two-way agreement."
  {
    type: "true_false",
    points: 10,
    translations: {
      ko: { text: "강사에 따르면 언약(covenant)은 항상 쌍방 합의이다.", explanation: "쌍방 합의라고 명시했습니다 (0:09:45)." },
      en: { text: "According to the speaker, a covenant is always a two-way agreement.", explanation: "The speaker states this explicitly around 00:09:45." },
    },
    items: [
      { role: "choice", isCorrect: true,  translations: { ko: { primary: "참" }, en: { primary: "True" } } },
      { role: "choice", isCorrect: false, translations: { ko: { primary: "거짓" }, en: { primary: "False" } } },
    ],
  },

  // [00:18:20] "... the three stages of breaking an altar are repentance, the blood, and the declaration."
  {
    type: "fill_blank",
    points: 10,
    translations: {
      ko: { text: "악한 제단을 헐 때의 세 단계 중 첫 번째는 {{ blank }}이다.", explanation: "첫 단계는 회개라고 명시했습니다 (0:18:20)." },
      en: { text: "The first of the three stages for breaking an altar is {{ blank }}.", explanation: "The speaker names repentance as the first stage (00:18:20)." },
    },
    items: [
      // role: 'blank' — exactly one canonical answer per `{{ blank }}` placeholder. The runtime
      // counts every `blank` row as a separate required slot, so do NOT add synonym rows
      // here. If two synonyms feel equally valid, rephrase the stem to admit only one.
      { role: "blank", isCorrect: true, translations: { ko: { primary: "회개" }, en: { primary: "repentance" } } },
      // Optional word bank — surface 3–5 distractors pulled from the same lesson when the answer is a proper noun / number / technical term.
      { role: "wordbank", isCorrect: false, translations: { ko: { primary: "선포" }, en: { primary: "declaration" } } },
      { role: "wordbank", isCorrect: false, translations: { ko: { primary: "보혈" }, en: { primary: "blood" } } },
      { role: "wordbank", isCorrect: false, translations: { ko: { primary: "금식" }, en: { primary: "fasting" } } },
    ],
  },

  // [00:24:08] "... ignorance opens a door — it's the default state of every believer until it's taught out."
  {
    type: "multiple_choice",
    points: 10,
    translations: {
      ko: {
        text: "강사는 ‘무지(ignorance)’를 어떤 상태로 설명했나요?",
        explanation: "0:24:08에서 무지를 ‘가르침을 받기 전까지 모든 신자의 기본 상태’라고 설명했습니다.",
      },
      en: {
        text: "How does the speaker describe ‘ignorance’ in this context?",
        explanation: "Around 00:24:08 the speaker calls ignorance the default state of every believer until it's taught out.",
      },
    },
    items: [
      { role: "choice", isCorrect: true,  translations: { ko: { primary: "가르침을 받기 전까지의 기본 상태" }, en: { primary: "The default state until it is taught out" } } },
      { role: "choice", isCorrect: false, translations: { ko: { primary: "의도적인 반역 상태" },            en: { primary: "A state of deliberate rebellion" } } },
      { role: "choice", isCorrect: false, translations: { ko: { primary: "구원받기 이전만 해당되는 상태" }, en: { primary: "A state that only applies before salvation" } } },
    ],
  },

  // [00:31:00] "... the goal of all of this is that the church walks free — free of those grounds."
  {
    type: "multiple_choice",
    points: 10,
    translations: {
      ko: {
        text: "강사가 말한 이 훈련의 최종 목표는 무엇인가요?",
        explanation: "0:31:00에서 교회가 ‘그 통로들로부터 자유롭게 걷는 것(walking free)’을 목표로 제시합니다.",
      },
      en: {
        text: "What does the speaker name as the ultimate goal of this training?",
        explanation: "At 00:31:00 the speaker frames the goal as the church ‘walking free’ of those grounds.",
      },
    },
    items: [
      { role: "choice", isCorrect: true,  translations: { ko: { primary: "교회가 그 통로들로부터 자유롭게 사는 것" }, en: { primary: "That the church walks free of those grounds" } } },
      { role: "choice", isCorrect: false, translations: { ko: { primary: "모든 영적 전쟁에서 승리하는 것" },        en: { primary: "Winning every spiritual conflict" } } },
      { role: "choice", isCorrect: false, translations: { ko: { primary: "더 많은 중보자를 배출하는 것" },         en: { primary: "Raising up more intercessors" } } },
    ],
  },
];
```

## Executor skeleton

Mirror this when writing `seed-<course-slug>-ch<N>-quiz.mjs`. The template is small; resist adding features.

```js
#!/usr/bin/env node
/**
 * Seed a chapter quiz. Idempotent: finds the quiz by (chapter_id, position)
 * and wipes + reinserts all questions on every run. Single transaction.
 *
 * Usage:
 *   node packages/database/scripts/seed-<slug>-ch<N>-quiz.mjs
 */
import { Pool } from "pg";
import { config } from "dotenv";
import path from "node:path";
import { fileURLToPath } from "node:url";

import { TARGET, QUIZ, QUESTIONS } from "./seed-data/<slug>-ch<N>-quiz.data.mjs";

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
const REPO_ROOT = path.resolve(__dirname, "../../..");

config({ path: path.join(REPO_ROOT, "apps/school/.env") });

if (!process.env.DATABASE_URL) {
  console.error("DATABASE_URL missing. Expected it in apps/school/.env");
  process.exit(1);
}

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function upsertTranslation(client, { table, entityCol, entityId, lang, columns }) {
  const cols = Object.keys(columns);
  const vals = Object.values(columns);
  const existing = await client.query(
    `SELECT 1 FROM ${table} WHERE ${entityCol} = $1 AND language_code = $2`,
    [entityId, lang],
  );
  if (existing.rowCount > 0) {
    if (cols.length === 0) return;
    const setSql = cols.map((c, i) => `${c} = $${i + 3}`).join(", ");
    await client.query(
      `UPDATE ${table} SET ${setSql} WHERE ${entityCol} = $1 AND language_code = $2`,
      [entityId, lang, ...vals],
    );
  } else {
    const insertCols = [entityCol, "language_code", ...cols];
    const placeholders = insertCols.map((_, i) => `$${i + 1}`).join(", ");
    await client.query(
      `INSERT INTO ${table} (${insertCols.join(", ")}) VALUES (${placeholders})`,
      [entityId, lang, ...vals],
    );
  }
}

async function resolveChapter(client) {
  const course = await client.query(
    `SELECT id FROM classes WHERE slug = $1`,
    [TARGET.courseSlug],
  );
  if (course.rowCount === 0) {
    const all = await client.query(`SELECT slug FROM classes ORDER BY slug`);
    console.error(`Course slug '${TARGET.courseSlug}' not found. Valid slugs:`);
    for (const r of all.rows) console.error(`  · ${r.slug}`);
    process.exit(1);
  }
  const classId = course.rows[0].id;

  const chapter = await client.query(
    `SELECT id FROM class_chapters WHERE class_id = $1 AND position = $2`,
    [classId, TARGET.chapterPosition],
  );
  if (chapter.rowCount === 0) {
    const all = await client.query(
      `SELECT cc.position, ct.title
         FROM class_chapters cc
         LEFT JOIN class_chapter_translations ct
           ON ct.chapter_id = cc.id AND ct.language_code = 'ko'
        WHERE cc.class_id = $1 ORDER BY cc.position`,
      [classId],
    );
    console.error(
      `Chapter position ${TARGET.chapterPosition} not found for '${TARGET.courseSlug}'. Actual chapters:`,
    );
    for (const r of all.rows) console.error(`  · position=${r.position} · ${r.title ?? "(no ko title)"}`);
    process.exit(1);
  }
  return chapter.rows[0].id;
}

async function findOrCreateQuiz(client, chapterId) {
  const existing = await client.query(
    `SELECT id FROM quizzes WHERE chapter_id = $1 AND position = $2`,
    [chapterId, QUIZ.position],
  );
  let quizId;
  if (existing.rowCount > 0) {
    quizId = existing.rows[0].id;
    // Count what will be wiped. `quiz_attempt_answers.question_id` ON DELETE
    // CASCADE through the questions, but `quiz_attempts.attempted_questions`
    // is a JSON array with no FK — re-seeding without explicitly wiping attempts
    // leaves them holding stale question IDs that fail the next submit with
    // an FK violation on `quiz_attempt_answers`. So we wipe attempts too.
    const qCount = await client.query(
      `SELECT COUNT(*)::int AS n FROM quiz_questions WHERE quiz_id = $1`,
      [quizId],
    );
    const aCount = await client.query(
      `SELECT COUNT(*)::int AS n
         FROM quiz_attempt_answers qaa
         JOIN quiz_questions q ON q.id = qaa.question_id
        WHERE q.quiz_id = $1`,
      [quizId],
    );
    const tCount = await client.query(
      `SELECT COUNT(*)::int AS n FROM quiz_attempts WHERE quiz_id = $1`,
      [quizId],
    );
    console.log(
      `  · quiz exists (id=${quizId}) — wiping ${qCount.rows[0].n} question(s),` +
      ` ${aCount.rows[0].n} answer row(s) (CASCADE), and ${tCount.rows[0].n} attempt(s)`,
    );
    await client.query(`DELETE FROM quiz_attempts WHERE quiz_id = $1`, [quizId]);
    await client.query(`DELETE FROM quiz_questions WHERE quiz_id = $1`, [quizId]);
    await client.query(
      `UPDATE quizzes
          SET questions_per_attempt = $1,
              passing_threshold = $2,
              is_required = $3,
              shuffle_questions = $4,
              time_limit_minutes = $5,
              updated_at = NOW()
        WHERE id = $6`,
      [QUIZ.questionsPerAttempt, QUIZ.passingThreshold, QUIZ.isRequired, QUIZ.shuffleQuestions, QUIZ.timeLimitMinutes, quizId],
    );
  } else {
    const inserted = await client.query(
      `INSERT INTO quizzes
         (chapter_id, position, questions_per_attempt, passing_threshold,
          is_required, shuffle_questions, time_limit_minutes)
       VALUES ($1, $2, $3, $4, $5, $6, $7)
       RETURNING id`,
      [
        chapterId, QUIZ.position, QUIZ.questionsPerAttempt, QUIZ.passingThreshold,
        QUIZ.isRequired, QUIZ.shuffleQuestions, QUIZ.timeLimitMinutes,
      ],
    );
    quizId = inserted.rows[0].id;
    console.log(`  · quiz inserted (id=${quizId})`);
  }

  for (const [lang, tr] of Object.entries(QUIZ.translations)) {
    await upsertTranslation(client, {
      table: "quiz_translations",
      entityCol: "quiz_id",
      entityId: quizId,
      lang,
      columns: { title: tr.title, description: tr.description },
    });
  }
  return quizId;
}

async function ensureSyllabusItem(client, chapterId, quizId) {
  const existing = await client.query(
    `SELECT id FROM chapter_syllabus_items WHERE quiz_id = $1`,
    [quizId],
  );
  if (existing.rowCount > 0) return;
  const maxPos = await client.query(
    `SELECT COALESCE(MAX(position), -1) AS max_pos
       FROM chapter_syllabus_items WHERE chapter_id = $1`,
    [chapterId],
  );
  await client.query(
    `INSERT INTO chapter_syllabus_items (chapter_id, item_type, quiz_id, position)
     VALUES ($1, 'quiz', $2, $3)`,
    [chapterId, quizId, Number(maxPos.rows[0].max_pos) + 1],
  );
}

async function insertQuestion(client, quizId, q, position) {
  const inserted = await client.query(
    `INSERT INTO quiz_questions (quiz_id, question_type, points, position)
     VALUES ($1, $2, $3, $4) RETURNING id`,
    [quizId, q.type, q.points, position],
  );
  const questionId = inserted.rows[0].id;

  for (const [lang, tr] of Object.entries(q.translations)) {
    await upsertTranslation(client, {
      table: "quiz_question_translations",
      entityCol: "question_id",
      entityId: questionId,
      lang,
      columns: { question_text: tr.text, explanation: tr.explanation ?? null },
    });
  }

  for (let i = 0; i < q.items.length; i++) {
    const it = q.items[i];
    const itemInsert = await client.query(
      `INSERT INTO quiz_question_items (question_id, role, position, is_mc_correct)
       VALUES ($1, $2, $3, $4) RETURNING id`,
      [questionId, it.role, i, it.isCorrect],
    );
    const itemId = itemInsert.rows[0].id;
    for (const [lang, tr] of Object.entries(it.translations)) {
      await upsertTranslation(client, {
        table: "quiz_question_item_translations",
        entityCol: "item_id",
        entityId: itemId,
        lang,
        columns: { primary_text: tr.primary ?? null, secondary_text: tr.secondary ?? null },
      });
    }
  }
}

/**
 * After wipe + reinsert, live `quiz_questions.id`s no longer match what's in
 * `classes.published_snapshot.quizzes[<quizId>]`, which is what the student
 * app reads. Leaving the stale snapshot entry in place causes the student
 * player to start attempts with non-existent IDs and crash on submit with an
 * FK error. Strip just this quiz from the snapshot so the student side
 * soft-fails ("퀴즈를 찾을 수 없습니다") until the admin republishes.
 */
async function invalidateSnapshotForQuiz(client, classId, quizId) {
  const result = await client.query(
    `UPDATE classes
        SET published_snapshot = jsonb_set(
              published_snapshot,
              '{quizzes}',
              COALESCE(published_snapshot->'quizzes', '{}'::jsonb) - $2::text
            )
      WHERE id = $1
        AND published_snapshot IS NOT NULL
        AND published_snapshot->'quizzes' ? $2::text
      RETURNING id`,
    [classId, String(quizId)],
  );
  return result.rowCount > 0;
}

async function main() {
  const client = await pool.connect();
  try {
    await client.query("BEGIN");
    const chapterId = await resolveChapter(client);
    const classId = (
      await client.query(`SELECT class_id FROM class_chapters WHERE id = $1`, [chapterId])
    ).rows[0].class_id;
    const quizId = await findOrCreateQuiz(client, chapterId);
    await ensureSyllabusItem(client, chapterId, quizId);
    for (let i = 0; i < QUESTIONS.length; i++) {
      await insertQuestion(client, quizId, QUESTIONS[i], i);
    }
    const snapshotInvalidated = await invalidateSnapshotForQuiz(client, classId, quizId);
    await client.query("COMMIT");
    console.log(`\n✔ quiz seeded · quiz_id=${quizId} · chapter_id=${chapterId} · ${QUESTIONS.length} question(s)`);
    if (snapshotInvalidated) {
      console.log(
        `\n⚠ published_snapshot for "${TARGET.courseSlug}" is now stale —` +
        ` the quiz has been REMOVED from the snapshot until you republish.` +
        `\n  Republish the course in the admin UI to make this quiz visible again.`,
      );
    }
  } catch (err) {
    await client.query("ROLLBACK");
    console.error(err);
    process.exit(1);
  } finally {
    client.release();
    await pool.end();
  }
}

main();
```

## Schema quick-card (relevant tables only)

| Table | Primary key | Columns in play |
|---|---|---|
| `quizzes` | `id` | `chapter_id`, `position`, `questions_per_attempt`, `passing_threshold`, `is_required`, `shuffle_questions`, `time_limit_minutes` |
| `quiz_translations` | `(quiz_id, language_code)` | `title`, `description` |
| `quiz_questions` | `id` | `quiz_id`, `question_type` (enum check), `points`, `position` |
| `quiz_question_translations` | `(question_id, language_code)` | `question_text`, `explanation` |
| `quiz_question_items` | `id` | `question_id`, `role` (enum check), `position`, `is_mc_correct` |
| `quiz_question_item_translations` | `(item_id, language_code)` | `primary_text`, `secondary_text` |
| `chapter_syllabus_items` | `id` | `chapter_id`, `item_type='quiz'`, `quiz_id`, `position` — required, one per quiz |

Check constraints enforced at the DB level:

- `quiz_questions.question_type IN ('multiple_choice','true_false','fill_blank','matching','ordering')`
- `quiz_question_items.role IN ('choice','blank','wordbank','match_pair','sequence')`
- `chapter_syllabus_items` has a unique index on `quiz_id` — one syllabus slot per quiz.

## Item-type rules (per type)

### multiple_choice

- Exactly **3 options** unless the question has a natural 4-option structure.
- **1 correct** (the quiz UI supports multi-correct; don't use it for attention checks — ambiguous to students).
- Stem carries the main idea — a reader should be able to answer before reading the options.
- All 3 options grammatically parallel, length within ±25%.
- Distractors sourced from the same transcript: another entity from the same category, a misconception the instructor corrects, a fact from a different beat of the lesson, or a close-unit numeric near-miss.
- **No synonym distractors** (rule 6). E.g., if the key is `피`, do not list `보혈` as a distractor — they're interchangeable in this context. Pick distractors from a different conceptual slot in the lesson, not a near-equivalent of the key.

### true_false

- Only use when the source statement is **categorically** true or false — no "usually / often / most".
- Exactly 2 `choice` items: `"참" / "True"` (one correct) and `"거짓" / "False"` (the other).
- No negation in the statement. No double negatives.
- No absolute determiners unless the source itself is absolute.

### fill_blank

- **One placeholder = one `role: "blank"` item.** Default to a single blank per question; multi-blank stems are allowed when the lesson yields multiple content words.
- **Pick the marker form by *grading rule*, not aesthetics** (rule #7):
  - `{{ blank }}` — **any-order grading** (multiset match). Use when the answers are a *set*. The publish step writes `fillBlankMode: "any_order"` on the snapshot and the runtime grades by multiset intersection.
  - `{{1}}` / `{{2}}` — **ordered grading** (per-slot). Use when the answers are a *sequence* tied to specific positions in the stem. Snapshot gets `fillBlankMode: "ordered"`.
  - For a single blank, the two forms grade identically — default to `{{ blank }}` for the seed-skill's visual consistency.
- **Never mix forms in one question.** The admin UI auto-converts on toggle and the substituter accepts both, but `buildQuizSnapshots` detects from the draft text *before* substitution, so a question is locked to one mode at publish time. Mixed text would silently corrupt grading.
- Delete a content word (noun / verb / number / proper noun), never a function word.
- Blank not inside the first sentence of the snippet — preserve context.
- **No synonym alternates.** Every `role: "blank"` row is a *separate required slot* in the runtime, not an alternate answer. If two synonyms feel equally valid for the same slot (`피` vs `보혈`, `회개` vs `회개하기`), rephrase the stem so only one fits — change the surrounding particle, narrow the cue, or pick a different content word from the transcript.
- **Word bank** (`role: "wordbank"`) is optional. Provide 3–5 wordbank entries only when the answer is a proper noun, number, or technical term the student couldn't plausibly spell unaided.
- **Wordbank distractors must be semantically distinct from the key** (rule 6). If the key is `피`, do not put `보혈` in the wordbank — but also do not put `보혈` in as a second `blank` row. Synonyms have no valid place in a fill_blank item; rephrase the stem instead.

#### Multi-blank examples

The main data-module example uses a single-blank `{{ blank }}` for brevity. For multi-blank questions, choose the form by grading rule:

**Any-order** (`{{ blank }}` × N) — the answers form a set; order doesn't matter. The student gets full credit if the bag of submitted words equals the bag of correct words.

```js
// [00:35:23] "the three altars Satan exploits are: ignorance, disobedience, and covenant."
{
  type: "fill_blank",
  points: 10,
  translations: {
    ko: {
      text: "강사가 언급한 세 가지 통로는 {{ blank }}, {{ blank }}, {{ blank }}이다.",
      explanation: "0:35:23에서 세 가지를 한꺼번에 나열했습니다 (순서 무관).",
    },
    en: {
      text: "The three grounds the speaker named are {{ blank }}, {{ blank }}, and {{ blank }}.",
      explanation: "Listed together at 00:35:23 — order is not part of the speaker's claim.",
    },
  },
  items: [
    // Three distinct blank items. Order in `items` matches order in `text`,
    // but at runtime grading is multiset — student can fill in any order.
    { role: "blank", isCorrect: true, translations: { ko: { primary: "무지" },     en: { primary: "ignorance" } } },
    { role: "blank", isCorrect: true, translations: { ko: { primary: "불순종" },   en: { primary: "disobedience" } } },
    { role: "blank", isCorrect: true, translations: { ko: { primary: "언약" },     en: { primary: "covenant" } } },
    // Wordbank distractors — pulled from elsewhere in the lesson, not synonyms of the keys.
    { role: "wordbank", isCorrect: false, translations: { ko: { primary: "보혈" }, en: { primary: "blood" } } },
    { role: "wordbank", isCorrect: false, translations: { ko: { primary: "회개" }, en: { primary: "repentance" } } },
  ],
},
```

**Ordered** (`{{1}}` / `{{2}}` / …) — each numbered slot has its specific answer. Surrounding context (Korean particles, grammar role, named position) fixes which word goes in which slot. Slot mismatch = wrong.

```js
// [00:18:20] "the three stages, in order, are: repentance first, then the blood, then the declaration."
{
  type: "fill_blank",
  points: 10,
  translations: {
    ko: {
      text: "악한 제단을 헐 때의 세 단계는 첫 번째 {{1}}, 두 번째 {{2}}, 세 번째 {{3}}이다.",
      explanation: "0:18:20에서 ‘회개 → 보혈 → 선포’ 순으로 명시했습니다 (순서 중요).",
    },
    en: {
      text: "The three stages for breaking an altar are first {{1}}, then {{2}}, then {{3}}.",
      explanation: "The speaker gives the order as repentance → the blood → the declaration (00:18:20).",
    },
  },
  items: [
    // Position N (0-indexed) ↔ {{N+1}} marker in text. The student must put
    // "회개" in slot {{1}}, "보혈" in slot {{2}}, "선포" in slot {{3}}.
    { role: "blank", isCorrect: true, translations: { ko: { primary: "회개" }, en: { primary: "repentance" } } },
    { role: "blank", isCorrect: true, translations: { ko: { primary: "보혈" }, en: { primary: "the blood" } } },
    { role: "blank", isCorrect: true, translations: { ko: { primary: "선포" }, en: { primary: "the declaration" } } },
    { role: "wordbank", isCorrect: false, translations: { ko: { primary: "금식" }, en: { primary: "fasting" } } },
    { role: "wordbank", isCorrect: false, translations: { ko: { primary: "묵상" }, en: { primary: "meditation" } } },
  ],
},
```

**Quick decision**: ask "if the student got every right word but in a different slot, should they get full credit?"
- Yes → `{{ blank }}` (any-order). The set is what matters.
- No → `{{1}}`/`{{2}}` (ordered). The position is part of the test.

### matching and ordering — not emitted

The DB supports `match_pair` and `sequence` item roles, and the admin UI can author them. But `apps/school/src/features/quiz/actions.ts` explicitly skips matching/ordering questions on the student side (see "Platform constraints" at the top of this file). Emitting these types would mean students silently skip them and are marked 0 even with a perfect understanding of the material. **Do not produce matching or ordering items.** If the transcript screams for a sequence, convert it into a `multiple_choice` — "Which step comes right after X?" — or drop it from the blueprint.

## Bilingual policy

- KO is required; EN is required for every item in a quiz (all five must have both locales — this is a content-complete quiz, not a draft).
- Author KO and EN independently — don't word-for-word translate. Each locale should read as native prose.
- No idioms, puns, cultural references, or double negatives in either locale. This is a cross-language equivalence rule from the International Test Commission guidelines.
- All string fields on the data module use the shape `translations: { ko: {...}, en: {...} }`. If a future locale is added to `packages/utils/src/i18n.ts`, drop a `jp: {...}` block into the same shape — executor iterates `Object.entries(...translations)` and writes whatever is present.

## Iterative edits

When the user asks to tweak questions after the first run, edit `seed-data/<slug>-ch<N>-quiz.data.mjs` and re-run the executor. Re-runs are wipe + reinsert — old question IDs are not preserved, and because `quiz_attempt_answers.question_id` has `ON DELETE CASCADE`, every student answer row on the current questions is destroyed.

For most iteration cycles this is fine — during authoring, nobody is taking the quiz yet. But once the course ships, re-seeding is a destructive content replacement, not a harmless edit. **Always run the pre-flight impact check (question count + attempt-answer count) and surface both numbers before re-running.** If `attempt_answers > 0`, stop and make the user say "wipe student data" explicitly — don't assume.

## When NOT to use this skill

- The course or chapter doesn't exist yet — use `seed-course-from-srt` first.
- The user wants more than a single post-lesson attention check (e.g., a proctored final exam, a placement test, a graded assignment). This skill is deliberately tuned for low-stakes comprehension.
- The user has already hand-authored the quiz in the admin UI and wants a specific tweak — tell them to edit it in the admin UI; re-running this skill will wipe their work.
- The lesson transcript is <5 minutes or doesn't contain enough distinct beats for 5 grounded items. Tell the user and ask if they want a shorter quiz (they'd need a different skill — this one is fixed at 5).

## Files this skill creates

Per quiz:

- `packages/database/scripts/seed-data/<course-slug>-ch<N>-quiz.data.mjs` (authored)
- `packages/database/scripts/seed-<course-slug>-ch<N>-quiz.mjs` (executor)

Both go into version control. Re-running weeks later is how content gets resynced after the data module changes.
