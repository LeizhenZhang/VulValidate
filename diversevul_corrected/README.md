# DiverseVul with corrected labels

The DiverseVul release file with VulValidate's labels, in the strict version that keeps only
rows whose label VulValidate decided. Every kept row keeps all of the release's fields and
the release's order. The fixed versions of confirmed functions and a duplicate-aware
train/validation/test split are included.

## Files

The data file is newline-delimited JSON despite its `.json` name. GitHub caps a file inside a
repository at 100 MB, so it ships as two pieces, each a valid gzip stream on its own:

| file | rows | compressed |
|---|---|---|
| `diversevul_20230702_relabelled.part1.json.gz` | 161,937 | 58 MB |
| `diversevul_20230702_relabelled.part2.json.gz` | 161,937 | 49 MB |
| `fixed_functions.jsonl.gz` | 9,491 | 9 MB |

```bash
cat diversevul_20230702_relabelled.part1.json.gz \
    diversevul_20230702_relabelled.part2.json.gz \
  | gunzip > diversevul_20230702_relabelled.json
gunzip -k fixed_functions.jsonl.gz
sha256sum -c SHA256SUMS
```

Concatenated gzip members decompress as one stream, so that reproduces the single file, and
its digest is in `SHA256SUMS`.

DiverseVul's release carries no row id, so rows are identified by their **line number in the
release file**, counting from 0. This version removes some lines, so the line of a row in
`diversevul_20230702_relabelled.json` differs from its release line. `label_changes.csv` lists
every removed release line, and the release lines minus the removed ones, in order, give the
rows of this file. The split files below carry the release line directly (`idx`, and
`id = diversevul:L<line>`).

## What changed from the release

The release has 330,492 rows, 18,945 of them vulnerable. This version has 323,874 rows, 10,561
of them vulnerable. `label_changes.csv` lists all 8,620 rows that differ from the release,
keyed on the release line, with the reason.

| change | rows | released label | what happened |
|---|---|---|---|
| `label_noise_corrected` | 1,977 | 1 | the function was built and attacked with inputs aimed at the claimed weakness, nothing triggered, and the label became 0 |
| `label_inherited_from_identical_body` | 25 | 1 | a function with the same normalised body in PrimeVul carries a non-vulnerable verdict, and the label was taken from it |
| `removed_no_dynamic_verdict` | 6,383 | 1 for 6,382, 0 for 1 | no dynamic attempt reached a verdict for the function, so the row is removed |
| `removed_duplicate_of_an_excluded_row` | 80 | 0 | the normalised body equals a row that VulValidate itself excludes |
| `removed_identical_to_a_fixed_function` | 9 | 0 | the normalised body equals the fixed version of a confirmed function |
| `removed_cross_corpus_conflict` | 146 | 0 | a function with the same normalised body in Big-Vul or PrimeVul had a vulnerability triggered in it, so the row is removed |

For the conflict rows and the inherited rows, `label_changes.csv` names the function whose
verdict decided the row.

### Rows decided through an identical PrimeVul or Big-Vul function

VulValidate stores each normalised function body once across the three corpora. 125,808
DiverseVul rows have the same normalised body as a PrimeVul row (125,807) or a Big-Vul row (1)
that VulValidate keeps, and their verdict is stored on that row. All of them are kept here with
that row's label: 2,723 confirmed vulnerable, 25 with a non-vulnerable verdict (listed above),
and 123,060 functions the release already labelled non-vulnerable. `build_record.json` lists
them as `kept_dedup_twin`.

## Fixed versions

The DiverseVul release ships no fixed versions. `fixed_functions.jsonl` holds the fixed version
of 9,491 confirmed vulnerable functions, keyed by `of_source_index = L<release line>`:

- 7,149 come from VulValidate's own pairing of DiverseVul functions;
- 2,342 are copies of the fixed version stored with the identical PrimeVul function, for the
  confirmed functions kept through a PrimeVul twin. Each copy names its source in
  `copied_from`.

381 of the 2,723 confirmed functions kept through a PrimeVul twin have no fixed version in
VulValidate or in the PrimeVul release, so they form no pair.

## The split

| split | rows | vulnerable | vulnerable/fixed pairs |
|---|---|---|---|
| train | 259,126 | 8,421 | 7,552 |
| valid | 32,385 | 1,062 | 958 |
| test | 32,363 | 1,078 | 981 |
| total | 323,874 | 10,561 | 9,491 |

- Random 80/10/10 cut with seed 42.
- Functions whose bodies are equal after removing C/C++ comments and all whitespace are kept
  in one split. `split/stats.json` confirms that 0 such bodies occur in more than one split.
- A vulnerable function and its fixed version are always in the same split.

`split/` holds `{train,valid,test}.jsonl.gz` with every field of the release,
`{train,valid,test}_paired.jsonl.gz` with each pair as two consecutive lines
(`func_fix_before`, then `func_fix_after`), `split_assignment.csv.gz`, `stats.json`, and
`SHA256SUMS` for the decompressed files.

## What a label here means

A 1 means a vulnerability was observed while running the function. It was compiled, reached
with an attack input, and the resulting failure was attributed to it by source line range.
The CWE field is reproduced from DiverseVul unchanged.

A 0 that came from `label_noise_corrected` means an attack was attempted and nothing
triggered within the effort spent. That is weaker evidence than a proof of safety, which is
why those rows are listed individually.

## The earlier labels-only version

The previous content of this folder kept all 330,492 rows, flipped the 146 conflict rows to 1
and kept the release's label where no verdict exists. Those files are in this folder's git
history.

## Related

The merged three-corpus collection is in [`../combined_dataset/`](../combined_dataset/), with one schema
across Big-Vul, DiverseVul and PrimeVul and a `label_source` field on every row.
