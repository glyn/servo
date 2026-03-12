# Generic Channels

Servo uses a `generic_channel` abstraction to select between IPC-capable and
in-process channels at runtime, depending on whether Servo is running in
multiprocess mode.

---

## The abstraction

The implementation lives in
[`components/shared/base/generic_channel/`](../components/shared/base/generic_channel/)
and its submodules.

### How the mode is chosen

[`generic_channel.rs`](../components/shared/base/generic_channel.rs) caches the
result of `opts.multiprocess || opts.force_ipc` in a `OnceLock<bool>`:

```rust
fn use_ipc() -> bool {
    *USE_IPC.get_or_init(|| {
        servo_config::opts::get().multiprocess || servo_config::opts::get().force_ipc
    })
}
```

All channel constructors call `use_ipc()` to select the appropriate backend.

### Generic types that switch at runtime

| Type | File | IPC variant | In-process variant |
|---|---|---|---|
| `GenericSender<T>` / `GenericReceiver<T>` | [`generic_channel.rs`](../components/shared/base/generic_channel.rs) | `ipc_channel::ipc::IpcSender/IpcReceiver<T>` | `crossbeam_channel::Sender/Receiver<Result<T, IpcError>>` |
| `GenericOneshotSender<T>` / `GenericOneshotReceiver<T>` | [`oneshot.rs`](../components/shared/base/generic_channel/oneshot.rs) | `ipc_channel::ipc::channel()` | `crossbeam_channel::bounded(1)` |
| `GenericReceiverSet<T>` | [`generic_channelset.rs`](../components/shared/base/generic_channel/generic_channelset.rs) | `ipc_channel::ipc::IpcReceiverSet` | `Vec<crossbeam_channel::Receiver<...>>` |
| `GenericSharedMemory` | [`shared_memory.rs`](../components/shared/base/generic_channel/shared_memory.rs) | `IpcSharedMemory` | `Arc<Vec<u8>>` |
| `GenericCallback<T>` | [`callback.rs`](../components/shared/base/generic_channel/callback.rs) | IPC sender + ROUTER | In-process function pointer (via leaked `Box`) |
| `LazyCallback<T>` / `CallbackSetter<T>` | [`lazy_callback.rs`](../components/shared/base/generic_channel/lazy_callback.rs) | IPC sender + ROUTER | In-process function pointer |

### Public constructor functions

- `channel<T>()` — [`generic_channel.rs:562`](../components/shared/base/generic_channel.rs#L562):
  creates an `IpcSender`/`IpcReceiver` pair in multiprocess mode, or a
  `crossbeam_channel::unbounded()` pair otherwise.
- `oneshot<T>()` — [`oneshot.rs:61`](../components/shared/base/generic_channel/oneshot.rs#L61):
  same logic but uses `crossbeam_channel::bounded(1)` in single-process mode.
- `GenericReceiverSet::new()` — [`generic_channelset.rs:88`](../components/shared/base/generic_channel/generic_channelset.rs#L88):
  creates `IpcReceiverSet` vs `Vec<crossbeam::Receiver>`.

### Serialization guards

Because `GenericSender`, `GenericReceiver`, `GenericCallback`, and
`GenericSharedMemory` are all `Serialize`/`Deserialize` (so they can be sent
over existing IPC channels during incremental porting), they include runtime
checks that error on serialization/deserialization if a Crossbeam/in-process
variant is encountered while `multiprocess` mode is active. See e.g.
[`generic_channel.rs:95-98`](../components/shared/base/generic_channel.rs#L95-L98).

---

## Usage outside the abstraction

A few places directly check `opts::get().multiprocess` to control behaviour
beyond just channel type:

- [`constellation.rs:625`](../components/constellation/constellation.rs#L625) —
  The `BackgroundHangMonitor` is only created in single-process mode; in
  multiprocess mode it is initialised per-content-process instead.
- [`event_loop.rs:117`](../components/constellation/event_loop.rs#L117) —
  `EventLoop` spawning: multiprocess mode calls `spawn_in_process` (which forks
  a subprocess), single-process mode calls `spawn_in_thread`.
- [`constellation.rs:2540`](../components/constellation/constellation.rs#L2540) —
  Service workers: in multiprocess mode they are spawned as a subprocess via
  `spawn_multiprocess`.
- [`servo.rs:747`](../components/servo/servo.rs#L747),
  [`servo.rs:781`](../components/servo/servo.rs#L781) — JS engine and media
  platform are only initialised in the main process (single-process mode);
  content processes initialise them separately.

---

## Where `GenericReceiver` is serialized

`GenericReceiver` implements `Serialize`/`Deserialize` so that a receiver can
itself be transmitted as the payload of another channel message. In multiprocess
mode this serializes the underlying `IpcReceiver` (which is natively
serializable via `ipc-channel`); in single-process mode it serializes a raw
pointer to a cloned `crossbeam` receiver, relying on shared address space.

The following locations embed a `GenericReceiver` inside a
`#[derive(Serialize, Deserialize)]` struct or enum variant, causing it to be
serialized when that message is sent:

### `InitialScriptState` — [`shared/script/lib.rs:352`](../components/shared/script/lib.rs#L352)

```rust
#[derive(Deserialize, Serialize)]
pub struct InitialScriptState {
    pub constellation_to_script_receiver: GenericReceiver<ScriptThreadMessage>,
    ...
}
```

Sent from the constellation to a new script event loop at startup. This is the
primary motivation for `GenericReceiver`'s `Serialize` impl — the constellation
hands the script thread's incoming channel to a content process.

### `ScriptThreadMessage::SetWebGPUPort` — [`shared/script/lib.rs:275`](../components/shared/script/lib.rs#L275)

```rust
SetWebGPUPort(GenericReceiver<WebGPUMsg>),
```

A `GenericReceiver` embedded directly as an enum variant payload, serialized
whenever this `ScriptThreadMessage` is sent over its channel.

### `RendererToCompositorMsg::SendDisplayList` — [`shared/paint/lib.rs:136`](../components/shared/paint/lib.rs#L136)

```rust
SendDisplayList {
    display_list_info_receiver: GenericReceiver<PaintDisplayListInfo>,
    display_list_data_receiver: GenericReceiver<SerializableDisplayListPayload>,
}
```

Two `GenericReceiver`s embedded in a paint message, so the receivers themselves
are transmitted as part of the display list notification.

### `ServiceWorkerUnprivilegedContent` — [`shared/constellation/from_script_message.rs:257`](../components/shared/constellation/from_script_message.rs#L257)

```rust
pub receiver: GenericReceiver<ServiceWorkerMsg>,
```

Sent to a service worker subprocess when spawning it in multiprocess mode.

### `WebGLCommand::BufferData` / `BufferSubData` — [`shared/canvas/webgl.rs:264`](../components/shared/canvas/webgl.rs#L264)

```rust
BufferData(u32, GenericReceiver<GenericSharedMemory>, u32),
BufferSubData(u32, isize, GenericReceiver<GenericSharedMemory>),
```

Receivers for shared memory blobs passed inline as part of WebGL command
messages.

### `PipelineNamespaceRequest` — [`shared/base/id.rs:131`](../components/shared/base/id.rs#L131)

```rust
namespace_receiver: GenericReceiver<PipelineNamespaceId>,
```

Part of the pipeline namespace allocation handshake sent to a new process.

---

## Direct `ipc-channel` usage outside the generic channel abstraction

Many parts of the codebase still use `ipc_channel` types directly rather than
going through the generic channel abstraction. These are channels whose message
types are inherently cross-process (e.g. streams, media events, file I/O
responses) and have not yet been migrated, or where the raw IPC API is needed
for integration with third-party libraries.

### HTTP body streaming — [`shared/net/request.rs`](../components/shared/net/request.rs), [`script/body.rs`](../components/script/body.rs)

The body-chunk streaming protocol between the network thread and script uses
`IpcSender`/`IpcReceiver` directly:

```rust
// shared/net/request.rs
chan: Arc<Mutex<IpcSender<BodyChunkRequest>>>,

// script/body.rs
bytes_sender: Option<IpcSender<BodyChunkResponse>>,
control_sender: Option<IpcSender<BodyChunkRequest>>,
```

`ROUTER.add_typed_route` is used in `body.rs` to bridge these IPC receivers
onto the script thread's task queue.

### Networking message types — [`shared/net/lib.rs`](../components/shared/net/lib.rs)

`IpcSender` is used directly in the core network message enums:

```rust
pub struct CustomResponseMediator {
    pub response_chan: IpcSender<Option<CustomResponse>>,
    ...
}

pub enum FetchChannels {
    ResponseMsg(IpcSender<FetchResponseMsg>),
    WebSocket {
        event_sender: IpcSender<WebSocketNetworkEvent>,
        ...
    },
    FetchRedirect(RequestBuilder, ResponseInit, IpcSender<FetchResponseMsg>),
    GetCookiesForUrl(ServoUrl, IpcSender<Option<String>>, CookieSource),
    ...
}
```

`FetchTaskTarget` is also implemented for `IpcSender<FetchResponseMsg>` and
`IpcSender<WebSocketNetworkEvent>` directly.

### File manager — [`shared/net/filemanager_thread.rs`](../components/shared/net/filemanager_thread.rs), [`net/filemanager_thread.rs`](../components/net/filemanager_thread.rs)

File read responses and blob URL operations use `IpcSender` for their reply
channels:

```rust
// FileManagerThreadMsg variants
ReadFile(IpcSender<FileManagerResult<ReadFileProgress>>, Uuid, ...),
PromoteMemory(RelativePos, IpcSender<Result<Uuid, BlobURLStoreError>>, ...),
RevokeBlobURL(IpcSender<Result<(), BlobURLStoreError>>),
FilePicker(FilePickerRequest, IpcSender<EmbedderControlResponse>),
```

### Media playback — [`media/player/lib.rs`](../components/media/player/lib.rs)

The seek-lock handshake between the media backend and the player controller
uses a raw IPC channel pair:

```rust
pub type SeekLockMsg = (bool, IpcSender<()>);

pub struct SeekLock {
    pub lock_channel: IpcSender<SeekLockMsg>,
}
```

### Broadcast channels — [`constellation/broadcastchannel.rs`](../components/constellation/broadcastchannel.rs)

The constellation's broadcast channel router map stores raw `IpcSender`s
keyed by router ID:

```rust
routers: FxHashMap<BroadcastChannelRouterId, IpcSender<BroadcastChannelMsg>>,
```

### `IpcSend` trait — [`shared/base/lib.rs`](../components/shared/base/lib.rs)

A `IpcSend<T>` trait (analogous to `GenericSend<T>`) is defined for types that
wrap a raw `IpcSender`:

```rust
pub trait IpcSend<T> {
    fn send(&self, _: T) -> Result<(), IpcError>;
    fn sender(&self) -> IpcSender<T>;
}
```

This predates `GenericSend` and is used by `ResourceThreads` and related types
in the networking stack.

### Direct `ROUTER` usage

Several components route raw `IpcReceiver`s onto the script-thread task queue
via `ROUTER.add_typed_route(...)` rather than using the generic channel's
`route_preserving_errors()` helper:

| File | Purpose |
|---|---|
| [`script/body.rs:145`](../components/script/body.rs#L145), [`:442`](../components/script/body.rs#L442) | Body chunk request routing |
| [`script/dom/websocket.rs:303`](../components/script/dom/websocket.rs#L303) | WebSocket DOM events |
| [`script/dom/html/htmlmediaelement.rs:238`](../components/script/dom/html/htmlmediaelement.rs#L238), [`:2162`](../components/script/dom/html/htmlmediaelement.rs#L2162) | Media element image and action events |
| [`script/dom/webxr/xrsession.rs:210`](../components/script/dom/webxr/xrsession.rs#L210) | XR session frame delivery |
| [`script/dom/webxr/xrsystem.rs:245`](../components/script/dom/webxr/xrsystem.rs#L245) | XR system frame receiver |
| [`script/dom/audio/analysernode.rs:120`](../components/script/dom/audio/analysernode.rs#L120) | Audio analyser block delivery |
| [`script/dom/cookiestore.rs:153`](../components/script/dom/cookiestore.rs#L153) | Cookie store responses |
| [`script/dom/document_embedder_controls.rs:141`](../components/script/dom/document_embedder_controls.rs#L141) | Embedder control responses |
| [`script/dom/globalscope.rs:1682`](../components/script/dom/globalscope.rs#L1682), [`:2195`](../components/script/dom/globalscope.rs#L2195), [`:2222`](../components/script/dom/globalscope.rs#L2222) | Broadcast control, profile, and file manager messages |
| [`script/serviceworker_manager.rs:515`](../components/script/serviceworker_manager.rs#L515) | Service worker resource port |

`ROUTER.shutdown()` is called explicitly at teardown in both the script thread
([`script/script_thread.rs:3269`](../components/script/script_thread.rs#L3269),
multiprocess mode only) and the constellation
([`constellation/constellation.rs:2812`](../components/constellation/constellation.rs#L2812)).
