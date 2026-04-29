---
name: seed-bible-translation
description: Seed a bible translation into the Neon database from a source file in any common bible format (OSIS XML, USFM, Zefania XML, scrollmapper JSON, generic JSON, CSV/TSV, or SQLite). Detects the format from the file, normalizes book identifiers to OSIS, and produces an idempotent two-file seed script (data + executor) that upserts into `bible_translations`, `bible_books`, and `bible_verses`. Use when the user provides a bible file and says "seed this bible", "import this translation", "load KRV from this file", or anything equivalent.
user-invocable: true
argument-hint: "<bible-file-path> [--code <CODE>] [--lang <ko|en>] [--name <Display Name>] [--keep-markup]"
---

# Seed a bible translation into the local OSIS tables

A one-shot importer that ingests a bible file in any common format and populates `bible_translations` + `bible_books` + `bible_verses`. The output is two files (data + executor) following the same pattern as `seed-quiz-from-srt` and `seed-course-from-srt`, so re-running is idempotent and the user can review the data file before the executor touches the DB.

## Why this skill exists

The bible domain is already in the schema (`packages/database/migrations/0001_initial.ts`, lines ~1450–1511), keyed on canonical OSIS ids (e.g. `JHN.3.16`). The student-facing `BibleTab` and the lesson-marker overlay both read from these tables. But seeding is fiddly: every translation comes in a different format, book identifiers vary (`John` vs `JHN` vs `43`), and verse counts must be validated before the run. This skill encapsulates that.

## What it produces

Two files per invocation:

- `packages/database/scripts/seed-data/<code>.bible.data.mjs` — flat list of `{translation, books[], verses[]}` arrays, ready to insert. Plain data, no DB calls.
- `packages/database/scripts/seed-<code>-bible.mjs` — idempotent executor, **single transaction**: upserts the translation row, upserts the book rows, wipes existing verses for this `translation_id`, then bulk-inserts verses in 1000-row batches. The translation/book upserts, the verse delete, and **every** verse-insert batch all run inside one `db.transaction().execute(...)` call so a partial failure rolls back to the prior state instead of leaving a half-seeded translation.

Re-running:
- The translation row is found by composite `(code, language_code)` and either updated in place or left alone if unchanged.
- Book rows are upserted on `(osis_id, language_code)` — idempotent because the OSIS id is canonical and the localized name is stable per language.
- Verse rows are deleted-then-reinserted for the matched `translation_id` (verses are immutable, but a re-run with a corrected source file should overwrite cleanly).

Both files run via `node` (the project uses `.mjs` for seeds, not `.ts`, to avoid a tsx round-trip):

```bash
node --env-file=apps/school/.env packages/database/scripts/seed-<code>-bible.mjs
```

## Supported source formats

The skill probes the file (extension + first ~10 KB of content) and selects a parser. **You must confirm the detected format with the user before generating the executor** — a mis-detected format leads to silently empty inserts.

| Format | Probe signal | Notes |
|--------|--------------|-------|
| **OSIS XML** | `<osis>`, `<osisText>`, `osisID="JHN.3.16"` attributes | Cleanest source; book ids already OSIS. |
| **Zefania XML** | `<XMLBIBLE>`, `<BIBLEBOOK bnumber="…">` | Books are numbered 1..66; map via the `BNUM_TO_OSIS` table below. |
| **USFM** (plain text) | `\id JHN`, `\c 3`, `\v 16 …` markers | Often one file per book; iterate the directory. Strip `\f...\f*` footnote markers. |
| **scrollmapper JSON** | `[{ "name": "Genesis", "chapters": [[ "v1", "v2" ]] }]` | Common open-source format. Book name → OSIS via `BOOK_NAME_TO_OSIS`. |
| **Generic JSON** (rows) | `[{ "book": "John", "chapter": 3, "verse": 16, "text": "…" }]` | Most flexible; ask user which key holds the OSIS id if ambiguous. |
| **CSV / TSV** | `.csv` / `.tsv` extension | Header row required. Same column expectations as generic JSON. |
| **SQLite** | `.db` / `.sqlite` magic bytes | `bible_databases` (scrollmapper) ships these. Dump with `sqlite3 file.db ".dump verses"` — easier than parsing inside Node. |

If the file format isn't in the table above, **stop and ask the user** rather than guessing.

## Protocol (do these in order)

### 1. Preflight — hard-fail, never infer

- `<bible-file-path>` exists and is readable. If not, print the path and exit.
- `bible_translations`, `bible_books`, `bible_verses` tables exist (run a quick `SELECT 1 FROM bible_translations LIMIT 1` before drafting). If they don't, `0001_initial.ts` hasn't run yet — surface that and exit.
- `--code` argument or auto-detected from file. The code must be **unique per language** (`(code, language_code)` is unique). If `KRV` already exists for `ko`, ask whether to *update* or pick a different code.

### 2. Probe and confirm

Read the first ~10 KB of the file. Use the table above to detect the format. Then surface a single-message summary to the user and **wait for confirmation** before generating anything:

```
Detected: Zefania XML
Translation code: KRV (override with --code)
Language: ko (override with --lang)
Display name: 개역한글 (override with --name)
License: <ask user — required>
Books found: 66
Verses found: 31102
Markup handling: STRIP (footnotes, italics, cross-refs removed) — pass --keep-markup to preserve verbatim
Sample verse [JHN.3.16] (after strip): "하나님이 세상을 이처럼 사랑하사…"
Sample verse [JHN.3.16] (raw):       "하나님이 세상을 이처럼 사랑하사<NOTE>…</NOTE>"

Proceed? (y/n)
```

Show the raw side-by-side with the stripped side so the user can confirm the strip didn't mangle anything. If the raw and stripped are identical, omit the "raw" line.

If any of those four settings are wrong, the user corrects them in their reply and you re-confirm. Do not generate the data file until the user says yes.

### 3. Build the data file

Walk the source and emit `seed-data/<code>.bible.data.mjs`:

```js
// AUTO-GENERATED by seed-bible-translation skill
export const TRANSLATION = {
  code: "KRV",
  languageCode: "ko",
  name: "개역한글",
  attribution: "Public domain (1956 Korean Revised Version)",
  licenseNotes: null,
};

// One row per (osis_id, language_code). The skill emits the books in OSIS
// order (Genesis → Revelation) so re-runs are deterministic.
export const BOOKS = [
  { osisId: "GEN", languageCode: "ko", name: "창세기", position: 1 },
  // …
  { osisId: "REV", languageCode: "ko", name: "요한계시록", position: 66 },
];

// (book_osis_id, chapter, verse, text). One row per verse. The executor
// batches these in chunks of 1000 to keep the WAL small.
export const VERSES = [
  { bookOsisId: "GEN", chapter: 1, verse: 1, text: "태초에 하나님이…" },
  // …
];
```

**Hard rules for the data file:**

- Every `bookOsisId` MUST be a valid 3-letter OSIS code. Use the embedded `BOOK_NAME_TO_OSIS` map below to normalize. If a source name doesn't map, **fail loudly** — don't silently skip the book.
- Verses must be sorted by `(book position, chapter, verse)`. The student-side renderer relies on this order.
- **Strip rich-text markup and footnotes by default** — see the "Text normalization" section below for the exact strip rules per format. The reading panel is narrow and footnote markers + italics are visually noisy. Pass `--keep-markup` to preserve the raw source verbatim instead.

### 4. Build the executor

`scripts/seed-<code>-bible.mjs`. **Every DB write must run inside a single `db.transaction().execute(...)` block** — translation upsert, book upsert, verse wipe, and all verse-insert batches. Do not split into multiple transactions or run any write on the bare `db` handle. If the transaction fails or the process is killed mid-run, the prior translation state must be intact.

```js
import { Kysely, PostgresDialect } from "kysely";
import { Pool } from "pg";
import { TRANSLATION, BOOKS, VERSES } from "./seed-data/<code>.bible.data.mjs";

const VERSE_BATCH = 1000;

async function main() {
  const db = new Kysely({
    dialect: new PostgresDialect({
      pool: new Pool({ connectionString: process.env.DATABASE_URL }),
    }),
  });

  await db.transaction().execute(async (trx) => {
    // 1. Upsert the translation row.
    const existing = await trx
      .selectFrom("bible_translations")
      .select("id")
      .where("code", "=", TRANSLATION.code)
      .where("language_code", "=", TRANSLATION.languageCode)
      .executeTakeFirst();
    let translationId;
    if (existing) {
      await trx
        .updateTable("bible_translations")
        .set({
          name: TRANSLATION.name,
          attribution: TRANSLATION.attribution,
          license_notes: TRANSLATION.licenseNotes,
        })
        .where("id", "=", existing.id)
        .execute();
      translationId = existing.id;
    } else {
      const inserted = await trx
        .insertInto("bible_translations")
        .values({
          code: TRANSLATION.code,
          language_code: TRANSLATION.languageCode,
          name: TRANSLATION.name,
          attribution: TRANSLATION.attribution,
          license_notes: TRANSLATION.licenseNotes,
        })
        .returning("id")
        .executeTakeFirstOrThrow();
      translationId = inserted.id;
    }

    // 2. Upsert book rows. ON CONFLICT updates the localized name + position.
    if (BOOKS.length > 0) {
      await trx
        .insertInto("bible_books")
        .values(BOOKS.map((b) => ({
          osis_id: b.osisId,
          language_code: b.languageCode,
          name: b.name,
          position: b.position,
        })))
        .onConflict((oc) =>
          oc.columns(["osis_id", "language_code"]).doUpdateSet({
            name: (eb) => eb.ref("excluded.name"),
            position: (eb) => eb.ref("excluded.position"),
          }),
        )
        .execute();
    }

    // 3. Wipe + reinsert verses for this translation. Wipe is scoped to
    //    `translation_id` so other translations stay intact.
    await trx
      .deleteFrom("bible_verses")
      .where("translation_id", "=", translationId)
      .execute();

    for (let i = 0; i < VERSES.length; i += VERSE_BATCH) {
      const chunk = VERSES.slice(i, i + VERSE_BATCH);
      await trx
        .insertInto("bible_verses")
        .values(chunk.map((v) => ({
          translation_id: translationId,
          book_osis_id: v.bookOsisId,
          chapter: v.chapter,
          verse: v.verse,
          text: v.text,
        })))
        .execute();
    }
  });

  console.log(
    `Seeded ${TRANSLATION.code} (${TRANSLATION.languageCode}): ${BOOKS.length} books, ${VERSES.length} verses.`,
  );
  await db.destroy();
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

### 5. Preflight before running the executor

Before the user runs `node scripts/seed-…`, surface this in the chat:

```
Ready to seed:
  - 1 translation row (KRV / ko / 개역한글)
  - 66 book rows (osis_id × ko)
  - 31102 verse rows (delete + reinsert for translation_id matching KRV/ko)

Existing rows that will be touched:
  - bible_verses: <N> rows currently exist for KRV/ko — these will be wiped before reinsert.

Run? (y/n)
```

If `<N>` is unexpectedly large or the source row count is way off the published verse count for that translation (~31,102 for a Protestant canon), pause and ask the user to verify before running.

### 6. Run + verify

After the user confirms, run the executor and then a verification query:

```bash
node --env-file=apps/school/.env packages/database/scripts/seed-<code>-bible.mjs
```

Then verify:

```sql
SELECT
  bt.code, bt.language_code, COUNT(*) AS verses
FROM bible_verses bv
INNER JOIN bible_translations bt ON bt.id = bv.translation_id
WHERE bt.code = '<CODE>' AND bt.language_code = '<LANG>'
GROUP BY 1, 2;
```

Compare the count to a known-good number for that translation. If it doesn't match, the source file is incomplete or the parser dropped rows — surface this to the user.

After a successful seed, **call `revalidateTag("bible")`** is NOT possible from a CLI (it's a Next.js server-only API). Instead, advise the user to either redeploy or wait up to 24h for the `unstable_cache` to expire — or that they can manually invoke any server action that calls `revalidateTag` (none currently exposed). Worth-mentioning future improvement: add an admin "rebuild bible cache" action.

## OSIS book-id reference

Use this map for normalizing source book names / numbers to OSIS. Three letters per id, 66 books for the Protestant canon. The skill embeds these directly so the executor doesn't depend on a separate lookup file.

```js
export const BOOK_NAME_TO_OSIS = {
  // English (long + short)
  "Genesis": "GEN", "Gen": "GEN", "Gn": "GEN",
  "Exodus": "EXO", "Exo": "EXO", "Ex": "EXO",
  "Leviticus": "LEV", "Lev": "LEV", "Lv": "LEV",
  "Numbers": "NUM", "Num": "NUM", "Nm": "NUM",
  "Deuteronomy": "DEU", "Deut": "DEU", "Dt": "DEU",
  "Joshua": "JOS", "Josh": "JOS", "Jos": "JOS",
  "Judges": "JDG", "Judg": "JDG", "Jdg": "JDG",
  "Ruth": "RUT", "Rut": "RUT",
  "1 Samuel": "1SA", "1Sa": "1SA", "1Sam": "1SA",
  "2 Samuel": "2SA", "2Sa": "2SA", "2Sam": "2SA",
  "1 Kings": "1KI", "1Ki": "1KI", "1Kgs": "1KI",
  "2 Kings": "2KI", "2Ki": "2KI", "2Kgs": "2KI",
  "1 Chronicles": "1CH", "1Ch": "1CH", "1Chr": "1CH",
  "2 Chronicles": "2CH", "2Ch": "2CH", "2Chr": "2CH",
  "Ezra": "EZR", "Ezr": "EZR",
  "Nehemiah": "NEH", "Neh": "NEH",
  "Esther": "EST", "Est": "EST", "Esth": "EST",
  "Job": "JOB",
  "Psalms": "PSA", "Psalm": "PSA", "Ps": "PSA", "Psa": "PSA",
  "Proverbs": "PRO", "Prov": "PRO", "Pr": "PRO",
  "Ecclesiastes": "ECC", "Eccl": "ECC", "Ec": "ECC",
  "Song of Solomon": "SNG", "Song": "SNG", "SoS": "SNG", "Canticles": "SNG",
  "Isaiah": "ISA", "Isa": "ISA", "Is": "ISA",
  "Jeremiah": "JER", "Jer": "JER",
  "Lamentations": "LAM", "Lam": "LAM",
  "Ezekiel": "EZK", "Ezek": "EZK", "Eze": "EZK",
  "Daniel": "DAN", "Dan": "DAN", "Dn": "DAN",
  "Hosea": "HOS", "Hos": "HOS",
  "Joel": "JOL", "Joe": "JOL",
  "Amos": "AMO", "Am": "AMO",
  "Obadiah": "OBA", "Obad": "OBA", "Ob": "OBA",
  "Jonah": "JON", "Jon": "JON",
  "Micah": "MIC", "Mic": "MIC", "Mi": "MIC",
  "Nahum": "NAM", "Nah": "NAM",
  "Habakkuk": "HAB", "Hab": "HAB",
  "Zephaniah": "ZEP", "Zeph": "ZEP", "Zep": "ZEP",
  "Haggai": "HAG", "Hag": "HAG",
  "Zechariah": "ZEC", "Zech": "ZEC", "Zec": "ZEC",
  "Malachi": "MAL", "Mal": "MAL",
  "Matthew": "MAT", "Matt": "MAT", "Mt": "MAT",
  "Mark": "MRK", "Mk": "MRK",
  "Luke": "LUK", "Lk": "LUK",
  "John": "JHN", "Jn": "JHN",
  "Acts": "ACT", "Ac": "ACT",
  "Romans": "ROM", "Rom": "ROM",
  "1 Corinthians": "1CO", "1Co": "1CO", "1Cor": "1CO",
  "2 Corinthians": "2CO", "2Co": "2CO", "2Cor": "2CO",
  "Galatians": "GAL", "Gal": "GAL",
  "Ephesians": "EPH", "Eph": "EPH",
  "Philippians": "PHP", "Phil": "PHP", "Php": "PHP",
  "Colossians": "COL", "Col": "COL",
  "1 Thessalonians": "1TH", "1Th": "1TH", "1Thess": "1TH",
  "2 Thessalonians": "2TH", "2Th": "2TH", "2Thess": "2TH",
  "1 Timothy": "1TI", "1Ti": "1TI", "1Tim": "1TI",
  "2 Timothy": "2TI", "2Ti": "2TI", "2Tim": "2TI",
  "Titus": "TIT", "Tit": "TIT",
  "Philemon": "PHM", "Phlm": "PHM",
  "Hebrews": "HEB", "Heb": "HEB",
  "James": "JAS", "Jas": "JAS",
  "1 Peter": "1PE", "1Pe": "1PE", "1Pet": "1PE",
  "2 Peter": "2PE", "2Pe": "2PE", "2Pet": "2PE",
  "1 John": "1JN", "1Jn": "1JN", "1Jo": "1JN",
  "2 John": "2JN", "2Jn": "2JN", "2Jo": "2JN",
  "3 John": "3JN", "3Jn": "3JN", "3Jo": "3JN",
  "Jude": "JUD",
  "Revelation": "REV", "Rev": "REV", "Rv": "REV",
  // Korean (full names)
  "창세기": "GEN", "출애굽기": "EXO", "레위기": "LEV", "민수기": "NUM", "신명기": "DEU",
  "여호수아": "JOS", "사사기": "JDG", "룻기": "RUT",
  "사무엘상": "1SA", "사무엘하": "2SA", "열왕기상": "1KI", "열왕기하": "2KI",
  "역대상": "1CH", "역대하": "2CH", "에스라": "EZR", "느헤미야": "NEH", "에스더": "EST",
  "욥기": "JOB", "시편": "PSA", "잠언": "PRO", "전도서": "ECC", "아가": "SNG",
  "이사야": "ISA", "예레미야": "JER", "예레미야애가": "LAM", "에스겔": "EZK", "다니엘": "DAN",
  "호세아": "HOS", "요엘": "JOL", "아모스": "AMO", "오바댜": "OBA", "요나": "JON",
  "미가": "MIC", "나훔": "NAM", "하박국": "HAB", "스바냐": "ZEP", "학개": "HAG",
  "스가랴": "ZEC", "말라기": "MAL",
  "마태복음": "MAT", "마가복음": "MRK", "누가복음": "LUK", "요한복음": "JHN", "사도행전": "ACT",
  "로마서": "ROM", "고린도전서": "1CO", "고린도후서": "2CO", "갈라디아서": "GAL",
  "에베소서": "EPH", "빌립보서": "PHP", "골로새서": "COL",
  "데살로니가전서": "1TH", "데살로니가후서": "2TH",
  "디모데전서": "1TI", "디모데후서": "2TI", "디도서": "TIT", "빌레몬서": "PHM",
  "히브리서": "HEB", "야고보서": "JAS", "베드로전서": "1PE", "베드로후서": "2PE",
  "요한일서": "1JN", "요한이서": "2JN", "요한삼서": "3JN", "유다서": "JUD", "요한계시록": "REV",
};

// Zefania-style numeric book ids (1..66) → OSIS, in canonical order.
export const BNUM_TO_OSIS = [
  "GEN","EXO","LEV","NUM","DEU","JOS","JDG","RUT","1SA","2SA",
  "1KI","2KI","1CH","2CH","EZR","NEH","EST","JOB","PSA","PRO",
  "ECC","SNG","ISA","JER","LAM","EZK","DAN","HOS","JOL","AMO",
  "OBA","JON","MIC","NAM","HAB","ZEP","HAG","ZEC","MAL","MAT",
  "MRK","LUK","JHN","ACT","ROM","1CO","2CO","GAL","EPH","PHP",
  "COL","1TH","2TH","1TI","2TI","TIT","PHM","HEB","JAS","1PE",
  "2PE","1JN","2JN","3JN","JUD","REV",
];

export function bookNameToOsis(name) {
  if (!name) return null;
  const trimmed = String(name).trim();
  if (BOOK_NAME_TO_OSIS[trimmed]) return BOOK_NAME_TO_OSIS[trimmed];
  // Tolerate punctuation variants like "1Samuel" / "I Samuel" / "1.Samuel".
  const normalized = trimmed
    .replace(/^I\s+/i, "1 ")
    .replace(/^II\s+/i, "2 ")
    .replace(/^III\s+/i, "3 ")
    .replace(/^(\d)([A-Za-z])/, "$1 $2")
    .replace(/[.\s]+/g, " ")
    .trim();
  return BOOK_NAME_TO_OSIS[normalized] ?? null;
}
```

The `BOOK_NAME_TO_OSIS` and `BNUM_TO_OSIS` maps live inline in the generated data file (or in a small `seed-data/_bible-osis.mjs` shared module if multiple translations are seeded in the same session). Don't try to import from a TS file — the executor is `.mjs`, no tsx in the loop.

## Text normalization

Default behavior is **strip-to-plain-prose**. The student panel is a narrow reading surface; footnote anchors, italics for supplied words, cross-reference markers, and editorial brackets all add visual noise without helping comprehension. After all format-specific stripping, run a final whitespace pass:

```js
function normalizeVerseText(s) {
  return s
    .replace(/\s+/g, " ")    // collapse internal whitespace + newlines
    .replace(/\s+([,.;:!?)])/g, "$1")  // tighten punctuation
    .replace(/(\()\s+/g, "$1")
    .trim();
}
```

The format-specific strip rules below are applied **before** the whitespace pass. Each is a regex/DOM operation — keep them in a small `stripVerseText(rawText, format)` helper at the top of the executor (or the data-builder script that produces the data file) so the strip set is auditable.

### OSIS XML
- Drop `<note ...>...</note>` and everything inside (footnotes, cross-refs).
- Drop `<rdg ...>...</rdg>` (variant readings) and `<seg type="x-…">…</seg>` of editorial types.
- Drop self-closing `<milestone/>`, `<lb/>`, `<pb/>`, `<reference/>`.
- Drop divine-name styling tags (`<divineName>`, `<transChange type="added">` is the OSIS equivalent of italicized supplied words — drop the wrapper, **keep the inner text**).
- Keep all other text content; preserve verse-internal `<l>` and `<lg>` line breaks as a single space.

### USFM (plain text)
Strip the following marker pairs verbatim — including their content:
- `\f ... \f*` (footnotes)
- `\x ... \x*` (cross-references)
- `\fe ... \fe*` (end-of-text footnotes)
- `\rq ... \rq*` (inline quotation references)

Strip the following marker pairs but **keep the inner text** (drop only the markers):
- `\add ... \add*` (supplied words / italics)
- `\wj ... \wj*` (words of Jesus)
- `\nd ... \nd*` (small-cap LORD)
- `\pn ... \pn*` (proper names)
- `\bk ... \bk*` (book titles in references)
- `\qs ... \qs*` (selah)
- `\sc ... \sc*` (small caps)
- `\bd ... \bd*`, `\it ... \it*`, `\em ... \em*` (bold / italic / emphasis)
- Any unrecognized `\xx ... \xx*` pair — drop the markers, keep the text. Surface a warning in the run summary listing the unrecognized markers so the user can review.

Drop standalone markers that don't have a closing form:
- `\v <num>` — the verse-number marker; the verse number is already in our row, drop the marker token.
- `\p`, `\q`, `\m`, `\b`, `\nb`, `\li`, `\pi`, `\mi` — paragraph / poetic line markers (treat as a space).

### Zefania XML
- Drop `<NOTE ...>...</NOTE>` (footnotes) and `<XREF ...>...</XREF>` (cross-refs).
- Drop `<STYLE>...</STYLE>` wrappers but keep inner text (italics, bold, supplied words).
- Drop `<gr>...</gr>` Greek transliteration aside — keep inner text.
- Treat `<BR/>` as a single space.

### Generic JSON / CSV / TSV
- Run a generic HTML/XML strip on the `text` field: drop `<…>` tags but keep inner text (`s.replace(/<[^>]+>/g, "")`).
- Drop bracketed editorial markers: `[and]`, `[a]`, `[*]` patterns (`s.replace(/\[[a-z*†‡§\d]{1,3}\]/g, "")`).
- Drop superscript footnote glyphs: `†`, `‡`, `§`, and trailing dagger-style markers.

### Verbatim mode (`--keep-markup`)
When the user passes `--keep-markup`, **skip all of the above** and store the raw text. Still run `normalizeVerseText` for whitespace cleanup, since unprocessed XML/USFM often has source-file indentation that shouldn't reach the DB.

## What NOT to do

- **Don't write outside a transaction.** Every DB mutation in the executor — translation upsert, book upsert, verse delete, every verse-insert batch — must run inside a single `db.transaction().execute(async (trx) => { ... })` block via `trx`, never the outer `db`. Splitting the verse inserts across multiple transactions (or running them on bare `db`) means a mid-run failure leaves the table half-populated for that `translation_id` while the BibleTab is already reading it.
- **Don't fail silently on unmapped book names.** A missing book = the BibleTab will have a hole in that translation. Hard-fail with the unmapped name and the line number.
- **Don't fail silently on unrecognized USFM markers.** Drop them, but warn — an unrecognized marker may indicate the source uses an extended marker set we should add to the strip table.
- **Don't write the executor as TypeScript.** The repo's seed scripts are `.mjs` so they run with bare `node` + `--env-file`, no compile step.
- **Don't try to seed multiple translations in one executor.** One translation per executor keeps re-runs targeted (each translation can be re-seeded independently). For batch seeding, generate N pairs of files, not one big script.
- **Don't `revalidateTag` from the executor.** It's a Next.js-only API. Advise the user to redeploy or wait out the 24h `unstable_cache` TTL after re-seeding.
- **Don't strip in the executor.** Apply all stripping while *building* the data file (step 3) so the data file is the human-readable source of truth — the user can diff it, audit it, and the executor just inserts plain rows.

## License reminders to surface

When you confirm with the user (step 2), prompt them to choose the license category:

- **Public domain**: KJV, ASV, WEB, KRV (1956 Korean Revised). `licenseNotes: null`, `attribution: "Public domain (<source>, <year>)"`.
- **Open license** (e.g. CC-BY): include the license name in `licenseNotes` and the attribution string in `attribution`.
- **Restricted / proprietary**: NIV, ESV, NKRV (개역개정). The user must confirm they have written permission. Set `licenseNotes` to the license terms verbatim. If they don't have permission, refuse to seed and suggest a public-domain alternative.

The `attribution` and `licenseNotes` columns are surfaced to the student in the BibleTab footer (future), so accuracy matters.
