# Plan: Migrate servo usecases from ipc-channel to ipc-channel-mux

## Context

Servo uses `ipc-channel` extensively for inter-process communication. A common pattern is embedding an `IpcSender<T>` inside a message enum variant to implement request-response communication. Each such sender consumes an OS file descriptor when sent over IPC. Under heavy load (many tabs, fetches, DOM queries), this can exhaust Linux's per-process file descriptor limit.

`ipc-channel-mux` v0.7.0 solves this by multiplexing multiple logical subchannels over a single underlying IPC channel. Critically, `SubSender<T>` can be sent over subchannels **without consuming additional file descriptors**, making it ideal for the request-response pattern.

This plan prioritises usecases involving the **transmission of senders**, since these are where ipc-channel-mux provides the most benefit.

## Key constraint: SubReceiver cannot be serialized

`SubReceiver<T>` does **not** implement `Serialize`/`Deserialize` and cannot be sent over channels. One existing usecase sends an `IpcReceiver` in a message:

- `BodyChunkRequest::Extract(IpcReceiver<BodyChunkRequest>)` in [request.rs:303](components/shared/net/request.rs#L303)

This pattern **cannot** be directly migrated and must be excluded or redesigned.

---

## Usecase 1: Script-to-Constellation request-response senders

**Impact: HIGH** — Every script thread sends messages to the constellation, many with embedded response senders.

### What happens today

`ScriptToConstellationMessage` ([from_script_message.rs:546](components/shared/constellation/from_script_message.rs#L546)) has these variants that embed an `IpcSender`:

| Variant | Response type |
|---|---|
| `NewBroadcastChannelRouter` | `IpcSender<BroadcastChannelMsg>` |
| `GetTopForBrowsingContext` | `IpcSender<Option<WebViewId>>` |
| `GetBrowsingContextInfo` | `IpcSender<Option<(BrowsingContextId, Option<PipelineId>)>>` |
| `GetChildBrowsingContextId` | `IpcSender<Option<BrowsingContextId>>` |
| `RemoveIFrame` | `IpcSender<Vec<PipelineId>>` |
| `GetWebGPUChan` | `IpcSender<Option<WebGPU>>` |
| `CreateAuxiliaryWebView` | contains `IpcSender<Option<AuxiliaryWebViewCreationResponse>>` |

Each time one of these messages is sent, a new `ipc::channel()` is created in the script thread, the `IpcSender` half is embedded in the message, and the `IpcReceiver` half is kept to await the response. This creates a new FD pair per request.

### Migration approach

The script-to-constellation channel is created per-pipeline. At pipeline setup time, create a `mux::Channel` and use it for both the main message subchannel and for response subchannels:

1. Replace the `ipc::channel()` calls that create response senders with `channel.sub_channel()` calls, where `channel` is a `mux::Channel` shared by the pipeline.
2. The `SubSender<T>` returned can be embedded in the message variant in place of `IpcSender<T>`.
3. The `SubReceiver<T>` is kept locally for the response (same pattern as today).

### Key files
- [from_script_message.rs](components/shared/constellation/from_script_message.rs) — message enum definitions
- [constellation.rs](components/constellation/constellation.rs) — constellation handles these messages and sends responses
- Script DOM files that create ad-hoc channels (see below)
- [windowproxy.rs](components/script/dom/windowproxy.rs) — `GetBrowsingContextInfo`, `GetChildBrowsingContextId`
- [globalscope.rs](components/script/dom/globalscope.rs) — `GetTopForBrowsingContext`, broadcast channel setup
- [document.rs](components/script/dom/document.rs) — various request-response channels

### Senders created in script DOM code (sample)
These all create `ipc::channel()` and embed the sender in a constellation message:
- [windowproxy.rs:303](components/script/dom/windowproxy.rs#L303), [windowproxy.rs:942](components/script/dom/windowproxy.rs#L942), [windowproxy.rs:961](components/script/dom/windowproxy.rs#L961)
- [script_window_proxies.rs:175](components/script/script_window_proxies.rs#L175)
- [globalscope.rs:1678](components/script/dom/globalscope.rs#L1678), [globalscope.rs:1883](components/script/dom/globalscope.rs#L1883), [globalscope.rs:2078](components/script/dom/globalscope.rs#L2078), [globalscope.rs:2116](components/script/dom/globalscope.rs#L2116), [globalscope.rs:2234](components/script/dom/globalscope.rs#L2234)

---

## Usecase 2: Script-to-Net (CoreResourceMsg) request-response senders

**Impact: HIGH** — Every fetch, cookie query, and history state lookup sends a response sender.

### What happens today

`CoreResourceMsg` ([lib.rs:612](components/shared/net/lib.rs#L612)) has these variants with embedded `IpcSender`:

| Variant | Response type |
|---|---|
| `Fetch` → `FetchChannels::ResponseMsg` | `IpcSender<FetchResponseMsg>` |
| `FetchRedirect` | `IpcSender<FetchResponseMsg>` |
| `GetCookiesForUrl` | `IpcSender<Option<String>>` |
| `GetCookiesDataForUrl` | `IpcSender<Vec<Serde<Cookie>>>` |
| `DeleteCookies` | `Option<IpcSender<()>>` |
| `NewCookieListener` | `IpcSender<CookieAsyncResponse>` |
| `GetHistoryState` | `IpcSender<Option<Vec<u8>>>` |
| `NetworkMediator` | `IpcSender<CustomResponseMediator>` |

Fetch is particularly high-traffic: every network request creates a new IPC channel and embeds the sender in `FetchChannels::ResponseMsg`.

### Migration approach

Same pattern as Usecase 1: share a `mux::Channel` between the script thread and the net resource thread, create subchannels for each request-response pair.

**Exclusion**: `BodyChunkRequest::Extract(IpcReceiver<BodyChunkRequest>)` cannot be migrated because it sends an `IpcReceiver`. This would need a redesign (e.g., send a new `SubSender` instead and reverse the communication direction) but that is out of scope for an initial migration.

### Key files
- [lib.rs](components/shared/net/lib.rs) — `CoreResourceMsg`, `FetchChannels` definitions
- [request.rs](components/shared/net/request.rs) — `BodyChunkRequest`, `RequestBody`
- [filemanager_thread.rs](components/shared/net/filemanager_thread.rs) — `FileManagerThreadMsg` (6 variants with `IpcSender`)
- [resource_thread.rs](components/net/resource_thread.rs) — net thread event loop
- Script fetch code: [fetch.rs:733](components/script/fetch.rs#L733), [body.rs:419](components/script/body.rs#L419)
- [http_loader.rs:698](components/net/http_loader.rs#L698), [fetch/methods.rs:947](components/net/fetch/methods.rs#L947)

---

## Usecase 3: FileManagerThreadMsg response senders

**Impact: MEDIUM** — File/blob operations each create a response channel.

### What happens today

`FileManagerThreadMsg` ([filemanager_thread.rs](components/shared/net/filemanager_thread.rs)) variants with `IpcSender`:

| Variant | Response type |
|---|---|
| `SelectFile` | `IpcSender<EmbedderControlResponse>` |
| `ReadFile` | `IpcSender<FileManagerResult<ReadFileProgress>>` |
| `PromoteMemory` | `IpcSender<Result<Uuid, BlobURLStoreError>>` |
| `AddSliceUrl` | `IpcSender<Result<(), BlobURLStoreError>>` |
| `DecRef` | `IpcSender<Result<(), BlobURLStoreError>>` |
| `RevokeBlobUrl` | `IpcSender<Result<(), BlobURLStoreError>>` |

These are sent via `CoreResourceMsg::ToFileManager`, so they would naturally benefit from the same `mux::Channel` used in Usecase 2.

---

## Usecase 4: WebSocket and media event senders

**Impact: MEDIUM** — Each WebSocket connection creates a sender.

- `FetchChannels::WebSocket { event_sender: IpcSender<WebSocketNetworkEvent>, .. }` — per-WebSocket connection
- Media element channels: [htmlmediaelement.rs:221](components/script/dom/html/htmlmediaelement.rs#L221), [htmlmediaelement.rs:2123](components/script/dom/html/htmlmediaelement.rs#L2123)

These could share the same `mux::Channel` as Usecase 2 (script↔net).

---

## Usecase 5: WebXR session senders

**Impact: LOW** — Few concurrent sessions expected.

- `IpcSender<Frame>` and `IpcSender<Result<Session, Error>>` in [registry.rs](components/shared/webxr/registry.rs) and [session.rs](components/shared/webxr/session.rs)

Lower priority due to low concurrency, but follows the same pattern.

---

## Integration with GenericChannel abstraction

Servo already has a `GenericSender<T>` / `GenericReceiver<T>` abstraction in [generic_channel.rs](components/shared/base/generic_channel.rs) that switches between IPC and crossbeam at runtime. This is used for the main message channels (script↔constellation, etc.) but **not** for the ad-hoc response channels which use raw `IpcSender<T>` directly.

The response sender migration does **not** require changes to GenericChannel. The ad-hoc `ipc::channel()` calls in DOM code would be replaced with `channel.sub_channel()` calls, and the `IpcSender<T>` types in message enums would become `SubSender<T>`.

A future phase could add a `Mux` variant to `GenericSenderVariants`/`GenericReceiverVariants` to fully integrate, but that is not required for the initial sender-transmission usecases.

---

## Recommended migration order

1. **Usecase 1** (Script→Constellation response senders) — Highest value, well-contained message enum, clear sender sites in DOM code.
2. **Usecase 2** (Script→Net response senders) — High value, high traffic (fetch), but slightly more complex due to the `BodyChunkRequest::Extract` exclusion.
3. **Usecase 3** (FileManager senders) — Falls out naturally from Usecase 2 since messages go through `CoreResourceMsg`.
4. **Usecases 4-5** — Lower priority, tackle after the main patterns are proven.

## Verification

- Run the full servo test suite (`mach test-unit`, `mach test-wpt`) to verify correctness
- Run servo with `--multiprocess` flag to exercise IPC paths
- Monitor file descriptor usage under load (e.g., `ls /proc/<pid>/fd | wc -l`) to confirm reduction
- Run with `RUST_LOG=debug` to check for any mux-related error logs
