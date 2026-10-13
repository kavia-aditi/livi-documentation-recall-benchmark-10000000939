[CodeWiki](../index.md) / [Historical / extracted](../Historical/index.md) / [Onboarding](index.md)

# LIVI developer guide

## Scope and evidence

This guide describes the eight implementation files supplied in the frozen LIVI bundle. It documents observable interfaces and local behavior, not a complete runnable application. Imported types, helpers, main-process IPC registrations, and dependency implementations are mostly absent. Their behavior is **unknown** unless explicitly established by a supplied caller. Examples illustrate the supplied logic; they are not evidence of missing integration.

Source references used throughout:

- **[Config](../../../src/main/config/loadConfig.ts)** — configuration loading and correction.
- **[Arbiter](../../../src/main/services/projection/transport/TransportArbiter.ts)** — transport candidate selection and snapshots.
- **[Store](../../../src/main/services/telemetry/TelemetryStore.ts)** — telemetry merge state.
- **[Socket](../../../src/main/services/Socket.ts)** — Socket.IO telemetry ingress and broadcasts.
- **[Downloader](../../../src/main/ipc/update/downloader.ts)** — HTTPS download, progress, cancellation.
- **[Updater](../../../src/main/ipc/update/updater.ts)** — update session orchestration.
- **[Preload](../../../src/preload/index.ts)** — renderer-facing Electron IPC bridge.
- **[Feed](../../../native/livi-host-proto/rust/src/feed.rs)** — native media record encoding and incremental framing.

The visible pieces form several separate paths:

1. Disk configuration → defaults and validation → returned configuration and optional rewrite.
2. Injected transport observations → candidate selection or switch intent → transport snapshot.
3. Socket.IO `telemetry:push` → store merge → `change` → Socket.IO `telemetry:update`.
4. Renderer bridge → named IPC channels → main-process handlers whose registration is not supplied.
5. Update URL/feed → temporary download → ready state → platform installer.
6. Native byte chunks → framer buffer → decoded media records.

No supplied code establishes an end-to-end connection between native records and renderer audio/video, or between store changes and Electron telemetry broadcasts. Shared event names alone do not prove those integrations.

## Configuration loading

### Input selection and defaults

`loadConfig(): Config` is synchronous. It begins with an empty partial configuration and selects the live `CONFIG_PATH` if it exists; otherwise it selects `CONFIG_BACKUP_PATH` if that exists. With neither file, it proceeds with defaults. The actual filesystem paths are **unknown**, because `paths` is not supplied. [Config]

Reading and JSON parsing are inside a catch that warns and continues with defaults. A malformed live file does **not** trigger a second attempt from the backup: source selection happens before parsing. Successful backup parsing logs a restoration message; the absent live file subsequently causes a write. JSON parsing alone is not a configuration-schema check. The code expects a property-bearing configuration value; for example, JSON `null` parses successfully but would fail at later property access outside the parse catch. [Config]

The base is imported `DEFAULT_CONFIG`, whose complete fields and values are **unknown**. Two defaults are locally adapted only when the file lacks the corresponding values:

- If `fileConfig.carName === undefined`, take the hostname segment before the first dot, trim it, reject an empty name or case-insensitive `localhost`, and otherwise truncate to `CAR_NAME_MAX`. Rejected names use `DEFAULT_CONFIG.carName`. The maximum’s numeric value is **unknown**. The comment explains that this name identifies the Wi-Fi access point, Bluetooth device, and head unit; those naming consumers are not supplied.
- Only if **both** `projectionWidth` and `projectionHeight` are `undefined`, call `sysfsPanelGeometry()`. If it returns geometry, compute `scale = min(1280 / widthPx, 720 / heightPx, 1)`, then round each scaled dimension to an even integer using `2 * round(dimension / 2)`. Set both projection and cluster default dimensions to those values. Explicit file dimensions remain inputs to validation; supplying just one projection dimension suppresses this panel-derived defaulting.

For example, a reported 1920×1080 panel produces 1280×720 defaults; 800×480 remains 800×480. This calculation preserves aspect before independent even rounding and does not upscale ordinary positive dimensions. Geometry discovery, validity checks, and behavior on probe failure are **unknown**. The call is not locally caught. [Config]

### Validation, correction, and persistence

The loader calls `validate(fileConfig, defaults)`. Merge precedence, type coercion, unknown-field handling, and the complete schema are **unknown**, because the validator and shared configuration types are missing. After it returns, two explicit checks run:

- `wifiPassword.length` must fall within imported `WIFI_PASSWORD_MIN..WIFI_PASSWORD_MAX`, inclusive; otherwise the loader warns and assigns the default password. The numeric bounds and default value are **unknown**.
- `isPagePath(merged.startPage)` must accept the start page; otherwise the loader warns and restores the default start page. Accepted routes and their default are **unknown**.

These checks assume the validator returned appropriately typed fields. They do not supply a general exception boundary. [Config]

A write occurs when the live file does not exist or when `JSON.stringify(fileConfig)` differs from `JSON.stringify(configForFile(merged))`. This is serialized representation comparison, not a dedicated semantic equality test. `writeConfig(merged)` runs before returning the merged configuration. File serialization, backup maintenance, atomicity, permissions, and write-error recovery are **unknown**; no local catch handles validation or persistence errors. Do not assume a corrupt live file is recoverable from the backup in this call. [Config]

## Transport arbitration

### Observations and candidates

`TransportArbiter` receives an imported `ArbiterDeps` object. The visible methods query active transport, wired-session presence and activity, wireless enablement and phone range, and call `onChange()`. The full dependency contract, helper discovery, and the implementation of `candidateEquals` are **unknown**. Candidate objects contain `transport` and `mode`; the supplied constants represent `aa`/`cp` and `wired`/`wireless`. [Arbiter]

`detectedCandidates()` builds an ordered list:

1. AA wired when `hasWiredAaSession()` is true.
2. CP wired when `hasWiredCpSession()` is true.
3. AA wireless when wireless is enabled and either a wireless phone is in range **or a wired AA session is active**.

A present wired session can be a candidate even if not active. CP wireless exists as a constant for describing current state, but this method never offers it as a detected candidate. No supplied logic identifies which physical phone a candidate represents. [Arbiter]

`hasNativeCandidate()` is related but different: it accepts either wired session, or enabled wireless with a phone actually in range. It does not use the active-wired-AA fallback found in `detectedCandidates()`. Therefore it can disagree with whether that list contains AA wireless. [Arbiter]

The private `currentCandidate()` maps active `aa` or `cp` to wired if the corresponding wired-active query is true, otherwise wireless. Any other active value yields no current candidate. An active transport is thus not automatically a detected candidate. [Arbiter]

### Selection and switching rules

The override starts as `null`. `pickPreferred()` applies this precedence:

1. If detection is empty, return `null` immediately. An existing override is retained in this case.
2. If an override compares equal to a detected candidate, return it.
3. Otherwise clear that override.
4. Keep the current candidate if it compares equal to a detected candidate.
5. Otherwise choose the first detected candidate.

This preserves an eligible current session rather than always preferring wired AA. With no current or valid override, discovery order supplies the default priority. Clearing an unavailable override here does not call `onChange()`. `decideNextStart()` wraps the result as `{ kind: 'none' }` or `{ kind: 'start', candidate }`; it does not start a session or check whether that candidate is already running. [Arbiter]

`setOverride(candidate)` accepts a candidate without checking detection, resets native-probe bookkeeping, and calls `onChange()`. `prepareSwitch()` instead requires at least two detected candidates. With fewer, it returns `{ ok: false, target: currentCandidate() }` without changing state. Otherwise it anchors on the current candidate, or the preferred candidate if none is current, and selects the next detected entry cyclically. If the current anchor is absent from the list, its index is `-1`, so the next entry is the first. It stores that target as the override, resets probe bookkeeping, calls `onChange()`, and returns `{ ok: true, target }`. [Arbiter]

For example, with AA wired and CP wired detected, no current transport, and no override, the preferred target is AA wired. Preparing a switch targets CP wired. This is intent only: execution, teardown, retries, and confirmation are **unknown**.

The native-probe fields are initialized and reset to false/zero. No supplied code sets a deadline, performs a probe, or implements deferred-start timing. Do not infer a timeout policy from their names. [Arbiter]

### Snapshot semantics

`getSnapshot()` reports the injected `active` value and:

- `targetTransport` / `targetMode`: override if present, otherwise current candidate, otherwise `null`.
- `switchPending`: an override exists and differs from current, or there is no current.
- `wiredPhoneDetected`: either wired session exists.
- `wirelessPhoneDetected`: wireless is enabled and at least one of phone-in-range, currently wireless-active, or wired-AA-active is true.
- `wiredPhoneActive`: active transport is AA or CP and its wired-active query is true.
- `wirelessPhoneActive`: active transport is AA or CP and is not wired.

The snapshot does not invoke preference selection or validate an override. With detected candidates but no active transport or override, target fields remain `null`; with an unavailable override, they can still report that override until selection clears it. Active wireless can remain true while wireless detection is false if wireless is disabled. These are separate observations, not contradictions automatically repaired by the arbiter. Failure handling for dependency calls is absent locally. [Arbiter]

## Telemetry ingestion and state propagation

### Merge contract

`TelemetryStore` extends `EventEmitter`, starts with `{}`, and exposes `merge`, `snapshot`, `reset`, and typed `on`/`off` helpers for `change(patch, snapshot)`. Runtime payload validation is minimal: nullish values, non-objects, arrays, and objects with no enumerable own keys are ignored. Field schemas and units are **unknown**, because `TelemetryPayload` is not supplied. [Store]

For accepted patches:

- Copy the top-level snapshot.
- Ignore incoming properties whose value is `undefined`.
- For `gps` and `can` only, merge incoming non-null, non-array objects field-by-field with a previous such object; otherwise copy the incoming sub-block.
- Replace every other property directly, including objects and arrays.
- Replace `gps`/`can` too if the incoming value is not considered an object by the local helper.

Despite its name, `_isPlainObject` only checks object/non-null/non-array; it does not inspect prototypes. The merge is not recursive below `gps` or `can`. `null` is a replacement value, not a deletion instruction, and there is no explicit key-deletion API. [Store]

For example:

```ts
store.merge({ gps: { lat: 51, lon: 7 }, speed: 10 })
store.merge({ gps: { lat: 52 }, speed: undefined })
// State retains gps.lon = 7 and speed = 10.
// The emitted second patch contains the merged gps block and ts.
```

If the resulting state has `ts === undefined`, assign `Date.now()`. Once present, that timestamp persists unless explicitly replaced; it is **not** automatically refreshed on every merge. An incoming `ts: null` is not repaired by this undefined-only check. [Store]

The emitted patch contains supplied non-undefined properties, but `gps`/`can` are replaced with their merged blocks. If that emitted patch lacks `ts`, it receives the snapshot timestamp. The store emits `change(emittedPatch, shallowSnapshot)` after updating its internal state. It emits even when values have not changed; a nonempty patch containing only undefined values can still emit and initialize a timestamp. [Store]

`snapshot()` and the event snapshot are fresh **shallow** copies. Nested values can remain shared references; the “read-only view” comment does not establish deep immutability. `reset()` empties state without emitting a change event. There is no persistence, expiration, source ordering, conflict resolution beyond merge order, or listener-error isolation in this file. [Store]

### Socket.IO ingress and egress

Constructing `TelemetrySocket(store, port = 4000)` immediately creates an HTTP server and a Socket.IO server with CORS `{ origin: '*' }`, attaches listeners, and listens on the supplied port. No host argument, TLS configuration, authentication, rate limit, or additional payload validation is shown. Exact network binding and dependency-level transport behavior are **unknown** from this bundle. [Socket]

On connection, a client receives `telemetry:update` only if the store snapshot is nonempty. Incoming `telemetry:push` passes its payload directly to `store.merge()`. Every store `change` broadcasts the **full merged snapshot**, not the emitted patch, to all Socket.IO clients through `io.emit('telemetry:update', snapshot)`. The code defines no application-level acknowledgement or error reply for pushes. [Socket]

HTTP server errors are logged with the port and error message; no retry, fallback port, or rejection of construction is implemented. `disconnect()` unsubscribes from the store, requests Socket.IO closure, and resolves after the HTTP close callback clears both server references. It does not explicitly await the Socket.IO close callback, whose purpose here is logging. Close errors are not surfaced explicitly. `connect()` waits 200 ms and starts a server; it does not wait for the listen callback and has no guard against repeated starts or duplicate subscriptions. [Socket]

The preload listens for Electron `telemetry:update` and offers `telemetry:snapshot` invocation. However, Socket.IO emission does not itself prove Electron emission. The adapter or handler that connects this store to renderer IPC is **unknown**. [Socket; Preload]

## Software updates

### Download helper

`downloadWithProgress(url, dest, onProgress)` returns `{ promise: Promise<void>, cancel: () => void }`. It uses `https.get`, so this implementation does not provide a separate HTTP download path. It creates the destination stream for an accepted response; parent-directory creation is not shown. [Downloader]

A response with status **at least 300** and a `Location` header recursively starts another download using that header verbatim. This is broader than a 3xx-only check. There is no redirect limit, relative-URL resolution, or URL validation in the supplied helper. Cancellation is delegated to the recursive download. Otherwise only status 200 is accepted; another status rejects with `HTTP <status>`. [Downloader]

`content-length` is parsed as an integer, falling back to zero. On every response data chunk, the callback receives cumulative `received`, `total`, and `percent = received / total`, or zero if total is zero. `percent` is a fraction, not a 0–100 value, and is not clamped or checked against actual final size. Response data is piped into the file. Stream `finish` attempts to close the file and then resolves; request, response, and file errors reject unless cancelled. Close exceptions are swallowed. [Downloader]

`cancel()` is idempotent per helper instance. It marks cancellation, calls any redirect cancellation, destroys the request/file, attempts to unlink the destination, and rejects with `aborted` after cleanup. Cleanup failures are swallowed. Ordinary HTTP or stream errors do **not** invoke this cleanup path, so a partial file can remain. No timeout, checksum, signature, resume, retry, size limit, or content-length verification is implemented here. Behavior inside Node’s dependency implementation is not established by these files. [Downloader]

### Session and platform flow

A module-level `updateSession` is shared across all `Updater` instances. Its visible states are `idle`, `downloading`, `ready`, and `installing`, with optional temporary file, cancellation function, and platform. The constructor receives runtime state and services. [Updater]

`perform(event, directUrl?)` follows this sequence:

1. Require `idle`; otherwise throw `Update already in progress`.
2. Emit `{ phase: 'start' }`.
3. Accept only `darwin` and `linux`; otherwise emit `Unsupported platform` and return.
4. Use a truthy direct URL, or fetch `releaseFeedUrl(config.updateNightly === true)` with `User-Agent: LIVI-updater`. Require an OK response, parse it as a release, and take `pickAssetForPlatform(json.assets || []).url`.
5. Require a truthy URL; otherwise fail with `No asset found for platform`.
6. Choose `os.tmpdir()/pcu-<Date.now()>.dmg` on macOS or `.AppImage` on Linux.
7. Set `downloading`, invoke the helper, and forward progress with `phase: 'download'`.
8. After successful download, set `ready` and emit `{ phase: 'ready' }`.

Feed endpoints, release schema validation, architecture/asset matching, version comparison, installer contents, and event delivery destinations are **unknown** because the imported helpers are absent. No supplied code verifies an artifact before declaring it ready. [Updater]

Errors in `perform` reset the session to `idle` and emit an error message; they are caught rather than rethrown. State remains `idle` during feed lookup, so that stage is not protected by the non-idle guard. A rejected concurrent call also resets the shared session through the same catch, without cancelling an existing download. The file does not establish robust exclusivity across overlapping calls. [Updater]

### Abort and install behavior

During download, the stored cancellation wrapper invokes helper cancellation, resets to idle, and emits `Aborted`. `abort()` invokes it when downloading, or attempts to delete the temporary file when ready. Its `finally` always resets to idle and emits `Aborted`, even if no update is active. Consequently downloading abort can emit duplicate `Aborted` events, and the rejected download can cause a later `aborted` error from `perform`. Cancellation cleanup is asynchronous while session reset is immediate. [Updater; Downloader]

`abort()` has no installer-cancellation branch: when invoked in `installing`, it resets session bookkeeping and emits an error but does not stop an installer in the visible code. [Updater]

`install()` requires `ready`, a temporary file, and a platform. Otherwise it catches `No downloaded update ready`, resets idle, and emits an error. With a ready update it sets `installing` and emits that phase:

- **macOS:** attempt `projectionService.stop()`. If it fails, warn and continue. Wait 150 ms, await `installOnMacFromFile(file)`, then return. No local `relaunching` event or restart call follows.
- **Linux:** await `installOnLinuxFromFile(file)`, emit `relaunching`, and await `restartApp(runtimeState, services)`.

On install errors, reset idle and emit the error. On successful return, neither branch explicitly resets session state or removes the temporary file. Restart, replacement mechanics, privilege handling, rollback, and cleanup performed by installers are **unknown**. [Updater]

The preload exposes calls and listeners for updates, but main-process IPC registration and `sendUpdateEvent`/`sendUpdateProgress` implementations are missing. The naming suggests an intended connection, not proof of registration. [Preload; Updater]

## Renderer IPC bridge

### Exposed boundary and calling conventions

The preload exposes two objects with `contextBridge.exposeInMainWorld`: `projection` and `app`. It does not expose a generic arbitrary-channel IPC method. Named wrappers use `invoke` for promise-returning requests and `send` for one-way messages. Main-process validation, authorization, persistence, side effects, and error behavior are **unknown** unless covered by another supplied implementation. TypeScript annotations are not runtime payload validation. [Preload]

Some listeners use `ApiCallback(event, ...args)`, preserving the Electron event argument. Payload-only listeners deliberately omit it. Callbacks are invoked without local catches, so the bridge does not isolate callback exceptions or guarantee continued queue draining after one throws. [Preload]

### Projection API reference

All channel mappings below are directly defined in [Preload].

| API under `projection` | IPC channel and visible contract |
| --- | --- |
| `quit()` | Invoke `quit`, typed `Promise<void>`. |
| `usb.getSysdefaultPrettyName()` | Invoke `get-sysdefault-mic-label`, string result. |
| `settings.get()` / `save(partial)` | Invoke `getSettings` / `save-settings`; `Config` / void results. |
| `audio.listSinks()` / `listSources()` | Invoke `audio:listSinks` / `audio:listSources`; arrays of `{ id, name, isDefault, offline? }`. |
| `ipc.start()` / `stop()` / `restart()` | Invoke `projection-start` / `projection-stop` / `projection-restart`. |
| `ipc.setVisible(visible)` / `sendFrame()` | Invoke `projection-set-visible` / `projection-sendframe`. |
| `ipc.setBluetoothPairedList(text)` | Invoke `projection-bt-pairedlist-set`; `{ ok }` result. |
| `ipc.connectBluetoothPairedDevice(mac)` / `forgetBluetoothPairedDevice(mac)` | Invoke `projection-bt-connect-device` / `projection-bt-forget-device`; `{ ok }`. |
| `ipc.switchTransport()` | Invoke `transport:switch`; `{ ok, active: 'aa' \| 'cp' \| null }`. |
| `ipc.getTransportState()` | Invoke `transport:state`; imported `TransportSnapshot`. |
| `ipc.getDevices()` | Invoke `devices:list`; device array described below. |
| `ipc.selectDevice(id)` / `forgetDevice(id)` | Invoke `devices:select` / `devices:forget`; `{ ok }`. |
| `ipc.cycleSession()` | Invoke `devices:cycle`, void result. |
| `ipc.sendTouch(x, y, action)` | Send `projection-touch` with `{ x, y, action }`. |
| `ipc.sendMultiTouch(points)` | Send `projection-multi-touch` with imported `MultiTouchPoint[]`. |
| `ipc.sendCommand(key)` | Send `projection-command` with a string. |
| `ipc.readMedia()` / `readNavigation()` | Invoke `projection-media-read` / `projection-navigation-read`; unknown payloads. |
| `ipc.setVolume(stream, volume)` | Send `projection-set-volume` with `{ stream, volume }`; stream typed as `music`, `nav`, `voiceAssistant`, or `call`. No range check. |
| `ipc.setVisualizerEnabled(enabled)` | Send `projection-set-visualizer-enabled`, coercing with `!!enabled`. |
| `ipc.requestCluster(enabled)` | Invoke `cluster:request`; `{ ok, enabled }`. |
| `ipc.clusterRepaintNudge()` | Invoke `cluster:repaint-nudge`; `{ ok }`. Comment identifies an intended macOS repaint workaround; implementation is unknown. |
| `ipc.getTelemetrySnapshot()` | Invoke `telemetry:snapshot`; unknown result. |

Device entries require `id` and `status` (`active`, `available`, or `offline`). Optional fields are `name`, `model`, `protocol` (`carplay` or `androidauto`), `lastTransport`, `batteryLevel`, `batteryCharging`, `signalStrength`, and `carrierName`. Units, identity rules, and the device registry implementation are unknown. Touch actions, coordinate conventions, command vocabulary, and volume units are likewise unknown. [Preload]

### Event buffering and subscriptions

The preload installs several listeners before exposing the APIs, allowing some events to precede renderer subscriptions:

| Subscription | Behavior before subscribers and cleanup |
| --- | --- |
| `onUSBResetStatus(cb)` | Listen directly to `usb-reset-start` and `usb-reset-done`; callback receives event/args. Returns removal function for both. No replay. |
| `usb.listenForEvents(cb)` | Queue `usb-event` event/args while no handlers exist. New callback drains the queue, then clears it. Returns callback removal. No queue bound. |
| `settings.onUpdate(cb)` / `onLinkSpeed(cb)` | Direct listeners on `settings` / `link-speed`; event/args callbacks and returned removal functions. No replay. Link-speed comment says payload is null when no dongle/phone is on air; sender is absent. |
| `ipc.onEvent(cb)` | Queue `projection-event` while no handlers exist; first subscriber drains and clears. Returns removal. No queue bound. |
| `ipc.onAudioChunk(handler)` / `offAudioChunk(handler)` | Payload-only `projection-audio-chunk`; one handler, replaced by a new registration. Without it, keep newest 32 chunks, dropping oldest. Registration drains and clears. Removal only clears a matching handler; no returned unsubscribe. |
| `ipc.onClusterResolution(handler)` | Payload-only `cluster-video-resolution`; cache latest value and replay on registration if non-nullish. Multiple handlers; returns removal. |
| `ipc.onTelemetry(handler)` / `offTelemetry(handler)` | Payload-only `telemetry:update`; multiple handlers, no buffering/replay. Explicit matching-handler removal; registration returns void. |
| `app.onMediaKey(handler)` | Payload-only `app:media-key`; accept only nonempty strings. Queue without bound while no handlers; first subscriber drains and clears. Returns removal of the matching entry. |
| `app.onUpdateEvent(cb)` / `onUpdateProgress(cb)` | Direct payload-only listeners on `update:event` / `update:progress`; return removal functions. No buffering/replay. |

USB and projection queues preserve arrival order but are not per-subscriber history: after the queue is drained, later subscribers get only new events. Audio buffering is chunk-count bounded, not byte bounded. Cluster caching retains the last payload without a visible disconnect reset. Telemetry consumers can request a snapshot separately, but the file supplies no atomic snapshot-and-subscribe operation. [Preload]

Example renderer lifecycle:

```ts
const onTelemetry = (payload: unknown) => renderTelemetry(payload)
window.projection.ipc.onTelemetry(onTelemetry)
const snapshot = await window.projection.ipc.getTelemetrySnapshot()
renderTelemetry(snapshot)
// At teardown:
window.projection.ipc.offTelemetry(onTelemetry)
```

`renderTelemetry` is illustrative application code. Snapshot ordering relative to updates and renderer global type declarations are unknown.

### App API reference

`app.platform` copies `process.platform`. `app.compositor` is true only when `process.env.LIVI_COMPOSITOR === '1'`; other values, including absence, yield false. This exposes a flag, not proof of compositor startup behavior. [Preload]

| API under `app` | IPC channel and visible contract |
| --- | --- |
| `getVersion()` | Invoke `app:getVersion`; string. |
| `listDisplayModes()` | Invoke `app:listDisplayModes`; string array. |
| `listWifiChannels()` / `listWifiCountryCodes()` / `listWifiInterfaces()` / `listBtAdapters()` | Invoke corresponding `app:` channel with the same method name; number array for channels, string arrays otherwise. |
| `dongleRadios()` | Invoke `app:dongleRadios`; `{ wifi: boolean \| null, bt: boolean \| null }`. |
| `switchDongleRadio(radio, on)` | Invoke `app:switchDongleRadio` with separate arguments; radio typed `wifi` or `bt`. |
| `getLatestRelease()` | Invoke `app:getLatestRelease`; `{ version?, url? }`. |
| `performUpdate(imageUrl?)` / `beginInstall()` / `abortUpdate()` | Invoke `app:performUpdate` / `app:beginInstall` / `app:abortUpdate`; void promises. |
| `customPageUrl()` / `customIconUrl()` | Invoke `app:customPageUrl` / `app:customIconUrl`; string or null. |
| `quitApp()` / `restartApp()` | Invoke `app:quitApp` / `app:restartApp`. |
| `openExternal(url)` | Invoke `app:openExternal`; `{ ok, error? }`. No URL restriction shown locally. |
| `notifyUserActivity()` | Send `app:user-activity` without payload. |
| `reportPath(path)` | Send `ui:path` with the path string. |
| `broadcastMediaKey(command)` | Send `app:media-key` with command; input validation is not shown on this send path. |

Update event payloads are typed `{ phase: string, message?: string }`; progress payloads permit optional `phase`, `percent`, `received`, and `total`. The bridge does not enforce a phase enumeration. Do not assume calling `broadcastMediaKey` locally invokes subscribers: the main-process rebroadcast implementation is absent. [Preload]

## Native media framing

### Record layout

`feed.rs` provides `Record { kind: u8, id: u32, ts: u64, payload: Vec<u8> }`, `encode(kind, id, ts, payload)`, and a buffered `Framer` with `new`, `push`, and `next_record`. Multibyte integers use Rust `to_ne_bytes` / `from_ne_bytes`: **native host byte order**, not fixed network byte order. Cross-endian interoperability is not established. [Feed]

| Byte offset | Size | Field |
| --- | --- | --- |
| 0 | 4 | Unsigned length of the following header plus payload. |
| 4 | 1 | Kind. |
| 5 | 4 | Stream/plane ID, native endian. |
| 9 | 8 | Timestamp, native endian. |
| 17 | Variable | Payload. |

`HEAD_BYTES = 13`, `LEN_BYTES = 4`; total record size is `17 + payload.len()`. Although the opening comment calls the prefix “payload length,” the encoder actually writes `13 + payload.len()`, including the record header and excluding the prefix. For a three-byte payload the prefix is 16, and the whole record is 20 bytes. [Feed]

Kinds are:

- `1`, `KIND_VIDEO`: one video access unit for the named plane.
- `2`, `KIND_AUDIO`: samples for the named audio stream.
- `3`, `KIND_VIDEO_START`: comment specifies a one-byte codec announcement, 0 for H.264 and 1 for H.265.
- `4`, `KIND_MIC`: captured microphone samples for a requested tap.

The parser does not validate kinds or the codec announcement’s length/value. Timestamp units and clock origin, ID allocation, audio sample format, codec packet details, and microphone tap routing are **unknown**. [Feed]

### Incremental decoding and malformed data

`push(chunk)` appends bytes without attempting decoding. `next_record()`:

1. Returns `None` if fewer than four bytes are buffered.
2. Reads the declared length.
3. Returns `None` if the entire prefix-plus-declared-body is not buffered.
4. If the complete body is shorter than 13 bytes, drains it and continues looking for another record.
5. Otherwise reads kind, ID, timestamp, copies payload, drains that record, and returns it.

Consumers should call repeatedly until `None` to drain multiple records from one chunk. Incomplete records remain buffered; empty payloads are valid. A short malformed body is silently skipped, not returned as an error. A large bogus length can hold up following bytes indefinitely because there is no maximum-length check, timeout, or resynchronization marker. Unknown kinds are returned unchanged. [Feed]

`encode` casts header-plus-payload length to `u32` without an explicit overflow check. Framer buffering has no visible cap; payload extraction allocates a copy. Allocation failure, extreme-size portability, and end-of-stream truncation handling are not addressed by this module. It does not decode media or enforce stream start/stop sequencing. [Feed]

The file includes five tests: round-trip record, reassembly from five-byte chunks, ordered extraction of several records, skipping a short body, and empty payload acceptance. These test definitions are evidence of intended framing cases, not evidence that a full native application was built or run. [Feed]

## Boundaries to preserve when extending LIVI

Keep the supplied responsibilities separate: configuration selects and corrects inputs; arbitration reports intent; the store merges state; Socket.IO broadcasts snapshots; preload maps renderer calls to channels; updater orchestrates imported operations; and framing handles bytes, not media semantics.

The following remain **unknown from this bundle**: complete default configuration and schemas, validation/persistence internals, transport execution and dependency equality, main IPC registration and renderer telemetry adapters, update feed/asset/install/restart internals, runtime security policy, media consumers, and deployment/build wiring. New documentation or implementations should obtain those sources before asserting guarantees about them.
