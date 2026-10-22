# Big-Vul with corrected labels

Big-Vul (`MSR_data_cleaned.csv`) with VulValidate's labels, in the strict version that keeps
only rows whose label VulValidate decided. A vulnerable label here means a vulnerability was
triggered in the function. Rows that no dynamic attempt reached are removed. This folder ships
the train/validation/test split of that version.

## The split

Every row keeps its Big-Vul row index (`idx`, `id = bigvul:<idx>`), so any result can be traced
back to the release.

| split | rows | vulnerable |
|---|---|---|
| train | 145,766 | 3,537 |
| valid | 18,245 | 439 |
| test | 18,243 | 436 |
| total | 182,254 | 4,412 |

Rules of the split (`split/stats.json` holds the full settings):

- Random 80/10/10 cut with seed 42.
- Functions whose bodies are equal after removing C/C++ comments and all whitespace are kept
  in one split. `stats.json` confirms that 0 such bodies occur in more than one split.
- A vulnerable function and its fixed version are always in the same split.

| file | content |
|---|---|
| `split/{train,valid,test}.jsonl.gz` | one function per line: `idx`, `id`, `target`, `func` (the pre-fix body) |
| `split/{train,valid,test}_paired.jsonl.gz` | every vulnerable function as two consecutive lines, `func_fix_before` (target 1) then `func_fix_after` (target 0), with a shared `pair_int_id` |
| `split/split_assignment.csv.gz` | the split of every row index |
| `split/stats.json` | counts, settings and the leakage check |
| `split/SHA256SUMS` | digests of the decompressed files |

```bash
cd split
gunzip -k *.gz
sha256sum -c SHA256SUMS
```

## What changed from the release

The release has 188,636 rows, 10,900 of them vulnerable. This version has 182,254 rows,
4,412 of them vulnerable. `label_changes.csv` lists all 7,512 rows that differ from the
release, keyed on the release's row index, with the reason.

| change | rows | released label | what happened |
|---|---|---|---|
| `label_noise_corrected` | 1,130 | 1 | the function was built and attacked with inputs aimed at the claimed weakness, nothing triggered, and the label became 0 |
| `removed_no_dynamic_verdict` | 5,356 | 1 | no dynamic attempt reached a verdict for the function, so the row is removed |
| `removed_identical_to_a_fixed_function` | 2 | 1 | the normalised body equals the fixed version of another Big-Vul row, so the row is removed |
| `removed_cross_corpus_conflict` | 1,024 | 0 | a function with the same normalised body in DiverseVul or PrimeVul had a vulnerability triggered in it, so the row is removed |

The two rows of the third kind are 179657 and 179666, whose bodies equal the repaired
versions of 179219 and 179214. For the 1,024 conflict rows, `label_changes.csv` names the
confirmed function that outranked the release's label and the shared body hash.

The remaining 181,124 rows keep the release's label: 4,412 confirmed vulnerable functions
and 176,712 functions the release already labelled non-vulnerable.

## What a label here means

A 1 means a vulnerability was observed while running the function. It was compiled, reached
with an attack input, and the resulting failure was attributed to it by source line range.
The CWE and CVE columns are reproduced from Big-Vul unchanged, and a function can trigger a
different weakness than the one it was filed under.

A 0 that came from `label_noise_corrected` means an attack was attempted and nothing
triggered within the effort spent. That is weaker evidence than a proof of safety, which is
why those rows are listed individually.

## Other files

- `label_changes.csv`: all 7,512 rows that differ from the release
- `build_record.json`: the counts the build produced and how each decision was sourced

## The earlier labels-only version

The previous content of this folder kept all 188,636 rows, flipped the 1,024 conflict rows to
1 and kept the release's label where no verdict exists. That CSV remains attached to the
[`bigvul-corrected` release](https://github.com/LeizhenZhang/VulValidate/releases/tag/bigvul-corrected),
and its documentation is in this folder's git history.

## Related

The merged three-corpus collection is in [`../combined_dataset/`](../combined_dataset/), with one schema
across Big-Vul, DiverseVul and PrimeVul, a `label_source` field on every row, and the fixed
counterpart of every confirmed vulnerable function.
