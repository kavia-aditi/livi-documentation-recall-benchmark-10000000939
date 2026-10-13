# Documentation Recall Comparison Scorecard

[CodeWiki](../../index.md) / [Quality](../index.md) / [Other Quality](index.md)

## Status and use

This scorecard is blank: no KAVIA or Mintlify documentation has been scored. Use it with the [benchmark protocol](documentation-recall-benchmark.md) and [reference set](documentation-recall-reference.md). Copy the embedded CSV to a separate evaluation workbook for each pair of frozen corpora; retain independent-review and adjudicated versions. The CSV is a template, not measured data.

Keep the reference IDs, levels, and weights fixed. Enter grades only as 0, 0.5, or 1. For every positive grade, supply an export-relative locator and a verbatim supporting excerpt. For a zero grade, record whether the fact was absent, vague, incorrect, or contradicted in the review log. When quoting comma-containing or multiline excerpts in CSV, use proper CSV quoting.

Blank grades mean pending. Do not fill them with zero merely to obtain a score.

## Run record

Complete this record for both workflows before scoring. Do not replace unknown configuration values with guesses.

| Field | KAVIA run | Mintlify run |
| --- | --- | --- |
| Run ID and track | Pending | Pending |
| UTC date | Pending | Pending |
| Source commit | `fa5e341d76990a53f13cb7aea6431e17bf5504b3` | `fa5e341d76990a53f13cb7aea6431e17bf5504b3` |
| Eight source checksums | Pending | Pending |
| Reference version | `LIVI-DR-1.0` | `LIVI-DR-1.0` |
| Product version and exact workflow | Unknown | Unknown |
| Model and relevant settings | Unknown | Unknown |
| Exact prompt and source isolation | Pending | Pending |
| Budget, passes, and elapsed time | Pending | Pending |
| Setup effort and manual edits | Pending | Pending |
| Output word count | Pending | Pending |
| Frozen export location and checksum | Pending | Pending |
| Eligibility and compliance status | Pending | Pending |
| Initial reviewers and adjudicator | Pending | Pending |

For Track B, also record the original documentation authors, audience, originating generation workflow if any, source revision alignment, and authoring effort. If revision alignment cannot be established, report the scores as corpus coverage with an explicit version-mismatch caveat.

## Per-fact CSV template

The six areas are CFG, TRN, TEL, UPD, IPC, and NAT. Columns E and H are the KAVIA and Mintlify grades, respectively. Columns F/G and I/J capture evidence. Do not sort or insert rows when using the range-based formulas below; if the workbook is rearranged, update every range consistently.

```csv
fact_id,area,level,weight,kavia_grade,kavia_locator,kavia_excerpt,mintlify_grade,mintlify_locator,mintlify_excerpt
CFG-01,CFG,L1,1,,,,,,
CFG-02,CFG,L1,1,,,,,,
CFG-03,CFG,L2,2,,,,,,
CFG-04,CFG,L2,2,,,,,,
CFG-05,CFG,L3,3,,,,,,
CFG-06,CFG,L3,3,,,,,,
TRN-01,TRN,L1,1,,,,,,
TRN-02,TRN,L1,1,,,,,,
TRN-03,TRN,L2,2,,,,,,
TRN-04,TRN,L2,2,,,,,,
TRN-05,TRN,L3,3,,,,,,
TRN-06,TRN,L3,3,,,,,,
TEL-01,TEL,L1,1,,,,,,
TEL-02,TEL,L1,1,,,,,,
TEL-03,TEL,L2,2,,,,,,
TEL-04,TEL,L2,2,,,,,,
TEL-05,TEL,L3,3,,,,,,
TEL-06,TEL,L3,3,,,,,,
UPD-01,UPD,L1,1,,,,,,
UPD-02,UPD,L1,1,,,,,,
UPD-03,UPD,L2,2,,,,,,
UPD-04,UPD,L2,2,,,,,,
UPD-05,UPD,L3,3,,,,,,
UPD-06,UPD,L3,3,,,,,,
IPC-01,IPC,L1,1,,,,,,
IPC-02,IPC,L1,1,,,,,,
IPC-03,IPC,L2,2,,,,,,
IPC-04,IPC,L2,2,,,,,,
IPC-05,IPC,L3,3,,,,,,
IPC-06,IPC,L3,3,,,,,,
NAT-01,NAT,L1,1,,,,,,
NAT-02,NAT,L1,1,,,,,,
NAT-03,NAT,L2,2,,,,,,
NAT-04,NAT,L2,2,,,,,,
NAT-05,NAT,L3,3,,,,,,
NAT-06,NAT,L3,3,,,,,,
```

## Spreadsheet calculations

Import the CSV starting at cell A1 and store summary formulas outside A1:J37. Format recall cells as percentages. In spreadsheet locales with different formula separators, translate the separators without changing the arithmetic.

First calculate a validity cell for each product. The following KAVIA expression is TRUE only when all 36 grades are numeric allowed values and every positive grade has both a locator and an excerpt. For Mintlify, substitute H for E, I for F, and J for G.

```excel
=AND(COUNT(E2:E37)=36,COUNTIF(E2:E37,0)+COUNTIF(E2:E37,0.5)+COUNTIF(E2:E37,1)=36,COUNTIFS(E2:E37,">0",F2:F37,"")=0,COUNTIFS(E2:E37,">0",G2:G37,"")=0)
```

Place this KAVIA validity formula in L2 and the Mintlify validity formula in M2. This check validates input shape, not factual truth: reviewers must verify excerpts and record reasons for zero grades separately.

Use the following guarded KAVIA formulas. For Mintlify, replace L2 with M2 and E2:E37 with H2:H37. The area example selects CFG; replace the area code for other subsystems.

```excel
Strict recall:
=IF($L$2,COUNTIF(E2:E37,1)/36,"Pending or invalid")

Fractional recall:
=IF($L$2,SUM(E2:E37)/36,"Pending or invalid")

Weighted recall:
=IF($L$2,SUMPRODUCT(D2:D37,E2:E37)/72,"Pending or invalid")

L1 recall:
=IF($L$2,SUMIF(C2:C37,"L1",E2:E37)/12,"Pending or invalid")

L2 recall:
=IF($L$2,SUMIF(C2:C37,"L2",E2:E37)/12,"Pending or invalid")

L3 recall:
=IF($L$2,SUMIF(C2:C37,"L3",E2:E37)/12,"Pending or invalid")

CFG area recall:
=IF($L$2,SUMIF(B2:B37,"CFG",E2:E37)/6,"Pending or invalid")

CFG weighted recall:
=IF($L$2,SUMPRODUCT((B2:B37="CFG")*D2:D37*E2:E37)/12,"Pending or invalid")
```

Before using the workbook, verify that D2:D37 sums to 72, that each level occurs 12 times, that each area occurs six times, and that all 36 IDs occur exactly once. Grades from an incomplete or noncompliant run must not appear as finalized comparative results.

## Arithmetic control fixtures

Apply these fixtures to a disposable copy of the workbook, not to the retained evidence scorecard. Supply dummy nonempty locators and excerpts for positive grades only to test formula gating; those dummy values are not acceptable evaluation evidence.

| Fixture | Strict recall | Fractional recall | Weighted recall | L1 / L2 / L3 |
| --- | ---: | ---: | ---: | --- |
| Every grade is 1 | 100% | 100% | 100% | 100% / 100% / 100% |
| Every grade is 0 | 0% | 0% | 0% | 0% / 0% / 0% |
| Every grade is 0.5 | 0% | 50% | 50% | 50% / 50% / 50% |
| L1 = 1; L2 = 0.5; L3 = 0 | 33.33% | 50% | 33.33% | 100% / 50% / 0% |
| Any blank or disallowed grade | Pending or invalid | Pending or invalid | Pending or invalid | Pending or invalid |
| Any positive grade lacks evidence | Pending or invalid | Pending or invalid | Pending or invalid | Pending or invalid |

Source existence, unique-ID counts, total weight, and the mixed-fixture arithmetic were checked during benchmark authoring. Spreadsheet application behavior must be verified by the evaluator in the application used for the run; no workbook engine was executed during authoring.

## Review and accuracy audit logs

Keep one review-log record per fact, product, and run, with initial reviewer grades, adjudicated grade, missing decisive requirements, contradiction excerpts where applicable, and adjudication rationale. If a fact is absent, record the reviewed corpus boundary and search approach rather than inventing an excerpt.

Maintain a separate claim audit with claim ID, product/run ID, page locator, exact claim, classification as supported/contradicted/not-established, and source rationale. Report the denominator and sampling method if it is not an exhaustive audit. This log is necessary to interpret recall without rewarding hallucinations.

## Final comparison record

When scoring is complete, publish strict, fractional, weighted, depth-level, and area recall for each run together with precision, contradiction counts, not-established counts, word count, elapsed time, and human effort. State the weighted-recall delta in percentage points. Preserve anonymized exports and adjudication logs so another evaluator can reproduce the result.

For repeated runs, report all attempts, eligible/compliant run counts, mean scores, and ranges. Do not discard failed runs or present unavailable runs as measured zeros. The final narrative must identify whether it compares native generators, composite authoring workflows, or existing published corpora.
