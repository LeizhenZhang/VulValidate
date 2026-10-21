# VulValidate

Labels for vulnerability-detection benchmarks, re-derived by running the code instead of
trusting the release.

Big-Vul, DiverseVul and PrimeVul label a function vulnerable because a commit that fixed
a CVE touched it. That is a claim about a commit, not about the function. VulValidate
re-examines those claims: it builds the function, attacks it, and records what happened.
A function is called vulnerable here only when a vulnerability was actually triggered in
it; a label is changed to non-vulnerable only when an attack was attempted and nothing
triggered, never on a reading of the code.

## What is here

- [`combined_dataset/`](combined_dataset/) — all three corpora merged into one, 537,591
  rows, with the corrected labels, the fixed counterpart of every confirmed vulnerable
  function, and the provenance of each label. Start with
  [`combined_dataset/README.md`](combined_dataset/README.md).

- [`bigvul_corrected/`](bigvul_corrected/) — Big-Vul alone, in its own release format, with
  the `vul` column replaced by these labels and every other column and row left as the
  release had them. For reproducing Big-Vul-only experiments without changing their data
  loader. The 10 GB file itself is a release asset; the folder holds the full list of label
  changes and how to download and verify it.

- [`primevul_corrected/`](primevul_corrected/) — PrimeVul alone, its three release files with
  `target` replaced by these labels, partition and field layout untouched.
- [`diversevul_corrected/`](diversevul_corrected/) — DiverseVul alone, same arrangement.

Each per-corpus folder lists every label it changed from that corpus's release, with the
reason for each one.

- [`reproduction/`](reproduction/) — re-run the findings yourself. Every label marked
  `dynamic_confirmed` came from running the code, and each of those runs is packaged as a
  Docker image: one command, no clone and no account.

## Docker images

The deduplicated, dynamically confirmed samples of each corpus are packaged as Docker images on Docker Hub:

- Big-Vul: <https://hub.docker.com/repository/docker/vulvalidate/bigvul-repro/general>
- PrimeVul: <https://hub.docker.com/repository/docker/vulvalidate/primevul-repro/general>
- DiverseVul: <https://hub.docker.com/repository/docker/vulvalidate/diversevul-repro/general>

More will be added to this repository.

## Each corpus on its own

| | rows | vulnerable (release → here) | labels changed |
|---|---|---|---|
| Big-Vul | 188,636 | 10,900 → 10,792 | 2,156 |
| DiverseVul | 330,492 | 18,945 → 17,089 | 2,148 |
| PrimeVul | 224,533 | 6,004 → 6,115 | 285 |

The PrimeVul row counts **PrimeVul-v0.1**, which holds 6,004 vulnerable functions where the
PrimeVul paper reports 6,968; v0.1 keeps only the vulnerabilities whose metadata its authors
could retrieve, and [`primevul_corrected/`](primevul_corrected/) sets the two releases side
by side.

PrimeVul gains vulnerable functions where the other two lose them: most of its changes are
functions it labelled non-vulnerable that a confirmed twin in another corpus outranked.

## The merged collection in one table

| | rows | vulnerable | pairs |
|---|---|---|---|
| train | 430,023 | 12,682 | 11,744 |
| validation | 53,772 | 1,599 | 1,486 |
| test | 53,796 | 1,609 | 1,482 |

15,890 functions carry a triggered vulnerability, 3,180 had a vulnerable label removed
after an attack found nothing, and 14,712 are the post-fix counterparts of confirmed
functions. The remaining 503,809 keep the non-vulnerable label their original release
gave them.

## The audit behind these labels

Every function the three corpora label vulnerable was put through the procedure below, and
each one ends in exactly one of four outcomes. The four counts add up to each corpus's
vulnerable population.

| outcome | Big-Vul | PrimeVul | DiverseVul | total | distinct bodies |
|---|---|---|---|---|---|
| confirmed — a vulnerability was triggered in the function | 4,412 | 5,537 | 10,561 | 20,510 | 15,890 |
| label correction — attacked, nothing triggered | 4,730 | 87 | 2,002 | 6,819 | 6,774 |
| attacked, no decision reached | 1,480 | 374 | 6,127 | 7,981 | 7,777 |
| not dynamically tested | 278 | 6 | 255 | 539 | 539 |
| **total** | **10,900** | **6,004** | **18,945** | **35,849** | **30,980** |

The last column counts byte-identical function bodies within each outcome, comments and
whitespace preserved. Repetition sits almost entirely in the confirmations: 4,620 of the
4,869 repeated instances are confirmed, against 45 label corrections. The four per-outcome
counts add to 30,980; the number of bodies in their union is 30,109, because one body can
sit under different outcomes in different corpora.

The 15,890 distinct confirmed bodies are the vulnerable class of the release. Only the first
two outcomes reach the released labels: the two unresolved outcomes are kept as audit
records and left out of the learning labels rather than folded into either class.

## Why the labels matter for measurement

A detector's precision is measured against the labels, so label noise moves the score
without anything about the detector changing. On Big-Vul, scoring one unchanged LineVul
checkpoint against the corrected labels instead of the original ones takes its F1 from
0.8551 to 0.5940 — its predictions are identical, and its false positives rise from 87 to
501 because the reference changed. Its AUROC moves from 0.9646 to 0.9599. The ranking the
model produces barely moves; what moves is where a fixed decision threshold sits in the
new label distribution.

## How the labels were produced

The procedure these labels come from is published as an installable Agent Skill:

**[vuln-label-dynamic-confirmation](https://github.com/lzhan011/vuln-label-dynamic-confirmation-skill)**

It is what a coding agent in Claude Code or OpenAI Codex reads to do this work: the
operating order, the hard rules, and the per-dataset parameters for Big-Vul, PrimeVul,
MegaVul and DiverseVul. The parts that decide what the labels in this repository mean:

- **Every positive sample ends in exactly one of four outcomes** — confirmed, label noise,
  attacked without a decision, or not dynamically tested — each with required evidence
  fields, and the four counts must add up to the denominator.
- **The two-sided differential is the standard path.** Build the pre-fix and post-fix code
  with the same driver and the same input, so the only variable is the labelled function
  body, and record a real integer exit code for both sides.
- **Attribution is by source line range, never by bare function name.** A fault in a
  function called from the labelled one still confirms; a fault in a caller, a sibling, or
  in harness-written code does not.
- **A label is only called noise after a real dynamic attack**, behind seven gates including
  proof that the stored body matches upstream and that the patched lines were executed.
  Reading the code alone yields a suspicion, not a label change.
- **Failure classes are kept apart** rather than folded into "confirmed" or "not
  confirmed": synthetic triggers, harness tautologies, inverted polarity, both sides
  faulting, unattributable sanitizer output, and untested samples each have a name.

The last two are why this repository lists every changed label individually instead of
only shipping the corrected files.

## Licence and provenance

The function bodies, commit ids and CVE references come from the Big-Vul, DiverseVul and
PrimeVul releases and from the upstream open-source projects they were collected from;
their terms apply to that content. What this repository adds is the labels, the pairings
and the provenance records.

Please cite the three source corpora alongside this work.
