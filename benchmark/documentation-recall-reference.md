**# Documentation Recall Reference Set**



[CodeWiki]\(../../index.md) / [Quality]\(../index.md) / [Other Quality]\(index.md)



**## Evaluator-only status and scope**



This is reference set \`LIVI-DR-1.0\`, grounded in commit \`fa5e341d76990a53f13cb7aea6431e17bf5504b3\`. It belongs to the [KAVIA and Mintlify benchmark]\(documentation-recall-benchmark.md). It contains 36 proposed gold facts across six areas, with two L1, two L2, and two L3 facts per area. Independent source review is required before a production evaluation.



Do not provide this page to either documentation-generation workflow or allow it into repository retrieval. Its linked questions are evaluator probes, not generation instructions. Each row is one scored fact; multiple clauses define the completeness of that fact and do not create extra denominator entries. L1 has weight 1, L2 weight 2, and L3 weight 3. The total weight is 72.



Full credit requires all decisive requirements in a row. Partial credit requires at least a substantive correct component, with no material contradiction. Generic topic mentions receive zero. Apply the shared rubric and record exported-document evidence in the [scorecard]\(documentation-recall-scorecard.md).



**## Configuration loading**



Source: [\`src/main/config/loadConfig.ts\`]\(../../../../src/main/config/loadConfig.ts). Anchors: \`loadConfig\`, \`carNameFromHost\`, \`source\`, \`needWrite\`.



\| ID | Level | Evaluator probe | Required reference behavior |

\| --- | --- | --- | --- |

\| CFG-01 | L1 | How is loaded configuration checked before use? | \`loadConfig\` passes the parsed configuration and selected defaults to \`validate\` and returns the resulting configuration after subsequent corrections. |

\| CFG-02 | L1 | When is corrected configuration written? | It calls \`writeConfig\` when the live configuration does not exist or the loaded configuration differs from \`configForFile(merged)\` under JSON serialization. |

\| CFG-03 | L2 | Which configuration source takes precedence? | The live file is selected if it exists; otherwise the backup file is selected if it exists; otherwise no file is read. |

\| CFG-04 | L2 | How is an absent car name defaulted? | When the file's \`carName\` is undefined, the first hostname component is trimmed and limited to \`CAR_NAME_MAX\`; empty or case-insensitive \`localhost\` falls back to the default car name. |

\| CFG-05 | L3 | When and how does panel geometry affect defaults? | Panel geometry is consulted only when both projection dimensions are undefined. If available, dimensions are scaled by \`min(1280/width, 720/height, 1)\`, rounded to multiples of two, and used as defaults for both projection and cluster dimensions. This is not a claim about later validator behavior. |

\| CFG-06 | L3 | Does malformed live JSON trigger backup parsing? | No. Source selection happens before parsing; a parse failure warns and leaves the file configuration empty for defaulting, without trying the backup in that invocation. |



The distinctions between a missing file and an unreadable selected file are essential. Do not credit a generic statement that every configuration failure restores the backup.



**## Transport arbitration**



Source: [\`src/main/services/projection/transport/TransportArbiter.ts\`]\(../../../../src/main/services/projection/transport/TransportArbiter.ts). Anchors: \`detectedCandidates\`, \`pickPreferred\`, \`getSnapshot\`, \`prepareSwitch\`.



\| ID | Level | Evaluator probe | Required reference behavior |

\| --- | --- | --- | --- |

\| TRN-01 | L1 | What is the detected-candidate ordering? | \`detectedCandidates\` appends wired Android Auto, wired CarPlay, and then wireless Android Auto when their conditions hold. It does not append a wireless CarPlay candidate. |

\| TRN-02 | L1 | What does the snapshot use as its intended target? | \`getSnapshot\` uses the override when present, otherwise the current candidate, and reports the intended transport and mode or null if neither exists. |

\| TRN-03 | L2 | When is an override preferred? | With a nonempty detected list, a matching override wins; a nonmatching override is cleared before ordinary preference selection. |

\| TRN-04 | L2 | What happens without a valid override? | A currently active candidate is retained if it is in the detected list; otherwise the first detected candidate is selected. |

\| TRN-05 | L3 | Is an unavailable override always cleared? | No. \`pickPreferred\` returns null immediately when the detected list is empty, before checking or clearing an existing override. |

\| TRN-06 | L3 | Can wireless Android Auto be offered without a phone-in-range signal? | Yes. Wireless must be enabled, and either a wireless phone is in range or a wired Android Auto session is active. The active wired session can satisfy the second condition. |



The availability list is not the same as every candidate constant defined in the file. Credit requires the documented behavior of the named method, not a generalized claim about all projection sessions.



**## Telemetry ingestion and state propagation**



Sources: [\`src/main/services/telemetry/TelemetryStore.ts\`]\(../../../../src/main/services/telemetry/TelemetryStore.ts) and [\`src/main/services/Socket.ts\`]\(../../../../src/main/services/Socket.ts). Anchors: \`merge\`, \`snapshot\`, \`setupListeners\`, \`TelemetryEvents\`.



\| ID | Level | Evaluator probe | Required reference behavior |

\| --- | --- | --- | --- |

\| TEL-01 | L1 | How do clients submit telemetry? | The Socket.IO listener receives \`telemetry:push\` and forwards its payload to \`store.merge\`. |

\| TEL-02 | L1 | What is broadcast after a store change? | The socket server broadcasts the merged snapshot, not merely the incoming patch, to clients using \`telemetry:update\`. |

\| TEL-03 | L2 | Which nested blocks are merged field by field? | \`gps\` and \`can\` merge fields when incoming values are non-null, non-array objects; other keys use replacement semantics. |

\| TEL-04 | L2 | How are undefined values in a patch handled? | Top-level keys whose incoming values are undefined are skipped both during state merging and while constructing the emitted patch. This is not recursive removal of undefined nested fields. |

\| TEL-05 | L3 | Is a fresh timestamp generated on every merge? | No. \`Date.now()\` is assigned only when the resulting snapshot has no \`ts\`; an existing timestamp remains unless the incoming patch changes it. |

\| TEL-06 | L3 | Is the snapshot deeply copied? | No. \`snapshot()\` returns a fresh shallow object; nested objects retain shared references rather than being recursively cloned. |



Do not infer timestamp freshness from the event frequency, or deep immutability from the comment describing a read-only view. The implementation is authoritative.



**## Software updates**



Sources: [\`src/main/ipc/update/updater.ts\`]\(../../../../src/main/ipc/update/updater.ts) and [\`src/main/ipc/update/downloader.ts\`]\(../../../../src/main/ipc/update/downloader.ts). Anchors: \`Updater.perform\`, \`Updater.install\`, \`downloadWithProgress\`, \`cancel\`.



\| ID | Level | Evaluator probe | Required reference behavior |

\| --- | --- | --- | --- |

\| UPD-01 | L1 | Which platforms pass the updater's platform gate? | \`perform\` accepts \`darwin\` and \`linux\`; other platforms receive an error event with message \`Unsupported platform\`. |

\| UPD-02 | L1 | Does completing a download immediately install it? | No. After the download promise completes, the state becomes \`ready\` and a ready event is emitted. Installation is a separate operation requiring ready state, a temporary file, and a platform. |

\| UPD-03 | L2 | How is the download URL selected? | A supplied direct URL is used; otherwise the updater requests \`releaseFeedUrl(config.updateNightly === true)\` and selects an asset URL through \`pickAssetForPlatform\`. This does not establish the unread helper's full selection algorithm. |

\| UPD-04 | L2 | What unit does download progress use? | \`percent\` is computed as received bytes divided by total bytes, a fraction rather than a 0-to-100 percentage; it is zero when the total is unavailable or zero. |

\| UPD-05 | L3 | What does download cancellation attempt to clean up? | It invokes any redirected download's cancellation, destroys the current request and file stream, attempts to unlink the destination if present, and rejects with an \`aborted\` error after cleanup. Do not generalize this cleanup to every error path. |

\| UPD-06 | L3 | Does a projection-stop failure prevent macOS installation? | No. The macOS branch catches and warns about a projection-stop failure, waits 150 milliseconds, and still calls \`installOnMacFromFile\`. |



This inventory measures documented updater control flow, not whether an installation succeeds on hardware. Dependencies outside the source bundle must remain unknown.



**## Renderer IPC bridge**



Source: [\`src/preload/index.ts\`]\(../../../../src/preload/index.ts). Anchors: \`contextBridge.exposeInMainWorld\`, \`api.ipc\`, \`usbEventQueue\`, \`lastClusterResolution\`, \`audioChunkQueue\`, \`telemetryHandlers\`.



\| ID | Level | Evaluator probe | Required reference behavior |

\| --- | --- | --- | --- |

\| IPC-01 | L1 | Which named browser globals are exposed? | The preload exposes the main projection API under \`projection\` and the application API under \`app\` through \`contextBridge.exposeInMainWorld\`. |

\| IPC-02 | L1 | Does a single-touch submission await an IPC result? | No. \`sendTouch\` uses \`ipcRenderer.send\` on \`projection-touch\` with \`{ x, y, action }\`, rather than an invoke call returning a promise. |

\| IPC-03 | L2 | What happens to USB events before a subscriber exists? | Events are queued when there are no USB handlers; \`listenForEvents\` replays the queue to the registering callback and clears it. Later subscribers do not get that drained history. |

\| IPC-04 | L2 | What does a late cluster-resolution subscriber receive? | The most recently received cluster resolution is retained and immediately replayed by \`onClusterResolution\` when it is non-null. |

\| IPC-05 | L3 | How is unsubscribed visualizer audio bounded? | Without an audio chunk handler, chunks are queued up to 32 entries; adding a 33rd removes the oldest entry with \`shift\`. |

\| IPC-06 | L3 | Does telemetry subscription replay the last telemetry event? | No. Telemetry events fan out only to current handlers, and \`onTelemetry\` merely adds a handler. A separate \`getTelemetrySnapshot\` invokes \`telemetry:snapshot\` for state retrieval. |



Replay behavior is channel-specific. A statement that every preload event is buffered is materially incorrect.



**## Native media framing**



Source: [\`native/livi-host-proto/rust/src/feed.rs\`]\(../../../../native/livi-host-proto/rust/src/feed.rs). Anchors: \`Record\`, \`KIND\_\*\`, \`encode\`, \`Framer::next_record\`.



\| ID | Level | Evaluator probe | Required reference behavior |

\| --- | --- | --- | --- |

\| NAT-01 | L1 | What fields are represented by a decoded record? | A record contains \`kind: u8\`, \`id: u32\`, \`ts: u64\`, and an owned byte-vector payload. |

\| NAT-02 | L1 | What are the declared record kind numbers? | Video is 1, audio is 2, video-start is 3, and microphone samples are 4. These constants do not imply that the framer rejects other kind bytes. |

\| NAT-03 | L2 | What does the four-byte length prefix count? | It counts the 13-byte record header plus the payload, excluding the four-byte length prefix itself. Total wire size is 17 bytes plus payload length. |

\| NAT-04 | L2 | What byte order is used for the integer fields? | Length, stream ID, and timestamp use native-endian integer conversions in both encoding and decoding, not a fixed network byte order. |

\| NAT-05 | L3 | What happens when a record is only partly received? | \`next_record\` returns None when fewer than four prefix bytes or fewer than \`4 + advertised_length\` bytes are buffered, retaining the buffer for later chunks. |

\| NAT-06 | L3 | How are complete records with an undersized header handled? | If the advertised length is below 13 and the full advertised bytes have arrived, the framer drains that prefix and record and continues seeking a later record rather than returning it. |



The file also contains framing tests, but their existence is not itself a recalled behavior. This pilot omits many other facts; coverage beyond these rows may improve documentation usefulness without changing the recall denominator.



**## Reference review record**



The authoring checks confirmed that all eight referenced source files exist, the 36 IDs are unique, each level contains 12 facts, and the weight sum is 72. Independent review, source-bundle checksums, and vendor-document scoring are pending.



When reviewers discover a gold-set defect, record it and version the inventory. Apply the corrected inventory to every compared corpus, or invalidate and repeat the comparison if the original fact influenced generation. Never make a one-sided denominator change.
