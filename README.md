# LIVI Documentation Recall Benchmark

Controlled benchmark repository for comparing KAVIA and Mintlify documentation recall against the same frozen LIVI source bundle.

## Source provenance

The eight implementation files are copied byte-for-byte from `f-io/LIVI` at commit `fa5e341d76990a53f13cb7aea6431e17bf5504b3`.

## Branches

- `main`: evaluator branch containing source + benchmark protocol + gold reference + blank scorecard.
- `kavia`: generation branch containing the source bundle and controlled prompt, with evaluator materials removed.
- `mintlify`: generation branch containing the source bundle and controlled prompt, with evaluator materials removed.

Do not run generation from `main`: the evaluator-only reference set would contaminate the comparison.
