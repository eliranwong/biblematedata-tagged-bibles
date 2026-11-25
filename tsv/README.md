# `tagged_bibles.tsv`

Word-level alignment of the public Bible translations in this repository to each Hebrew, Aramaic, and Greek word.

Each row is one original-language word occurrence, in reading order. The translation columns show the words or short phrase aligned to that occurrence. The same `WordID` values are used in the tagged `*.bible` files described in the [repository README](../README.md): `wh{WordID}` in the Old Testament and `w{WordID}` in the New Testament. Original text comes from [OpenHebrewBible](https://github.com/eliranwong/OpenHebrewBible) and [OpenGNT](https://github.com/eliranwong/OpenGNT).

The file is UTF-8, tab-separated, with a header row and Unix line endings. It has **564,594** data rows: **426,581** Old Testament and **138,013** New Testament, covering Protestant books **1–66**.

## Columns

| Column | Description |
| :--- | :--- |
| `WordID` | Word-occurrence id. Unique within the Old Testament (`1`–`426581`) and unique within the New Testament (`1`–`138013`). Numbering starts again at `1` in Matthew (`Book` 40). |
| `Book` | Protestant book number. `1`–`39` Old Testament, `40`–`66` New Testament. |
| `Chapter`, `Verse` | Chapter and verse of this word. |
| `Word` | Surface form: pointed Hebrew or Aramaic (with cantillation) or Greek. Empty on the **6,448** Old Testament rows whose lexeme is the prefixed article `הַ`; those consonants are written on the following word. |
| `NET` | New English Translation |
| `KJV` | King James Version |
| `LEB` | Lexham English Bible |
| `BSB` | Berean Study Bible |
| `UST` | unfoldingWord Simplified Text |
| `ULT` | unfoldingWord Literal Text |
| `ASV` | American Standard Version |
| `WEB` | World English Bible |
| `CUV` | Chinese Union Version, Traditional Chinese |
| `CSVs` | Simplified Chinese of the `CUV` cell, produced with [OpenCC](https://github.com/BYVoid/OpenCC) `t2s`. Same alignment as `CUV`. |

`CUV` is the second-to-last column and `CSVs` is the last column. An empty translation cell means that version has no text aligned to this word. A filled cell may be one word or a short phrase.

Aligned cells:

| Column | Rows with text |
| :--- | ---: |
| `Word` | 558,146 |
| `NET` | 442,963 |
| `KJV` | 437,586 |
| `LEB` | 495,807 |
| `BSB` | 446,212 |
| `UST` | 419,223 |
| `ULT` | 506,971 |
| `ASV` | 476,826 |
| `WEB` | 488,914 |
| `CUV` | 347,660 |
| `CSVs` | 347,660 |

Genesis 1:1, first words:

| WordID | Word | KJV | CUV | CSVs |
| ---: | :--- | :--- | :--- | :--- |
| 1 | בְּ | In | | |
| 2 | רֵאשִׁ֖ית | the beginning | 起初 | 起初 |
| 3 | בָּרָ֣א | created | 創造 | 创造 |
| 4 | אֱלֹהִ֑ים | God | 上帝 | 上帝 |

## How to use this file

The file is the flat, spreadsheet-friendly form of the word alignment behind the tagged Bibles. Current Excel and LibreOffice both open it (564,594 data rows is under the 1,048,576-row limit). For scripts, read it as UTF-8 TSV.

**Compare translations of one original word.** Filter to a `Word` or to `Book` + `WordID`, then read across `NET` … `CSVs`. Literal and simplified English sit side by side (`ULT` and `UST`). Traditional and Simplified Chinese sit side by side (`CUV` and `CSVs`).

**Read a verse as an interlinear.** Filter to one `Book`, `Chapter`, and `Verse` and keep row order. Each row is the next Hebrew, Aramaic, or Greek word; the translation columns are what each version aligned to it.

**Study where versions diverge.** On a single word, one version may carry a phrase, another a single word, and another an empty cell. Empty means “no aligned text here,” often because that version placed the wording on a neighboring word.

**Connect the TSV to the tagged Bibles.** In `tagged_chapters_and_verses/` and `tagged_chapters_plain_verses/`, an English or Chinese span tagged `wh2` (Old Testament) or `w2` (New Testament) is the text aligned to `WordID` 2 in that testament. This TSV is the table form of those links for the public versions.

**Join lexicon and Strong’s data.** This file stops at the aligned translations. Lexicon headword ids and Extended Strong’s numbers for the same words are published separately; see below.

```python
import csv

with open("tagged_bibles.tsv", encoding="utf-8", newline="") as f:
    rows = list(csv.DictReader(f, delimiter="\t"))

verse = [
    r for r in rows
    if r["Book"] == "1" and r["Chapter"] == "1" and r["Verse"] == "1"
]
for r in verse:
    print(r["WordID"], r["Word"], r["KJV"], r["CUV"], r["CSVs"])
```

Load it in the `sqlite3` shell with `WordID` kept alongside `Book`, because `WordID` repeats across the two testaments:

```text
CREATE TABLE morphology (
  WordID INTEGER,
  Book INTEGER,
  Chapter INTEGER,
  Verse INTEGER,
  Word TEXT,
  NET TEXT, KJV TEXT, LEB TEXT, BSB TEXT,
  UST TEXT, ULT TEXT, ASV TEXT, WEB TEXT,
  CUV TEXT, CSVs TEXT
);
.mode tabs
.import --skip 1 tagged_bibles.tsv morphology
```

## Lexical mappings

[biblematedata-mapping-HALOT-CHALOT-BDBE-BDAG-LSJ](https://github.com/eliranwong/biblematedata-mapping-HALOT-CHALOT-BDBE-BDAG-LSJ) maps the same 564,594 word occurrences to lexicon entries and Extended Strong’s numbers:

- Hebrew and Aramaic: **BDBE**, **CHALOT**, **HALOT**
- Greek: **BDAG**, **LSJ**
- `eStrong` and context-specific `eStrong` (`eStrongContextSpecific`)
- Lemma (`Lexeme`), English `Gloss`, and BibleMate `LexicalEntry` ids

Use `data/tagged_bibles.tsv` in that repository. It has the same rows as this file. Join on **`Book` + `WordID`**.

Per-lexicon headword tables (`BDBE.tsv`, `CHALOT.tsv`, `HALOT.tsv`, `BDAG.tsv`, `LSJ.tsv`) and the `Morphology.lexicon` SQLite database are in that repository as well.

## See also

- [Tagged Bible layout and word-id convention](../README.md)
- [OpenHebrewBible](https://github.com/eliranwong/OpenHebrewBible)
- [OpenGNT](https://github.com/eliranwong/OpenGNT)
- [BibleMate](https://github.com/eliranwong/biblemate) · [app.biblemate.ai](https://app.biblemate.ai)
