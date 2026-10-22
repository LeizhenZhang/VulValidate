# PrimeVul with corrected labels

PrimeVul's three release files with VulValidate's labels, in the strict version that keeps
only rows whose label VulValidate decided. Every kept row keeps all of the release's fields,
its `idx`, and its place in the release's own train/validation/test files. A second,
re-cut split with duplicate-aware rules is in `split/`.

## Which PrimeVul release this is built on

**PrimeVul-v0.1**, the metadata-enhanced release of 2024-09-05: 224,533 functions, 6,004 of
them vulnerable, 4,704 vulnerable/fixed pairs.

That is fewer than the 6,968 vulnerable and 228,800 benign functions the PrimeVul paper
reports in its Table III, and the difference comes from the PrimeVul authors. v0.1 adds commit
URLs, CVE descriptions, NVD links and file-level context, and its release note says it keeps
only the vulnerabilities whose metadata they could retrieve:

> In PrimeVul-v0.1, we only include vulnerabilities that we successfully retrieved their
> metadata. For the full set of samples that we originally used in the paper, please refer
> to the original release.

| | original release (2024-03-27) | v0.1, used here |
|---|---|---|
| vulnerable | 6,968 | 6,004 |
| benign | 228,800 | 218,529 |
| pairs | 5,480 | 4,704 |
| fields per function | 8 | 15 |

## Files in the release's own partition

| file | rows | vulnerable (release → here) | compressed |
|---|---|---|---|
| `primevul_train_relabelled.jsonl.gz` | 174,932 | 4,862 → 4,459 | 62 MB |
| `primevul_valid_relabelled.jsonl.gz` | 23,833 | 593 → 560 | 8 MB |
| `primevul_test_relabelled.jsonl.gz` | 24,698 | 549 → 518 | 9 MB |

```bash
gunzip -k primevul_*_relabelled.jsonl.gz
sha256sum -c SHA256SUMS
```

`SHA256SUMS` holds the digests of the decompressed files.

## What changed from the release

The release has 224,533 rows. This version has 223,463 rows, 5,537 of them vulnerable.
`label_changes.csv` lists all 1,157 rows that differ from the release, keyed on the release
file and `idx`, with the reason.

| change | rows | released label | what happened |
|---|---|---|---|
| `label_noise_corrected` | 73 | 1 | the function was built and attacked with inputs aimed at the claimed weakness, nothing triggered, and the label became 0 |
| `label_inherited_from_identical_body` | 14 | 1 | a function with the same normalised body in Big-Vul carries a non-vulnerable verdict, and the label was taken from it |
| `removed_no_dynamic_verdict` | 380 | 1 | no dynamic attempt reached a verdict for the function, so the row is removed |
| `removed_identical_to_a_fixed_function` | 279 | 0 | the normalised body equals the fixed version of a confirmed PrimeVul function, which VulValidate keeps as that function's fixed counterpart |
| `removed_duplicate_of_an_excluded_row` | 213 | 0 | the normalised body equals a Big-Vul row that VulValidate itself excludes |
| `removed_cross_corpus_conflict` | 198 | 0 | a function with the same normalised body in Big-Vul or DiverseVul had a vulnerability triggered in it, so the row is removed |

For the conflict rows and the inherited rows, `label_changes.csv` names the function whose
verdict decided the row.

### Rows decided through an identical Big-Vul function

VulValidate stores each normalised function body once across the three corpora. 80,904
PrimeVul rows have the same normalised body as a Big-Vul row that VulValidate keeps, and their
verdict is stored on the Big-Vul row. All 80,904 are kept here with the label of that Big-Vul
row: 1,897 confirmed vulnerable, 14 label noise (listed above), 1,990 identical to the fixed
version of a confirmed Big-Vul function, and 77,003 functions the release already labelled
non-vulnerable. `build_record.json` lists them as `kept_bigvul_twin`.

## The re-cut split

`split/` pools the kept rows of the three files and cuts them again.

| split | rows | vulnerable |
|---|---|---|
| train | 178,770 | 4,383 |
| valid | 22,379 | 607 |
| test | 22,314 | 547 |
| total | 223,463 | 5,537 |

- Random 80/10/10 cut with seed 42.
- Functions whose bodies are equal after removing C/C++ comments and all whitespace are kept
  in one split. `split/stats.json` confirms that 0 such bodies occur in more than one split.
- Pairs come from the release's own `primevul_*_paired.jsonl` files, and both members of a
  pair are always in the same split.

`split/` holds `{train,valid,test}.jsonl.gz` with every field of the release,
`{train,valid,test}_paired.jsonl.gz` with each pair as two consecutive lines
(`func_fix_before`, then `func_fix_after`), `split_assignment.csv.gz`, `stats.json`, and
`SHA256SUMS` for the decompressed files.

## What a label here means

A 1 means a vulnerability was observed while running the function. It was compiled, reached
with an attack input, and the resulting failure was attributed to it by source line range.
The CWE and CVE fields are reproduced from PrimeVul unchanged.

A 0 that came from `label_noise_corrected` means an attack was attempted and nothing
triggered within the effort spent. That is weaker evidence than a proof of safety, which is
why those rows are listed individually.

## The earlier labels-only version

The previous content of this folder kept all 224,533 rows, flipped the 198 conflict rows to 1
and kept the release's label where no verdict exists. Those files are in this folder's git
history.

## Related

The merged three-corpus collection is in [`../combined_dataset/`](../combined_dataset/), with one schema
across Big-Vul, DiverseVul and PrimeVul, a `label_source` field on every row, and the fixed
counterpart of every confirmed vulnerable function.
