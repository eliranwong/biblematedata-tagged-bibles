# biblematedata-tagged-bibles

Word-level tagged Bible SQLite databases (`*.bible`) for the **BibleMate AI** ecosystem. Each Bible translation is mapped at the word/phrase level to the original Hebrew/Aramaic and Greek words in [OpenHebrewBible](https://github.com/eliranwong/OpenHebrewBible) and [OpenGNT](https://github.com/eliranwong/OpenGNT).

Live Demo: [web.biblemate.ai](https://web.biblemate.ai)

---

## Repository Structure

### Directories

- **`tagged_chapters_and_verses/`** — `.bible` SQLite databases with word-ID taggings in **both** the `Bible` (chapter-level) and `Verses` (verse-level) tables.
- **`tagged_chapters_plain_verses/`** — `.bible` SQLite databases with word-ID taggings in the `Bible` (chapter-level) table and **plain, untagged text** in the `Verses` (verse-level) table.

### Included Translations

| File | Translation | Language |
| :--- | :--- | :--- |
| `ASV.bible` | American Standard Version | English |
| `BSB.bible` | Berean Study Bible | English |
| `CUV.bible` | Chinese Union Version (Traditional Chinese) | Chinese |
| `CUVs.bible` | Chinese Union Version (Simplified Chinese) | Chinese |
| `KJV.bible` | King James Version | English |
| `LEB.bible` | Lexham English Bible | English |
| `NET.bible` | New English Translation | English |
| `ULT.bible` | unfoldingWord Literal Text | English |
| `UST.bible` | unfoldingWord Simplified Text | English |
| `WEB.bible` | World English Bible | English |

### Mapping Documentation

Detailed pipeline notes for word-ID alignment and morphology mapping:

- [bsb_morphology_mapping_process.md](bsb_morphology_mapping_process.md)
- [cuv_morphology_mapping_process.md](cuv_morphology_mapping_process.md)
- [kjv_morphology_mapping_process.md](kjv_morphology_mapping_process.md)
- [leb_morphology_mapping_process.md](leb_morphology_mapping_process.md)
- [net_morphology_mapping_process.md](net_morphology_mapping_process.md)

---

## Database Schema & Tagging Format

Each `.bible` file is a SQLite database containing three tables:

- **`Details`** — Bible metadata (`Title`, `Abbreviation`, `Information`, `Version`, `OldTestament`, `NewTestament`, `Apocrypha`, `Strongs`, and optionally `Language`).
- **`Bible`** — Chapter-level HTML/text (`Book INT, Chapter INT, Scripture TEXT`).
- **`Verses`** — Verse-level text (`Book INT, Chapter INT, Verse INT, Scripture TEXT`).

### Word-ID Tagging Convention

Word and phrase taggings link translated text to `WordID` entries in `OpenHebrewBible` and `OpenGNT`:

- **Old Testament (`Book <= 39`)**: `id="wh{WordID}"`
  - English: `<en id="wh1">In</en> <en id="wh2">the beginning</en> ...`
  - Chinese: `<zh id="wh2">起初</zh><zh>，</zh><zh id="wh4">上帝</zh> ...`
- **New Testament (`Book >= 40`)**: `id="w{WordID}"`
  - English: `<en id="w49133">In</en> <en id="w49134">the beginning</en> ...`
  - Chinese: `<zh id="w49134">太初</zh><zh id="w49135">有</zh> ...`

---

# `*.bible` File Structure & Word-ID Tagging Guide

This guide documents the internal structure of BibleMate/Marvel `*.bible` SQLite databases and the complete word-ID tagging pipeline implemented in [SKILL.md](https://github.com/eliranwong/biblemate-agentic-workspace/blob/main/.agents/skills/tag-bible/SKILL.md) and [tag_bible.py](https://github.com/eliranwong/biblemate-agentic-workspace/blob/main/.agents/skills/tag-bible/tag_bible.py), including all special cases encountered across `CUV`, `RCUV`, `NET`, `KJV`, `LEB`, `BSB`, `UST`, `ULT`, `NIV`, `NLT2015`, `NASB1995`, `NASB2020`, `ASV`, and `WEB`.

---

## 1. `*.bible` SQLite File Structure

A standard `*.bible` file is a SQLite database containing three core tables (plus an optional `Notes` table):

### 1.1 `Details`
Stores version metadata:
```sql
CREATE TABLE Details (
    Title NVARCHAR(100),
    Abbreviation NVARCHAR(50),
    Information TEXT,
    Version INT,
    OldTestament BOOL,
    NewTestament BOOL,
    Apocrypha BOOL,
    Strongs BOOL
    -- Some files also include Language, FontSize, FontName
)
```

### 1.2 `Verses`
Stores one row per verse (`Book INT, Chapter INT, Verse INT, Scripture TEXT`):
- **Protestant book numbering**: `Book` `1`–`39` (OT), `40`–`66` (NT). Some files include a metadata row `(0, 0, 0, '<Title>')`.
- **Plain vs. language-wrapped text**:
  - English Bibles (`NET`, `KJV`, `LEB`, `BSB`, `UST`, `ULT`, `ASV`, `WEB`, `NASB1995`, `NASB2020`) store plain text (or text with inline markers/footnotes).
  - Chinese Bibles (`CUV`, `RCUV`) wrap verse text in `<zh>...</zh>` (or plain text in `RCUV` `Verses`).

### 1.3 `Bible`
Stores chapter-level formatted HTML (`Book INT, Chapter INT, Scripture TEXT`), typically `1,189` chapters:
- Each verse is wrapped in `<verse>...</verse>` with an anchor `<vid id="v{Book}.{Chapter}.{Verse}" onclick="luV({Verse})">{Verse}</vid>`.
- May contain section headings (`<u><b>...</b></u>`), cross-references (`<ref onclick="bcv(...)">...</ref>`), paragraph/poetic breaks (`<br>`, `<pb/>`, `&emsp;`), inline styling (`<span class="smcaps">Lord</span>`, `<i>...</i>`), and inline footnote/name tags (`<mbn>...</mbn>`, `<mbe>...</mbe>`, `<sup>...</sup>`).

---

## 2. Word-ID Tagging Convention

Matching [OHGB.bible](file:///home/eliran/dev/OpenHebrewBible/data/OHGB.bible) and the `WordID` primary sequence in `morphology.sqlite`:
- **OT (`Book <= 39`)**: `id="wh{WordID}"` (e.g., `<en id="wh1">In</en>`, `<zh id="wh2">起初</zh>`).
- **NT (`Book >= 40`)**: `id="w{WordID}"` (e.g., `<en id="w1">The book</en>`, `<zh id="w3">耶穌</zh>`).
- **Untagged spans**:
  - English (`<en>`): spaces, punctuation, and unmapped words stay outside `<en>` tags.
  - Chinese (`<zh>`): unmapped characters, punctuation, and notes are wrapped in untagged `<zh>...</zh>` so the full verse remains inside `<zh>` elements.

---

## 3. End-to-End Tagging Pipeline

### 3.1 Safety, Paths & Execution Order
1. **Morphology database priority**:
   - Use `~/biblemate/data_custom/morphology.sqlite` if it exists, else `~/biblemate/data/morphology.sqlite`.
   - Always create/overwrite `<morphology_path>.bak` before modifying `morphology.sqlite`.
2. **Output isolation**:
   - Copy the source `.bible` to `biblemate_tagged_bibles/<name>.bible` (creating the directory if needed).
3. **Strict 3-step order**:
   - **Step 1**: Tag `Verses` table first and verify **0 text mismatches** when stripping `<en>`/`<zh>` tags.
   - **Step 2**: Add/populate the `<Version>` column at the end of `morphology` **strictly by `rowid`**.
   - **Step 3**: Propagate `Verses` tags into the `Bible` table last, verifying **0 mismatches** in chapter HTML.

### 3.2 Base Alignment & Refinement Stages
- **Stage 1 (Phrase Boundaries)**:
  - *With helper* (`*x.bible`, TSV like `bsb_tables.tsv`, or USFM 3.0 directory): extract **only** phrase boundaries (ignoring Strong's numbers in the base stage) and align onto clean `Verses` text via `difflib.SequenceMatcher`.
  - *Without helper (Standalone)*: for English, use longest-phrase matching seeded from `morphology.sqlite`'s existing English columns (`NET`, `KJV`, `LEB`, `BSB`, `UST`, `ULT`, `NASB1995`, `NASB2020`, `ASV`, `WEB`, `NIV`, `NLT2015`, `ESV2016`, `NRSV`, `Translation`, `Interlinear`, `Gloss`); for Chinese, use PMI n-gram discovery + `morphology.CUV` Viterbi segmentation.
- **Stage 2 (Clean Base EM)**:
  - Run `6` iterations of symmetric Dirichlet-smoothed EM (`alpha = 2.0`, 3 IBM Model 1 + 3 mild position-prior iterations) between `eStrongContextSpecific` and target tokens, combined with English lexical stem similarity (`Translation`, `Interlinear`, `Gloss`) and decoded via confidence-gated 1-to-1 bipartite matching.
  - Suppress untranslated functional markers (`H853` `[object marker]`, `H9009`, `G3588` `-`).
- **Stage 3 (Validated Helper Strong's Check)**:
  - Validate helper `H...`/`G...` numbers against `eStrongContextSpecific`, `eStrong`, and `LexicalEntry` in the same verse of `morphology.sqlite` to fill unclaimed gaps and override conflicts only when global `(eStrongContextSpecific, word)` support is `>= 2x` higher.
- **Stage 4 (`v ± 1` Sliding-Window Recovery)**:
  - Match remaining untagged tokens against unmapped `WordID`s in adjacent verses (`v - 1`, `v + 1`, including chapter boundaries) **only** for low-coverage/shifted verses (`< 40%` filled) or high lexical overlap (`>= 0.7`).
- **Stage 5 (Language-Specific Refinement)**:
  - **English**: Peel leading function words (`in`, `the`, `and`, `then`, `of`, `to`, `for`, `from`, `unto`, etc.) onto adjacent empty `WordID`s, and split/reassign multi-word phrases and single-word leftovers across empty `WordID`s in the same verse (ordered by proximity, checking core `Translation`/`Interlinear`/`Gloss` columns).
  - **Chinese**: Guarded `LCT` refinement matches untagged `<zh>` spans against `morphology.LCT` while blocking suppressed particles and noisy single characters (`那`, `這`, `的`, `得`, etc.).
- **Stage 6 (`Bible` Table Projection)**:
  - Tokenize each `<verse>` body in `Bible` into HTML/entity/whitespace tokens vs. text runs, map `Verses` word IDs character-by-character, and wrap text runs cleanly.

---

## 4. Special Cases Encountered & Solutions

1. **SQLite `UPDATE` Performance Bottleneck (`~118 min` → `< 5 sec`)**:
   - *Issue*: `morphology` has `564,594` rows with no index on `WordID` or `(Book, WordID)`. Running `UPDATE morphology SET col = ? WHERE Book <= 39 AND WordID = ?` caused hundreds of thousands of full-table scans.
   - *Fix*: Preload `(rowid, Book, Chapter, Verse, WordID, ...)` in one sequential scan and execute `UPDATE morphology SET col = ? WHERE rowid = ?`.
2. **Crossed HTML Tags in `Bible` Table (`NET` Ps 23:1)**:
   - *Issue*: Propagating a `WordID` across `<span class="smcaps">` produced `<en id="wh313896">The <span class="smcaps">Lord</en></span>`.
   - *Fix*: Propagate `tid` across whitespace **only**, never across `<...>` HTML tags, yielding valid `<en id="wh313896">The</en> <span class="smcaps"><en id="wh313896">Lord</en></span>`.
3. **Dynamic-Equivalence `v ± 1` False Matches (`UST` Gen 1:1–2)**:
   - *Issue*: Running `v ± 1` unconditionally on `UST` matched `"there"` from `Gen 1:2` onto empty `WordID=1` (`בְּ`) in `Gen 1:1`.
   - *Fix*: Restricted Stage 4 `v ± 1` to low-coverage verses (`< 40%` filled) or strong lexical matches (`>= 0.7` with `sup >= 1`).
4. **Single-Word Leftovers & Narrow `±3` Window (`NASB1995`/`NASB2020` Gen 1:1)**:
   - *Issue*: After Stage 5a peeled `"the"` out of `"created the heavens"`, `<en id="wh7">created</en>` became a single-word tag on `שָּׁמַיִם` (`WordID=7`), `4` positions away from `בָּרָא` (`WordID=3`)—outside `±3`—and `NASB2020` saw `NASB1995`'s stale `'created heavens'` on row 7.
   - *Fix*: Searched all empty `WordID`s in the verse ordered by proximity, checked `host_core` strictly against `Translation`/`Interlinear`/`Gloss`, and reassigned single-word leftovers (`created` → `WordID=3`).
5. **Psalm `Verse = 1` Superscriptions**:
   - *Issue*: `morphology.sqlite` includes Psalm superscriptions in `Verse = 1`, whereas `CUV` and `BSB` `Verses` omit them.
   - *Fix*: Detect omitted superscriptions in `Book = 19, Verse = 1` and trim leading superscription `WordID`s before EM alignment.
6. **Inline Translator Footnotes (`CUV`, `RCUV`, `WEB`)**:
   - *Issue*: `CUV` has `〔註：...〕`, `RCUV` has `<mbn>...</mbn>` (and inline note text in `Verses`), and `WEB` has `[1:1 ...]` bracketed notes.
   - *Fix*: Extract/mask all footnote spans before tokenization and advance past `<mbn>...</mbn>` during `Bible` table projection so footnotes stay untagged.
7. **Sibling / Revised Editions (`NASB1995 → NASB2020`, `KJV → ASV → WEB`, `CUV → RCUV`)**:
   - *Issue*: Re-running blind PMI on `RCUV` merged `天`+`地` (`天地`) and `地`+`是`+`空虛` (`地是空虛`), losing `CUV`'s clean word boundaries.
   - *Fix*: Project `CUV` boundaries character-by-character onto `RCUV` first (normalizing `上帝 ↔ 神`, Traditional variants, and `cu1995t → rcuv2011t` TSV names), and use `--norm-archaic` (`Jehovah`/`LORD` ↔ `Yahweh`, `-eth`, `-est`, `thou`/`ye` ↔ `you`) for `ASV → WEB`.
8. **USFM 3.0 Alignment Directories (`en_ust`, `en_ult`)**:
   - *Issue*: USFM folders include front-matter (`A0-FRT.usfm`) and number NT books `41`–`67` (skipping `40`).
   - *Fix*: Skip non-digit prefixes, map `41..67` → `40..66`, and parse `\zaln-s`/`\zaln-e\*`/`\w` alignment stacks.

---

## Related Projects

### Original Language Sources
- [OpenHebrewBible](https://github.com/eliranwong/OpenHebrewBible)
- [OpenGNT](https://github.com/eliranwong/OpenGNT)

### Other BibleMateData Projects
- [biblematedata](https://github.com/eliranwong/biblematedata)
- [biblematedata-names](https://github.com/eliranwong/biblematedata-names)
- [biblematedata-mapping-HALOT-CHALOT-BDBE-BDAG-LSJ](https://github.com/eliranwong/biblematedata-mapping-HALOT-CHALOT-BDBE-BDAG-LSJ)
- [More repositories by eliranwong](https://github.com/eliranwong?tab=repositories)

### BibleMate AI Ecosystem
- [biblemate](https://github.com/eliranwong/biblemate)
- [biblemateweb](https://github.com/eliranwong/biblemateweb)
- [biblemate-agentic-workspace](https://github.com/eliranwong/biblemate-agentic-workspace)
