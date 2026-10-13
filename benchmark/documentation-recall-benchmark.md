**# KAVIA and Mintlify Documentation Recall Benchmark**



[CodeWiki]\(../../index.md) / [Quality]\(../index.md) / [Other Quality]\(index.md)



**## Purpose and status**



This benchmark measures how much source-verifiable information appears in documentation produced or published through KAVIA and Mintlify. Documentation depth means coverage of mechanisms, conditions, boundaries, and failure behavior, not page count, prose length, or visual polish. Recall is measured against an independently defined reference inventory, rather than against another tool's documentation.



The deliverable is a ready-to-administer benchmark protocol with a 36-fact LIVI pilot, an [evaluator-only reference set]\(documentation-recall-reference.md), and a [blank comparison scorecard]\(documentation-recall-scorecard.md). Neither product has been evaluated in this invocation. Product versions, available generation workflows, output documents, and measured scores remain unknown. No winner is claimed.



The pilot is grounded in LIVI commit \`fa5e341d76990a53f13cb7aea6431e17bf5504b3\`. The inspected tracked working tree had no reported changes; an untracked \`.kavia/\` directory was present and is excluded from benchmark inputs. The reference inventory is deliberately bounded and is not an exhaustive inventory of LIVI.



**## What the comparison does and does not measure**



The primary metric is weighted factual recall in the exported documentation corpus. Strict recall, fractional recall, depth-level recall, and subsystem recall explain the result. A correct source path alone does not establish recall: the exported documentation must actually describe the fact.



This benchmark does not measure runtime latency, application performance, automated test coverage, vendor search quality, or an assistant's ability to answer questions while reading source code. It also does not assume that KAVIA and Mintlify provide equivalent code-ingestion or generation features. Record the exact workflow evaluated, not merely the vendor name. A model or integration chosen by the evaluator is part of that workflow and must be disclosed.



**### Track A: source-conditioned documentation generation**



Use this track to compare the depth of documentation generated from identical source inputs. Both workflows must be able to consume the same source bundle, receive the same instruction, and export a frozen documentation corpus. If a product requires an external generator, record its identity and label the result as a composite workflow. Do not compare a composite workflow against a native workflow as though the difference arose solely from the publishing platform.



If either workflow cannot ingest the source bundle or generate documentation under the agreed conditions, mark it ineligible for Track A. This is not zero recall and is not evidence that its published documentation is shallow. Do not silently substitute manually authored pages for one workflow.



**### Track B: published-corpus coverage**



Use this track when the aim is to measure documentation already available in each platform. Freeze each corpus, record its originating workflow, audience, authoring history, and revision, and score only its documented content. Different authoring effort and source access are confounders that must be reported.



Hosting the same Markdown in both systems is a useful export-fidelity control, but it is not a documentation-generation comparison. Its expected factual recall is identical unless content is omitted or altered. Track A and Track B results must be reported separately.



**## Frozen LIVI pilot input**



The Track A bundle contains exactly the following eight implementation files at the pinned revision. Preserve their project-relative paths and source bytes. Dependency implementations outside this bundle are out of scope; the instruction must allow unknowns rather than require inference about unread dependencies.



\| Area | Source files | Facts |

\| --- | --- | ---: |

\| Configuration | \`src/main/config/loadConfig.ts\` | 6 |

\| Transport arbitration | \`src/main/services/projection/transport/TransportArbiter.ts\` | 6 |

\| Telemetry | \`src/main/services/telemetry/TelemetryStore.ts\`; \`src/main/services/Socket.ts\` | 6 |

\| Software updates | \`src/main/ipc/update/updater.ts\`; \`src/main/ipc/update/downloader.ts\` | 6 |

\| Renderer IPC bridge | \`src/preload/index.ts\` | 6 |

\| Native media framing | \`native/livi-host-proto/rust/src/feed.rs\` | 6 |



Only these files enter the controlled source bundle. Do not include this benchmark, its reference set, its scorecard, existing CodeWiki pages, or prior generated documents. Disable repository-wide retrieval if it would expose evaluator artifacts. If isolation is not possible, disclose contamination and do not treat the result as a clean Track A comparison.



Before a real run, export the bundle from the pinned revision and record a cryptographic checksum for each file. This invocation verified file existence and denominator arithmetic, but did not create an archived bundle or compute its checksums.



**## Reference construction and depth levels**



The reference set has six areas with two facts at each of three depth levels. L1 identifies an interface or principal behavior and has weight 1. L2 describes an implementation mechanism or selection rule and has weight 2. L3 captures a boundary, exceptional condition, or easily missed limitation and has weight 3. These labels describe required specificity, not how many words an answer should contain.



There are 12 facts per depth level, 36 facts overall, and a total weight of 72. Each area has six facts and weight 12. These equal strata prevent the larger preload file from dominating simply because it contains more interfaces.



The reference set was derived from implementation code before inspecting either product's output. It remains a proposed gold set until an independent reviewer verifies the source anchors. Freeze it before generation. Do not add facts after seeing which product covers them. For an extended study, construct new inventories from independent modules and repositories using the same pre-output review process; do not treat this pilot as a representative sample of all software projects.



**## Controlled generation procedure**



The suggested default run envelope is one initial generation pass, no fact-specific follow-up prompts, a 30-minute end-to-end limit, and a maximum export of 6,000 documentation words per workflow. These are benchmark parameters, not vendor limits or industry standards. Agree and record the envelope before running either tool. Do not change it based on observed performance.



If an export exceeds the limit, mark the run noncompliant and retain its raw artifacts; do not truncate selectively. Source access, elapsed time, configured output budget, manual intervention, and tool/model settings must be equal where technically possible. Record any unavoidable differences. Necessary product-specific setup is permitted, but injecting explanatory text, gold facts, or handwritten documentation is not.



Use the following prompt verbatim for eligible workflows. It contains no reference answers or fact identifiers.



\`\`\`text

Create developer documentation for the attached LIVI source bundle.

Cover configuration loading, transport arbitration, telemetry ingestion

and state propagation, software updates, the renderer IPC bridge, and

native media framing. Explain interfaces, data flow, selection and state

rules, defaults, boundaries, and failure behavior supported by the files.

Use readable prose, examples where useful, and source-file citations.

Do not infer behavior of dependency implementations that were not supplied.

Mark information that cannot be established from the bundle as unknown.

Keep the exported documentation within 6,000 words. Generate in one pass.

\`\`\`



Apply the same word-budget substitution to both prompts if a different budget was agreed. Maintain an access log where available. Freeze all exported text, examples, diagrams, tables, and linked generated pages at completion; only pages in the declared export corpus can receive credit. Vendor UI chrome, ordinary source-code viewers, and files linked but not exported do not count as documentation evidence.



Run three independent generations per eligible workflow where possible, using the same inputs and envelope with fresh sessions. Label them K1 through K3 and M1 through M3. Retain failures and noncompliant runs rather than silently replacing them. If only one run is feasible, report it as a single-run pilot without uncertainty claims.



An eligible run that returns no documentation within the envelope has zero recalled facts and a recorded generation failure. An inaccessible product, missing account, unsupported workflow, or absent export is not a measured zero; report it as unavailable or ineligible.



**## Evidence and grading**



Two reviewers should independently score anonymized corpora A and B without seeing vendor identity. The reviewers may inspect the source bundle and frozen reference set, but may not generate missing explanations for a corpus. Reconcile disagreements with an adjudicator and retain both initial grades and the adjudicated grade.



For each fact, record a grade, an exact documentation excerpt, and an export-relative page or section locator. If more than one passage is needed, retain all relevant excerpts and locators. Equivalent wording is acceptable. A example or diagram can earn credit if its documented meaning is explicit and correct; copying raw implementation without explaining its behavior does not earn credit.



\| Grade | Meaning |

\| ---: | --- |

\| 1 | The documentation states the complete reference behavior, including its decisive condition or boundary, without contradiction. |

\| 0.5 | It states a substantive correct part but omits a required condition or detail. A generic topic mention is insufficient. |

\| 0 | The fact is absent, too vague, materially incorrect, or contradicted elsewhere in the corpus. |



The reference tables identify decisive requirements. For example, documenting a bounded audio queue but omitting the bound and eviction direction can earn partial credit; merely naming audio buffering cannot. A contradiction about the same fact forces zero even if another passage is correct. Document the contradiction explicitly rather than averaging the passages.



Do not double-credit repeated explanations. Each reference fact contributes once per corpus. Do not award credit for an evaluator's successful search of the implementation. Pending and blank grades are unscored, not zero.



**## Metrics and calculations**



For fact i, let its adjudicated grade be g_i in {0, 0.5, 1} and its weight be w_i in {1, 2, 3}. Calculate the following only after all 36 facts have been graded.



\`\`\`text

Strict recall = count(g_i = 1) / 36

Fractional recall = sum(g_i) / 36

Weighted recall = sum(w_i \* g_i) / 72



Depth recall for level L = sum(g_i where level = L) / 12

Area recall for area A = sum(g_i where area = A) / 6

Area weighted recall = sum(w_i \* g_i where area = A) / 12



Comparison delta in percentage points =

&#x20;   100 \* (KAVIA weighted recall - Mintlify weighted recall)

\`\`\`



Weighted recall is the primary measure because missing boundaries costs more than missing overview statements. Always show L3 recall alongside it: a high overall score must not hide shallow edge-case documentation. Strict recall makes full coverage visible, while fractional recall distinguishes incomplete documentation from total absence.



An all-correct corpus scores 100% on all recall metrics; an all-absent corpus scores 0%. If every L1 fact is complete, every L2 fact is partial, and every L3 fact is absent, strict recall is 33.33%, fractional recall is 50%, weighted recall is 33.33%, and L1/L2/L3 recall is 100%/50%/0%. These arithmetic controls were checked during authoring; they are not product results.



**### Accuracy guardrail**



Recall alone can reward lengthy documentation containing extra unsupported claims. Separately atomize and audit all factual claims about the six scoped areas in each corpus, including claims not in the gold set. Count supported, contradicted, and not-established claims against the source bundle.



Report supported-claim precision as supported claims divided by all audited claims, and report contradiction and not-established counts separately. Split compound claims before classifying them. If exhaustive auditing is infeasible, report a clearly labeled accuracy sample with its selection method and sample size; do not present sampled precision as corpus-wide precision.



No truth should be asserted about unread dependencies. Such claims are not established by this benchmark rather than automatically false. Avoid declaring a practical winner on recall alone if the higher-recall corpus has materially worse precision or substantially more contradictions. Any numerical acceptance threshold must be agreed before scoring; this benchmark imposes no universal pass threshold.



**## Run metadata and reporting**



Each run needs an identifier, track, UTC date, source revision and checksums, reference-set version, product and workflow versions, model identity where disclosed, product configuration, exact input prompt, source-access policy, elapsed time, output word count, setup and editing effort, export checksum, corpus location, reviewer identities, adjudication status, and eligibility/compliance status. Record undisclosed values as unknown rather than inventing them.



Use the scorecard for per-fact evidence and the following report shape. A dash means not measured, never zero.



\| Measure | KAVIA | Mintlify |

\| --- | --- | --- |

\| Workflow, version, and track | Pending | Pending |

\| Eligible/compliant runs and attempted runs | Pending | Pending |

\| Strict recall | — | — |

\| Fractional recall | — | — |

\| Weighted recall, primary | — | — |

\| L1 / L2 / L3 recall | — | — |

\| Six area recalls | — | — |

\| Supported-claim precision and audit size | — | — |

\| Contradicted / not-established claims | — | — |

\| Output words / elapsed time / human effort | — | — |



For repeated runs, publish every run score plus the arithmetic mean and minimum-to-maximum range for each workflow. Report the difference of mean weighted recalls and explain protocol deviations. Three runs on the same small fact set do not establish statistical significance or general vendor superiority.



**## Administration checklist and limitations**



The benchmark owner freezes the source bundle, reviewed gold set, product workflows, budget, and isolation policy before generation. The generation operator captures immutable outputs without the answer key. Reviewers score anonymized corpora with excerpts. The adjudicator finalizes all facts, and the reporting owner publishes metrics together with failures, deviations, and accuracy findings.



The main limitations are the small single-repository sample, deliberate rather than random fact selection, subjective partial-credit judgments, and differences between generation and publishing workflows. Scores measure recall against this inventory only. General claims about KAVIA or Mintlify require broader repositories, independent gold-set authors, and equivalent workflows.



**## Source evidence**



The pilot facts were grounded directly in the eight implementation files listed under the frozen input section. The accompanying reference set supplies relative links and symbol anchors for each area. No vendor capability claim or product score was inferred from repository documentation.
