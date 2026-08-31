---
license: other
language:
- en
task_categories:
- text-classification
- text-generation
tags:
- security
- smart-contracts
- code-audit
- vulnerability
- solidity
pretty_name: Smart Contract Audit Findings
size_categories:
- 10K<n<100K
configs:
- config_name: default
  data_files:
  - split: train
    path: data/train-00000-of-00001.parquet
---

# Smart Contract Audit Findings

> **This is raw, semi-structured data — not a ready-to-train dataset.** It still requires
> further cleaning and preparation (deduplication, severity/label normalization, filtering
> low-quality or malformed entries, etc.) before it should be used to train or fine-tune an AI model.

A collection of **23,625 smart-contract security audit findings** (bug reports), each with a
title, description, proof-of-concept code, recommendation, and severity rating.

## Dataset structure

The primary format is a single Parquet file at `data/train-00000-of-00001.parquet`
(loadable via `datasets`/`pandas`/`polars`). The original, unmodified source CSV is kept
under `raw/` for provenance.

### Columns

| column | type | description |
|---|---|---|
| `id` | int | row identifier |
| `file_name` | string | source file name the finding was extracted from |
| `bug_title` | string | finding title (often includes the reporting researcher's handle) |
| `bug_desc` | string | free-text description of the vulnerability |
| `bug_poc` | string | proof-of-concept / exploit code, usually Solidity/Foundry |
| `bug_rec` | string | recommended fix |
| `bug_sev` | string | severity, normalized to one of: `Critical`, `High`, `Medium`, `Low`, `Gas Optimization`, `Informational`, `Other`, `Unknown` |
| `bug_sev_raw` | string | original, unnormalized severity text as it appeared in the source |
| `bug_weight` | float | a 0–1 importance score computed from write-up length/detail and severity, see [How `bug_weight` is computed](#how-bug_weight-is-computed) |
| `bug_full` | string | full combined text of the finding |

## How `bug_weight` is computed

`bug_weight` is produced by an earlier pipeline step (`step_2/weight.py`) that scores each
source finding (a markdown file with `## description`, `## recommendation`, `## poc` and
`## severity` sections) as follows:

1. **Length factor per section** (`description`, `recommendation`, `poc`): 0 if the section is
   shorter than 50 characters, 1.0 if it's 2000+ characters, otherwise `len(section) / 2000`
   (i.e. a linear ramp between those two thresholds).
2. **Weighted content score**: `1.2 * desc_factor + 0.5 * rec_factor + 1.8 * poc_factor`
   — the PoC section is weighted heaviest, the recommendation lightest.
3. **Code-block bonus**: count ` ```solidity ` fenced blocks in each of the three sections
   (capped at 2 per section, so up to 6 total), `code_bonus = 2.0 * total_blocks`.
4. **Severity multiplier**, from a substring match on the file's own `## severity` section
   (checked in this order: `critical` → `high` → `medium` → `low` → `info`; anything that
   matches none of these — including labels like `"Major"` or scraping artifacts — silently
   falls back to the `Low` multiplier):
   `Critical: 1.5`, `High: 1.5`, `Medium: 0.8`, `Low: 0.4`, `Info: 0.1`, default: `0.4`.
5. **Raw weight**: `(content_score + code_bonus) * severity_multiplier`, capped at `15.0`.
6. **Normalization**: `bug_weight = ln(1 + raw_weight) / ln(1 + 15)`, rounded to 8 decimals —
   a log-compressed score in `[0, 1]`.

Two consequences worth knowing before using this column as a ranking signal:

- **`Critical` and `High` share the same multiplier (1.5)** — the formula does not distinguish
  between them at all beyond that shared tier. Any weight difference you see between findings
  labeled `Critical` vs. `High` in this dataset comes entirely from write-up length and code-block
  count, not from severity.
- The severity match here is a coarse, independent substring check run by the pipeline at
  scoring time — it is not the same value as this dataset's `bug_sev`/`bug_sev_raw` columns.
  Any severity text that doesn't contain `critical`/`high`/`medium`/`low`/`info` (e.g. `"Major"`,
  or a scraping artifact like `"Commit\nLocation"`) is silently scored as `Low` (multiplier 0.4),
  which is why such rows tend to have low `bug_weight` regardless of their true severity.

Net effect: treat `bug_weight` primarily as a **write-up thoroughness score gated by a coarse
severity tier**, not as a fine-grained or fully severity-accurate importance ranking.

## Notes / known limitations

- **`bug_poc` is a placeholder (no real code) in ~78.1% of rows** (18,443 / 23,625) — values like
  `"no poc"`, `"No poc."`, `"N/A"`, `"No data"` rather than actual exploit code. Filter on
  `bug_poc` length/content before using this column for PoC-generation training.
- **`bug_rec` is a placeholder in ~12.2% of rows** (2,880 / 23,625) — values like
  `"No recommendation"`, `"No data"`.
- A handful of rows (3) are exact full-record duplicates (`bug_full`); 136 rows share an
  identical `bug_desc` with at least one other row. Not deduplicated in this release.
- Findings were originally scraped from multiple audit-contest sources with inconsistent
  severity labeling (e.g. `"Low Risk"`, `"low"`, `"Crit"`, or labels with an appended
  `"Reported By <handle>"` line). `bug_sev` normalizes these into a fixed set of categories;
  `bug_sev_raw` preserves the original text for anyone who wants to re-derive their own mapping.
- 216 rows (~0.9%) could not be confidently mapped to a severity and are labeled `Unknown`;
  147 of those had `"Commit\nLocation"` as the raw value, which looks like a scraping artifact
  rather than an actual severity.
- License/authorship of the underlying audit reports is unclear (findings appear to originate
  from third-party researchers on public audit-contest platforms). Treat this dataset as
  `license: other` until provenance is confirmed; do not assume redistribution or commercial-use
  rights beyond what the original platforms grant.

## Intended use

Training/evaluating models on smart-contract vulnerability detection, classification, and
report/PoC generation for **defensive security research** purposes.
