---
name: seed-course-from-srt
description: Seed a draft course into the Neon database from one or more SRT transcripts. Produces a bilingual KO+EN data module (title, description, duration, features, FAQs, chapters, lessons, instructor bio, TipTap detail_page_json using the numberedCards/bulletCard custom blocks in @repo/tiptap), per-lesson YouTube-style video chapters, per-lesson bible-verse markers, and an idempotent executor, then runs it. If the slug already exists, it switches to evaluate-and-feedback mode instead of clobbering. Use when the user drops SRT paths and says "fill up the course database", "create a course from this transcript", or anything equivalent.
user-invocable: true
argument-hint: "<srt-path> [additional-srt-paths...]"
---

# Seed a course from SRT transcripts

A one-shot, bilingual (KO + EN), draft-only course insert driven directly by the provided SRT transcripts. No guesswork content, no fabricated credentials. The canonical reference implementation is the `invisible-altars` pair already on disk:

- `packages/database/scripts/seed-data/invisible-altars.data.mjs` — authored KO + EN content + TipTap `detail_page_json`
- `packages/database/scripts/seed-invisible-altars.mjs` — idempotent executor, single transaction, preflight-guarded

Copy those as your template. Every new course gets the same two-file pair.

## Protocol (do these in order)

### 0. If the slug already exists, switch to evaluate-and-feedback

Before authoring, propose a slug from the SRT (lowercased, hyphenated, English-derived) and check existence:

```sql
SELECT id, status, published_at FROM classes WHERE slug = $1;
```

If the row exists, do **not** author and re-seed silently. Run the **Evaluate** branch (Section 9 below): pull the live data, derive what the new SRT pass would produce, present a structured diff, and ask the user which surfaces to update — not the script. The executor stays idempotent and oblivious; the *skill* reasons about the diff.

If no row exists, continue with Section 1.

### 1. Read the SRT(s) in full

You cannot author good course copy from a skim. Read every provided SRT top to bottom — both the original-language cues and any interpreted cues — before drafting. Identify:

- The teaching's **spine** — 5–8 load-bearing sections the speaker actually walks through
- **Key Scripture references** with their **SRT cue start time** — every one the speaker cites at chapter or chapter:verse precision (used in Section 3a). Book-only mentions ("the book of Romans") are dropped at parse time. Apocrypha (Tobit, Sirach, 1–4 Maccabees, Wisdom, Sirach, Baruch, Judith, etc.) and pseudepigrapha (Enoch, Jubilees, etc.) are dropped at parse time — they have no row in `bible_books`.
- **Natural section transitions** — speaker says "now let's move to…", "next", a long pause, an audience-prayer break, a Q&A pivot. Used to cut **video chapters** (Section 3b). Aim for 4–10 per lesson, YouTube-coarse, not lesson-outline-fine.
- **Live exercises / guided prayers / activities** the speaker leads the audience through
- **Q&A or follow-up sessions** — these map to their own chapter
- **Speaker credentials** that are *literally spoken in the transcript* — do not import credentials from placeholder mocks or your training data. If the transcript doesn't evidence a specific book, do not name a specific book.

### 2. Know where each piece of data goes — do not duplicate

Before you author `detail_page_json`, understand that every surface has a dedicated database column. **Do not put any of the following into `detail_page_json` — they render from their own columns:**

| Content | Dedicated home |
|---|---|
| Course title | `class_translations.title` |
| Short blurb | `class_translations.description` |
| Duration display | `class_translations.duration_text` |
| Language / format / level labels | `classes.language`, `classes.format`, `classes.level` (free-text, user-friendly KO) |
| Per-chapter recap | `class_chapter_translations.description` |
| Per-lesson outline / notes | `class_lessons.content_json` (TipTap, non-localized — put bilingual outline in one doc) |
| Instructor name / headline / bio | `instructor_translations.{name,headline,biography}` via `class_instructors` |
| FAQs | `class_faqs` + `class_faq_translations.{question,answer}` |
| Feature highlights (4 max, icon-constrained) | `class_features.icon` + `class_feature_translations.{label,description}` |
| Course hook (short TipTap) | `class_translations.introduction` |

`detail_page_json` is for content that has **no dedicated column** — narrative framing, enumerable outcomes, target audience, prerequisites, closing call. That's it.

**Cross-entity references use TipTap `link` marks — never embedded duplicate detail.** If prose in `detail_page_json` needs to name an instructor, another course, or any entity that has its own detail page, wrap the reference in a `link(text, href)` mark instead of inlining a description of it.

| Entity | Link target |
|---|---|
| Instructor | `/instructors/<slug>` (the route is named `[username]` but it resolves via the instructor slug) |
| Another course | `/courses/<slug>` |
| A specific lesson | `/courses/<slug>/learn/<lesson-id>` |

Concretely:

```js
p(
  t("In this two-session practicum "),
  link("Charlie Shamp", "/instructors/charlie-shamp"),
  t(" walks you step by step…"),
)
```

**Not:**

```js
p(
  t("In this two-session practicum Charlie Shamp (founder of Destiny Encounters International, …long bio…) walks you…"),
)
```

The instructor detail page reads `instructor_translations.biography`. The link is the handoff; duplicating the bio in prose creates two things to keep in sync.

### 3. Design the data module

Copy `packages/database/scripts/seed-data/invisible-altars.data.mjs` to `packages/database/scripts/seed-data/<slug>.data.mjs` and rewrite every authored string. Keep the TipTap helpers at the top (`t`, `p`, `h`, `bullet`, `ordered`, `quote`, `numberedCards`, `bulletCard`, `twoColumn`, `link`, `em`, `bold`) — they are generic.

**Rules**

- **KO and EN author independently.** Do not translate word-for-word. Each locale reads as native prose.
- **Translated fields are monolingual per locale.** `instructor_translations.{name,headline,biography}`, `class_translations.{title,description,...}`, `class_chapter_translations.*`, `class_feature_translations.*`, `class_faq_translations.*` — the KO row is Korean-only, the EN row is English-only. **Never concatenate the other locale's form as a parenthetical** (i.e., do NOT write `name: "벤 림 (Ben Lim)"` in the KO row — write `"벤 림"`; the EN row carries `"Ben Lim"` on its own). The row's `language_code` is the cross-locale handoff, not an inline gloss. This applies equally to `link()` marks inside TipTap prose — the KO document links `"벤 림 목사님"` → `/instructors/ben-lim`; the EN document links `"Ben Lim"` → `/instructors/ben-lim`. Proper nouns that are genuinely untranslated (registered English org names like `"Destiny Encounters International"` appearing inside Korean prose) are fine — the rule is *no redundant duplication of the same name in both scripts in the same string*.
- **Slug**: lowercase, hyphenated, English-derived, concise (e.g. `invisible-altars`, not `the-invisible-altars-teaching`).
- **Title** is a role-or-outcome + specific practice, not a topic label ("Hearing God in Intercession: A 6-Week Guided Path" beats "Intercession 101"). **Never include speaker names in the title** (not `"벤 림 · 척 피어스의 한국을 향한 줌예배"` — instructors render on the detail page from `class_instructors`; the title is for the *outcome*, not the roster). **Never include the delivery platform** (no "줌/Zoom/YouTube/온라인") — platform belongs on `classes.format` (e.g. `"온라인 Zoom 녹화 강의"`), not in the title. If the course only makes sense because of who is teaching it, link the instructor detail page from `introduction`; do not shortcut that by stuffing names into the title. — see Enrollment rule 2.
- **Description** is one sentence, under 22 words, naming *who it's for* and *what changes after* — no adjectives about the course itself. **No speaker names, no delivery-platform names** (Zoom / YouTube / 줌 / 온라인) — same rationale as the title: speakers live on `class_instructors`, platform lives on `classes.format`. The description is an outcome sentence, not a format sheet. — see Enrollment rule 3.
- **Base columns** (`classes.language`, `format`, `level`) carry **user-friendly KO strings** — e.g. `"영어 (한국어 통역 포함)"`, `"온라인 Zoom 녹화 강의"`, `"중급"`. These are rendered as-is to the student. There is no per-locale translation for these fields currently.
- **`duration_text`** is **total duration only** — just hours (or hours + minutes for short courses). Do **not** append session counts ("4개 세션"), cadence ("하루 20분 · 3주"), or any structural metadata. Chapter/lesson counts belong on the `class_features` book-icon card, not in `duration_text`. Examples — KO: `"약 4시간"`, `"약 8시간 30분"`. EN: `"~4 hours"`, `"~8 hours 30 minutes"`. Never raw minutes alone (`"240분"`). — see Enrollment rule 8.
- **Price defaults to `"0"`** (stored as a numeric-string: `price: "0"`). Do **not** set `original_price` and do **not** invent a price. Pricing is an admin decision made after seed — leave `price: "0"` and `original_price: null` unless the user explicitly names both a price and (optionally) a strike-through in the same turn. If a price is set in draft, the admin can adjust it in the UI; defaulting to `0` makes that absence obvious instead of smuggling a made-up number into production.
- **`class_features.icon`** must be one of `case | book | check | message`. The DB column is unconstrained `text` (no CHECK), but the student-side renderer only knows these four — anything else falls through to a default. Pick exactly 4, and use the icon contract: `case` = named outcome · `book` = scope (hours + chapter count) · `check` = tangible deliverable or practice · `message` = support / Q&A commitment. Never two outcomes. — see Enrollment rule 5.
- **FAQs** must map to real questions from the source (post-talk Q&A, recurring concerns in the transcript), not invented FAQs. Group as (a) fit/prerequisites, (b) format/time commitment, (c) refund + access duration. 5–8 pairs. — see Enrollment rule 14.
- **Introduction** (short TipTap, **exactly 3 paragraphs, PAS structure**): (1) the reader's current frustration in their words, (2) why prior approaches stall, (3) what this course does differently — close the third paragraph with a forward-leaning question, not a CTA. — see Enrollment rule 4.
- **Chapter descriptions** start with an action verb and state one capability the student leaves with — not a summary of contents. — see Enrollment rule 6.
- **Chapter + lesson titles are visible unpaywalled** on the detail page. Do not hide structure behind enrollment. — see Enrollment rule 7.
- **`detail_page_json` structure (the block palette to draw from):**
  - Opening: **1 `heading(2)` hook naming the audience → 1 paragraph promising the transformation → 1 paragraph of credibility (specific organization / role / year count, no adjectives).** This is the pre-scroll spine. No `blockquote`, no Scripture here.
  - `numberedCards` eyebrow `COURSE OUTCOMES` — 4–6 **sequenced** items. Reserve numberedCards for *ordered* content (a method in N steps, a path, a flow). Never for unordered benefits. — see Enrollment rule 9.
  - `bulletCard` variant `primary` eyebrow `WHO THIS IS FOR` — 4–6 outcome claims / audience descriptors.
  - `bulletCard` variant `muted` eyebrow `PREPARE` — 3–5 prerequisites, "who this is *not* for," or honest limitations. Negative qualification raises conversion. — see Enrollment rule 10.
  - Optional: one `twoColumn` for before/after contrast (left: current state · right: trained state), placed near the midpoint. At most one per page. — see Enrollment rule 11.
  - `horizontalRule`
  - Closing: one paragraph restating the reader's new identity ("become a…") + one single CTA. No discount flash, no urgency countdown. — see Enrollment rule 18.
  - **No** `h(1)` title, **no** instructor section, **no** per-session `twoColumn` session-flow block — all duplicate dedicated columns.
- **Scripture appears at most once on the entire detail page**, as a quiet anchor inside the `introduction` or the instructor bio. Never as a headline, never as proof for an outcome promise. — see Enrollment rule 16.
- **No guarantee language that conflates spiritual outcome with purchase.** "You *will* hear God", "breakthrough guaranteed" — remove. Frame the product as access to training and practice, not an outcome only God can deliver. — see Enrollment rule 17.
- **Lesson `content_json`** is non-localized — author it as one bilingual TipTap doc (KO header · EN header on the same heading line) with: session outline, key Scripture list, reflection prompt. KO first, then `·` separator, then EN.
- **Instructor bio** leads with one concrete, verifiable credential (named organization, published title, years leading X), then one sentence of teaching philosophy. No generic ministry adjectives. Restricted to what the SRT literally evidences plus uncontroversial public facts. No invented book titles, no invented affiliations. — see Enrollment rule 12.

### 3a. Author per-lesson video chapters (YouTube-style, must start at 0:00)

Every lesson in the data module gets a `videoChapters: []` array. These are the timeline markers Mux Player surfaces via `addChapters()`. Treat them like YouTube chapters, not a lesson outline:

- **First entry's `startSeconds` MUST be `0`** — the welcome/intro cap. The executor hard-fails if this is violated.
- **Coarse, not granular.** 4–10 chapters per lesson. Group by *movement*, not by every subtopic ("도입", "본론 — 제단의 비밀", "기도 시연", "Q&A"). The detail of the teaching belongs in the lesson `content_json`, not the chapter list.
- **`titleKo` is required, `titleEn` is optional.** KO ≤ 80 chars. Mirror the bilingual stance — KO and EN written natively, no parenthetical glosses.
- **Strictly ascending, unique `startSeconds`.** Validate as you author.
- **Schema cap:** 50 chapters per video.
- **Don't author `id`** — the executor derives a stable id (`vc-<slug>-<chapterPos>-<lessonPos>-<idx>`, with `chapterPos` included so ids never collide across chapters where `lessonPos` resets to 0) so re-runs don't churn ids and the publish-diff can tell rename vs. add+remove.
- **Use real SRT cue times.** Pick the cue where the topic shift starts, not a guesstimate. Round down to the cue's start second.

Shape:

```js
videoChapters: [
  { startSeconds: 0, titleKo: "도입과 환영", titleEn: "Welcome and Setup" },
  { startSeconds: 480, titleKo: "제단이란 무엇인가", titleEn: "What an Altar Is" },
  { startSeconds: 1820, titleKo: "기도 시연", titleEn: "Live Prayer" },
  // …
],
```

### 3b. Author per-lesson bible references (chapter-precision or finer)

Every lesson gets a `bibleReferences: []` array of timestamped scripture markers — every reference the speaker actually cites in the SRT, not invented ones. The student player surfaces these as overlay cards + side-panel jumps.

**Inclusion rules** — apply at SRT-parse time:

- **Must be at least chapter-level.** Accept `book + chapter` (whole chapter) or `book + chapter:verseStart[-verseEnd]` (single verse / verse range). **Drop book-only mentions** ("the book of Romans", "Paul writes to the Galatians") — there is nowhere to anchor the marker.
- **Drop apocrypha and pseudepigrapha.** Tobit, Sirach, Wisdom, 1–4 Maccabees, Baruch, Judith, the additions to Esther/Daniel, Enoch, Jubilees, etc. are not in `bible_books` and cannot be linked.
- **Resolve every book to its OSIS id.** Use the lookup logic in `apps/school-admin/src/features/lessons/bible-reference-utils.ts:buildBookLookup` — Korean abbreviations (`창`, `요`, `롬`, `고전`…) and English forms both map to canonical OSIS (`GEN`, `JHN`, `ROM`, `1CO`…). If a book name doesn't resolve, drop the line.
- **Whole-chapter shorthand** (`"Ps 23"`, `"시 23"`) → encode as `verseStart: 1, verseEnd: 176`. The student-side renderer collapses this to the chapter's natural verse count at fetch time. (Matches the bulk-import parser already in the admin UI.)
- **`startSeconds`** = the SRT cue's start where the verse is named or read aloud. If the verse is read across a span, anchor to the start of the read.
- **Strictly ascending, unique `startSeconds`.** Schema cap: 100 refs per video.
- **Don't author `id`** — the executor derives `br-<slug>-<chapterPos>-<lessonPos>-<idx>` (4-part for the same collision-avoidance reason as `videoChapters`).

Shape (no `id` field — executor adds it):

```js
bibleReferences: [
  { startSeconds: 720,  bookOsisId: "GEN", chapter: 22, verseStart: 1,  verseEnd: 14 },
  { startSeconds: 1840, bookOsisId: "ROM", chapter: 8,  verseStart: 28, verseEnd: 30 },
  { startSeconds: 3120, bookOsisId: "PSA", chapter: 23, verseStart: 1,  verseEnd: 176 }, // whole chapter
],
```

The executor will reject any `bookOsisId` that does not exist in `bible_books` — so a typo or an apocryphal book that slipped through manifests as a hard seed-time error, not a silent broken marker.

### 4. Enrollment conversion ruleset

The rules below govern *what drives a browser to enroll*. Each is followed by the source pattern it comes from. When a rule earlier in the Design section says "see Enrollment rule N," this is where N lives.

**Overarching principle — tease the outcome, don't teach it.** The detail page sells the teaching; it does not substitute for it. Name the transformation and the territory; do not publish the formula, the step list, the prayer script, or the answer key. A hesitant buyer should leave the page *more curious*, not *already-answered*. This governs rules 6, 9, 10, and 14 below — when in conflict, this principle wins.

- **Good** (`numberedCards` body): "사탄이 내 삶에 접근하는 세 가지 합법적 통로를 분별하고, 그 문을 하나씩 닫는 실제 절차를 배웁니다." — names the territory (three legal grounds), not the list.
- **Bad** (`numberedCards` body): "무지 · 불순종 · 언약. 원수가 내 삶에 어느 문을 통해 들어왔는지 정확히 짚어 내고…" — spells out the three items; the reader now has the framework.
- **Good** (chapter description): "악한 제단을 헐고 의의 제단을 세우는 3단계 실전 기도를 배웁니다." — tells the reader there are three steps.
- **Bad** (chapter description): "3단계 — 보혈 · 희생의 언약 · 선포 — 를 함께 배웁니다." — tells the reader what the three steps are.
- **FAQ calibration**: answer the *objection at 50–70% depth* — enough that the reader is relieved, not enough that the course is redundant. "네, 그 원리는 본 강의에서 구체적으로 다룹니다" + one-sentence orientation is often right.

If you catch yourself listing N named items on the landing page and those same items are the N-step framework the course teaches, cut the list. Name the territory; keep the answer inside the course.


1. Open the page with a named-audience hook, then a transformation promise, then one line of credibility — in that order, all above the first scroll (— CopyHackers Picture-Promise-Prove / 4Ps).
2. Write `title` as [role/outcome] + [specific practice], not a topic label ("Hearing God in Intercession: A 6-Week Guided Path" beats "Intercession 101"). Exclude **speaker names** and **delivery-platform names** (Zoom / YouTube / 줌 / 온라인) — the speaker's credibility carries through `class_instructors` + `introduction` link, and platform belongs on `classes.format`. A title that survives a speaker change or a platform change is the right abstraction (— Ruzuku sales-page blueprint; platform-neutral titling convention).
3. Write `description` in under 22 words as one sentence naming *who it's for* and *what changes after* — no adjectives about the course itself, and no speaker names or delivery-platform names (Zoom / YouTube / 줌 / 온라인). Those live on `class_instructors` and `classes.format`; description is outcome-only (— Joanna Wiebe value-prop formula).
4. Structure `introduction` as three paragraphs: (1) the reader's current frustration in their own words, (2) why prior approaches stall, (3) what this course does differently — end paragraph 3 with a forward-leaning question, not a CTA (— PAS formula, CopyHackers).
5. Fill `class_features` by icon contract: `case` = named outcome · `book` = curriculum scope (hours + chapter count) · `check` = tangible deliverable or practice · `message` = support / community commitment — one role per slot, never two outcomes (— platform constraint + Sensei LMS outcome-verb rule).
6. Every `class_chapter_translations.description` starts with an action verb and states one capability the student leaves the chapter with — not a summary of contents (— Thinkific "verb-first, 5–8 outcomes" guidance).
7. Show all chapter titles and at least first-lesson titles unhidden on the detail page; do not gate structure behind enrollment (— Inflearn pattern: 60 sections fully visible).
8. Put `duration_text` as **total duration only** — hours (or hours + minutes), never raw minutes alone, and never appended with session counts or cadence. Structure metadata (session count, chapter count) lives on the `class_features` book-icon card, not here (— FastCampus course-metadata convention, minus the cadence overlay which the student-side UI does not render).
9. Use `numberedCards` for a sequenced "how it works" or "the method in N steps" block placed after introduction — reserve for ordered content, never unordered benefits (— platform constraint; ordered beats bulleted for process).
10. Use `bulletCard` primary for 4–6 outcome claims; `bulletCard` muted for prerequisites, "who this is *not* for," and honest limitations — negative qualification raises conversion (— Boagworld objection-handling).
11. Use `twoColumn` for before/after contrast (left: current state · right: trained state) — at most one per page, deployed once near the midpoint (— Before-After-Bridge formula).
12. In `instructor_translations.biography`, lead with one concrete credential a reader can verify (organization name, published title, years leading X), then one sentence of teaching philosophy — skip generic ministry adjectives (— Maven credibility guide; eSEOspace E-E-A-T).
13. When testimonials exist, the first must name a concrete before/after moment with specific context — city, role, or named situation — not generic "life-changing" language; 3–5 total, not more (— Spiegel 270% / LifePoint authenticity pattern).
14. Write `class_faqs` as 5–8 pairs grouped (a) fit / prerequisites, (b) format / time commitment, (c) refund + access duration — lead with the top three objections a hesitant buyer already has, not trivia (— Thrive Themes / Conversion Crimes).
15. State access duration, refund window, and "who should not buy this" plainly in the FAQ — no guarantee theatre, no urgency countdowns (— Kajabi refund-policy guidance; off-brand for ministry).
16. Deploy Scripture at most once on the entire page, as a quiet anchor inside `introduction` or instructor bio — never as proof for an outcome promise, never as a headline (— BSSM pattern: zero direct Scripture on landing).
17. Avoid guarantee language that conflates spiritual outcome with purchase ("you *will* hear God," "breakthrough guaranteed") — frame promises as *access to training and practice*, not outcomes only God can deliver (— ministry-ethics constraint; Leviticus 19:35 framing).
18. Close the long-form body with a restated identity line (who the reader becomes) + a single CTA — price anchoring via value reframing, not strikethroughs (— diverges from FastCampus/VOD norms; aligns with BSSM aspirational close).

#### Copy-module mapping

| Content type | Primary field(s) |
|---|---|
| Hook + promise | `title`, `description`, `introduction` (¶1) |
| Outcomes promised | `class_features` (case icon), `bulletCard` primary, chapter descriptions |
| Method / sequence | `numberedCards`, `twoColumn` (before→after) |
| Structure & scope | `duration_text`, `class_features` (book icon), `class_chapters`, `class_lessons` |
| Credibility & proof | `instructor_translations.biography`, testimonial blocks (if any) in `detail_page_json` |
| Objections & fit | `class_faqs`, `bulletCard` muted (prerequisites / not-for-you) |

#### Korean-vs-English divergences

- **Social-proof substitution.** KO pages lean on enrollment counts, instructor-authority framing (일타 / 대표강사 language), and platform trust marks rather than testimonial walls. EN version should restore 3–5 named testimonials with city/role context (— FastCampus vs. Senja pattern).
- **Pricing language.** KO readers expect explicit value framing (hours, chapter count, 평생 소장 permanence). EN should swap the strikethrough/installment pattern for one value-per-dollar line — aggressive discounting is off-brand in this niche (— task constraint + FastCampus divergence).
- **Tone register.** KO can be tighter, more declarative, and section-header-driven (scannable long-scroll). EN should be slightly more narrative and identity-forward ("become a…") to match Western ministry register (— BSSM landing pattern).
- **FAQ framing.** KO FAQs skew logistical (access, device, certificate). EN FAQs should add 1–2 theological-fit questions ("is this for Pentecostal/Reformed readers?") that KO audiences resolve via instructor brand recognition (— niche convention).

### 5. Handle null-by-design fields honestly

You cannot seed the following — tell the user they're admin-UI follow-ups:

- `classes.thumbnail_id` (needs CDN image upload via `uploadImageAndCreateRecord`)
- `class_lessons.mux_video_id` (needs Mux asset creation via the admin lesson dialog)

The course lands in `status='draft'` and is not publishable until a thumbnail is attached. Say this in the end-of-run message.

**Note on video chapters / bible refs:** these *are* authored at seed time (Sections 3a/3b) and live in the data module from day one — but they ride on the lesson's `mux_video_id`, which is null until the admin uploads each session's video. The executor's `applyVideoMarkers()` block (Section 6) handles this gracefully:

- On the **first run** (videos not yet uploaded), markers are kept in the data module and skipped on the DB side with a friendly per-lesson "no mux video yet — re-run after upload" log line.
- After the admin uploads videos, **re-run the same seed script**. Idempotent upserts no-op on prior rows; the marker block now finds `mux_video_id` set and writes `chapters` + `bible_references` JSONB onto each `mux_videos` row.
- Re-publishing the course refreshes `published_snapshot.chapters[].lessons[].videoChapters` / `bibleReferences` for the student side.

### 6. Write the executor script

Copy `seed-invisible-altars.mjs` to `seed-<slug>.mjs` and update the import path. The base script pattern is:

- Single `pg.Pool` + transaction
- Reads `DATABASE_URL` from `apps/school/.env` (same as `backup-and-clear-courses.mjs`)
- **Preflight**: verifies `class_translations.{detail_page_json,introduction}`, `class_lessons.content_json`, `instructor_translations.biography`, **`mux_videos.chapters`**, **`mux_videos.bible_references`** exist — hard-fails with a missing-migration message if any are absent (all defined in `0001_initial.ts`; if a column is missing, the migrations haven't been run)
- **Idempotent**: SELECT-then-insert keyed by slug (instructors, classes), by `position` (features, FAQs, chapters, lessons), and upsert on translations keyed by `(entity_id, language_code)`
- Re-syncs base columns on the classes row on every run, but **preserves** non-draft `status` (uses `COALESCE(NULLIF(status, 'draft'), $desired)` so a published course isn't flipped back to draft)
- Writes a `chapter_syllabus_items` row per lesson (item_type `'lesson'`)
- Writes a `class_instructors` link row (skip if exists)

**Plus, the new marker-apply block.** The reference `seed-invisible-altars.mjs` predates this — when you copy it, you MUST add the snippet below. Call it from the lesson loop right after `chapter_syllabus_items` is written, passing the parent course slug + the chapter and lesson positions so ids are deterministic.

```js
// applyVideoMarkers — paste once near the bottom of seed-<slug>.mjs.
//
// Writes mux_videos.chapters + mux_videos.bible_references for whichever
// mux_video the lesson is currently linked to. Silently skips when the lesson
// has not been uploaded yet (first-run case); re-running the seed after the
// admin uploads videos picks them up.
//
// Clobber rules — the admin UI also writes these columns, so the seed must
// not destroy admin work:
//   * `lesson.videoChapters` / `lesson.bibleReferences` is **a non-empty
//     array** → write it (full replace of that column).
//   * the field is **absent / undefined** → leave the column alone.
//   * the field is **explicitly `null`** → clear the column to NULL. Use this
//     and only this when the data module deliberately wants to wipe markers.
//   * the field is **`[]`** → treated the same as absent (do nothing). The
//     data module is *not* the source of truth for "no markers" — that's the
//     admin UI's job.
//
// Validation mirrors the server actions in
// apps/school-admin/src/features/lessons/actions.ts where they overlap, with
// one extra rule the seed enforces but the action does not:
//   - first chapter must start at 0:00 (skill-side invariant only;
//     apps/school-admin/src/features/lessons/actions.ts does NOT enforce this,
//     so the seed is stricter than the admin UI by design — drop the buffered
//     0:00 entry and the executor will reject the data file)
//   - chapters strictly ascending, unique startSeconds, < duration_seconds
//   - bible bookOsisId must exist in bible_books
//   - all refs strictly ascending, unique startSeconds, < duration_seconds
async function applyVideoMarkers(client, { courseSlug, chapterPosition, lessonPosition, lessonId, videoChapters, bibleReferences }) {
  const linked = await client.query(
    `SELECT l.mux_video_id, mv.duration_seconds, mv.status
       FROM class_lessons l
       LEFT JOIN mux_videos mv ON mv.id = l.mux_video_id
      WHERE l.id = $1`,
    [lessonId],
  );
  const muxVideoId = linked.rows[0]?.mux_video_id ?? null;
  if (!muxVideoId) {
    const buffered =
      (Array.isArray(videoChapters) && videoChapters.length > 0) ||
      (Array.isArray(bibleReferences) && bibleReferences.length > 0);
    if (buffered) {
      console.log(
        `    · ch${chapterPosition}/lesson${lessonPosition}: no mux video yet — markers buffered in data module; re-run after upload`,
      );
    }
    return;
  }
  const durationSeconds = Number(linked.rows[0]?.duration_seconds ?? 0);

  // --- Video chapters ---
  if (videoChapters === null) {
    await client.query(
      `UPDATE mux_videos SET chapters = NULL, updated_at = NOW() WHERE id = $1`,
      [muxVideoId],
    );
  } else if (Array.isArray(videoChapters) && videoChapters.length > 0) {
    if (videoChapters[0].startSeconds !== 0) {
      throw new Error(
        `ch${chapterPosition}/lesson${lessonPosition}: first video chapter must start at 0:00`,
      );
    }
    const sortedC = [...videoChapters].sort((a, b) => a.startSeconds - b.startSeconds);
    for (let i = 0; i < sortedC.length; i += 1) {
      const c = sortedC[i];
      if (durationSeconds && c.startSeconds >= durationSeconds) {
        throw new Error(
          `ch${chapterPosition}/lesson${lessonPosition}: chapter[${i}] startSeconds=${c.startSeconds} ≥ duration=${durationSeconds}`,
        );
      }
      if (i > 0 && c.startSeconds === sortedC[i - 1].startSeconds) {
        throw new Error(
          `ch${chapterPosition}/lesson${lessonPosition}: duplicate chapter startSeconds=${c.startSeconds}`,
        );
      }
    }
    const chaptersJson = sortedC.map((c, i) => ({
      id: `vc-${courseSlug}-${chapterPosition}-${lessonPosition}-${i}`,
      startSeconds: c.startSeconds,
      titleKo: c.titleKo,
      titleEn: c.titleEn ?? null,
    }));
    await client.query(
      `UPDATE mux_videos SET chapters = $1, updated_at = NOW() WHERE id = $2`,
      [JSON.stringify(chaptersJson), muxVideoId],
    );
  }
  // else: absent or [] → leave the column alone (admin may have authored markers)

  // --- Bible references ---
  if (bibleReferences === null) {
    await client.query(
      `UPDATE mux_videos SET bible_references = NULL, updated_at = NOW() WHERE id = $1`,
      [muxVideoId],
    );
  } else if (Array.isArray(bibleReferences) && bibleReferences.length > 0) {
    const uniqueBooks = [...new Set(bibleReferences.map((r) => r.bookOsisId))];
    const known = await client.query(
      `SELECT DISTINCT osis_id FROM bible_books WHERE osis_id = ANY($1::text[])`,
      [uniqueBooks],
    );
    const knownSet = new Set(known.rows.map((r) => r.osis_id));
    const missing = uniqueBooks.filter((b) => !knownSet.has(b));
    if (missing.length > 0) {
      throw new Error(
        `ch${chapterPosition}/lesson${lessonPosition}: unknown OSIS book(s): ${missing.join(", ")} (apocrypha / pseudepigrapha or typo — drop at parse time)`,
      );
    }
    const sortedR = [...bibleReferences].sort((a, b) => a.startSeconds - b.startSeconds);
    for (let i = 0; i < sortedR.length; i += 1) {
      const r = sortedR[i];
      if (durationSeconds && r.startSeconds >= durationSeconds) {
        throw new Error(
          `ch${chapterPosition}/lesson${lessonPosition}: ref[${i}] startSeconds=${r.startSeconds} ≥ duration=${durationSeconds}`,
        );
      }
      if (i > 0 && r.startSeconds === sortedR[i - 1].startSeconds) {
        throw new Error(
          `ch${chapterPosition}/lesson${lessonPosition}: duplicate ref startSeconds=${r.startSeconds}`,
        );
      }
      if (r.verseEnd < r.verseStart) {
        throw new Error(
          `ch${chapterPosition}/lesson${lessonPosition}: ref[${i}] verseEnd=${r.verseEnd} < verseStart=${r.verseStart}`,
        );
      }
    }
    const refsJson = sortedR.map((r, i) => ({
      id: `br-${courseSlug}-${chapterPosition}-${lessonPosition}-${i}`,
      startSeconds: r.startSeconds,
      bookOsisId: r.bookOsisId,
      chapter: r.chapter,
      verseStart: r.verseStart,
      verseEnd: r.verseEnd,
    }));
    await client.query(
      `UPDATE mux_videos SET bible_references = $1, updated_at = NOW() WHERE id = $2`,
      [JSON.stringify(refsJson), muxVideoId],
    );
  }
  // else: absent or [] → leave the column alone (admin may have authored refs)
}
```

End-of-run summary should additionally:

- Count how many lessons had markers applied vs. buffered (no `mux_video_id` yet).
- For applied bible refs, surface which translations are present (`SELECT DISTINCT translation_id FROM bible_verses WHERE book_osis_id = ANY($1::text[])` joined back to `bible_translations.code`) so a missing translation (e.g. only KRV is loaded but EN students will see empty side-panels) shows up in the report.

### 7. Gate on the advisor — score must be ≥ 8

Before presenting to the user, call `advisor()` once the data module and executor are durable on disk. If the score is below 8, fix what the advisor flags and call again. Only present to the user once you're at 8 or above.

### 8. Present what's going in, then run

Per the user's meta-instruction, after the advisor clears you:

1. Tell the user what data is about to be inserted — a scannable shortlist: KO + EN titles, the **KO + EN instructor name pair** (explicitly shown side by side so a parenthetical-contamination like `"벤 림 (Ben Lim)"` is caught before the insert), the **price** (should be `0` by default — flag prominently if you are inserting anything else so the user can catch it), the 4–6 `numberedCards` outcome headings, the FAQ questions, the 4 feature labels, chapter titles, lesson titles, **per-lesson video chapter count + first chapter title** (so the 0:00 entry is visible), **per-lesson bible-ref count + a 3-item sample**. Not the full strings — the things a content owner would scan for errors.
2. Run `node packages/database/scripts/seed-<slug>.mjs` in the same turn. Treat "then begin" / "now do it" / "go" as authorization, not a prompt for another confirmation.
3. Report the resulting class + instructor ids, the marker-apply summary (applied vs. buffered counts, available bible translation codes), and the admin-UI next steps (thumbnail → N × session video upload → re-run this seed to attach markers → publish).

### 9. Evaluate-existing branch (when Section 0 found a row)

If Section 0 detected an existing class with the target slug, do **not** author a fresh data module. Instead:

1. **Pull the live snapshot** from the DB:
   - `classes` row + both `class_translations` rows
   - `class_features` + `class_feature_translations`
   - `class_faqs` + `class_faq_translations`
   - `class_chapters` + `class_chapter_translations`
   - `class_lessons` + `class_lesson_translations`
   - `class_instructors` + `instructors` + `instructor_translations`
   - `mux_videos.chapters` and `mux_videos.bible_references` (joined via `class_lessons.mux_video_id`)
   - `classes.status`, `classes.published_at` — surface "this is published" prominently in the report
2. **Re-derive what the new SRT pass would produce** in memory only — do not write to disk yet. Run the same parse-and-extract logic as Sections 3–3b.
3. **Build a structured diff** and present it to the user. For each surface, list:
   - **No change** (skip in the report unless surface is empty in DB)
   - **Add** (live row missing)
   - **Update** (live row present, value differs — show old → new, max ~120 chars per side)
   - **Remove** (live row present, no analogue in new pass)
   Diff *every* surface: KO/EN titles & descriptions, `duration_text`, `language`/`format`/`level`, `introduction`, `detail_page_json` (shallow — just the top-level block sequence), 4 features, FAQs, chapter titles + descriptions, lesson titles + content_json, instructor bio, **per-lesson `videoChapters`**, **per-lesson `bibleReferences`**.
4. **Ask the user what to do.** Default options to offer (let them pick subsets, not all-or-nothing):
   - "Just refresh the markers (chapters + bible refs) — leave content alone."
   - "Refresh chapters only / bible refs only."
   - "Replace specific surfaces (features / FAQs / instructor bio / detail_page / chapter N / lesson N)."
   - "Nothing — abort."
5. **Then write only the slices the user picked.** Either patch the existing data module (if one is on disk for this slug) or, if the existing course was authored manually via the admin UI and there's no data module, write a fresh `seed-data/<slug>.data.mjs` containing only the picked slices and an executor that *only touches those tables*.
6. **Surface published-snapshot impact.** If `classes.status='published'`, remind the user that `published_snapshot` is frozen until the admin re-publishes — DB writes will not reach students until then. If they want students to see the change, they must re-publish via the admin UI after the seed.

The executor for the evaluate branch is still idempotent — a re-run of a partial seed-script is safe; it just no-ops on the surfaces you didn't include.

## Iterative edits

When the user asks to tweak content after the first run (rename fields, move copy between surfaces, remove duplicates, refresh markers after a video upload), edit `seed-data/<slug>.data.mjs` and re-run `node packages/database/scripts/seed-<slug>.mjs`. The script is idempotent — a re-run is safe and is also how marker buffering catches up after admin video uploads.

Two specific patterns that came up on the reference course and will come up again:

- **"Put the curriculum copy in the chapter, not on the detail page"** → move the per-session paragraphs from any `twoColumn` "How the sessions flow" block into `CHAPTERS[i].translations.{ko,en}.description`, delete the heading + `twoColumn` from both locales of `detail_page_json`.
- **"All fields should be user-friendly"** for `language`/`format`/`level` → the user wants KO display text directly in those base columns, not enum codes. e.g. `"영어 (한국어 통역 포함)"`, `"온라인 Zoom 녹화 강의"`, `"중급"`.

## When NOT to use this skill

- The user wants to *upload the video* → that's Mux + the admin UI flow, not this skill. (But once videos are uploaded, **do** re-run the seed script — that's how the buffered video chapters and bible references attach to the new `mux_videos` rows.)
- The user wants to publish the course → they do that in the admin UI after thumbnail upload. Never flip `status` to `published` from a seed script.
- The course already exists and the user just wants to read its current shape (no edits planned) → use the admin UI / a `SELECT` query; don't run the skill at all. The evaluate branch (Section 9) is for when they want to *act on* a diff.

## Files this skill creates

Per course (first run, no existing slug):
- `packages/database/scripts/seed-data/<slug>.data.mjs` (authored — includes per-lesson `videoChapters` and `bibleReferences`)
- `packages/database/scripts/seed-<slug>.mjs` (executor — includes `applyVideoMarkers`)

Per evaluate-branch run (existing slug, user picked specific surfaces): a partial data module + executor scoped to only the picked surfaces.

Both go into version control. Re-running is how you resync draft content after the data module changes — and how buffered markers attach once the admin uploads each lesson's Mux video.
