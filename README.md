# mcpl-core

A Rust library for MCPL (MCP Live). It has serde types for MCPL's capabilities and messages, and an async JSON-RPC connection over TCP or any other byte stream. Its `Cargo.toml` describes it as "MCPL v0.4 wire protocol types and async TCP transport".

MCPL is specified in [anima-research/mcpl](https://github.com/anima-research/mcpl), which describes it as "a backward-compatible extension to the Model Context Protocol (MCP) that enables servers to be active participants in the inference lifecycle." As of October 2026, the specification is at 0.5.0-draft.

## mcpl-core and mcpl-core-ts

This crate is the Rust counterpart of [mcpl-core-ts](https://github.com/anima-research/mcpl-core-ts), the TypeScript MCPL library. The npm package `@animalabs/mcpl-core` is built from mcpl-core-ts, not from this repository.

The same author started both libraries in February 2026. This crate was written for version 0.4 of the specification, and its code was last changed on 13 February 2026, in `795a014`. mcpl-core-ts has kept changing since then. It has added a WebSocket transport, the `state/update`, `state/get` and `branches/*` methods, and event tags, and in August 2026 it changed its capability types for version 0.5. This crate has none of those changes, and it can't read capabilities in the 0.5 shapes. In the 0.5 draft, `channels` can be an object of its parts, where this crate expects a boolean, and `featureSets` is keyed by name, where this crate expects an array. Deserializing the capabilities object in the draft's digest test vector (SPEC.md §17.2) into `McplCapabilities` fails.

## What's in it

| Module | Contents |
|---|---|
| `types` | JSON-RPC 2.0 messages (`JsonRpcRequest`, `JsonRpcResponse`, `JsonRpcNotification`), MCPL's error codes (`ERR_*`), and `ContentBlock` (text, image, audio or resource) |
| `capabilities` | `McplCapabilities`, which MCPL places under `experimental.mcpl` in MCP's `initialize`, and the params and result of `initialize` |
| `methods` | The types of MCPL's messages for feature sets, scoped access, state rollback and checkpoints, host-managed state (`HostManagedState`, `JsonPatchOperation`), push events, context hooks, server-initiated inference, model info and channels, and the method names in `method` |
| `connection` | `McplConnection`, `IncomingMessage` and `ConnectionError` |

The crate root re-exports everything in `types`, `capabilities` and `methods`, and `McplConnection`. Import `IncomingMessage` and `ConnectionError` from `mcpl_core::connection`.

`McplConnection` sends and reads one JSON message per line:

- `McplConnection::new` takes a tokio `TcpStream`, and `from_parts` takes any boxed `AsyncRead` and `AsyncWrite` that are `Unpin` and `Send`, such as tokio's stdin and stdout.
- `send_request` sends a request and waits for its response, with no timeout. An error response comes back as `ConnectionError::Rpc`. Requests and notifications that arrive while it waits are kept for `next_message`.
- `next_message` returns the next request or notification from the other side. Answer a request with `send_response` or `send_error`, and send a notification with `send_notification`.

## Using it

The crate isn't published on crates.io. Depend on it from Git:

```toml
[dependencies]
mcpl-core = { git = "https://github.com/anima-research/mcpl-core" }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
```

A host that connects to an MCPL server over TCP and declares, in MCP's `initialize`, that it accepts push events:

```rust
use mcpl_core::{
    method, ExperimentalCapabilities, ImplementationInfo, InitializeCapabilities, McplCapabilities,
    McplConnection, McplInitializeParams, McplInitializeResult,
};
use tokio::net::TcpStream;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut conn = McplConnection::new(TcpStream::connect("127.0.0.1:7777").await?);

    let mut mcpl = McplCapabilities::new("0.4");
    mcpl.push_events = Some(true);
    let params = McplInitializeParams {
        protocol_version: "2024-11-05".into(), // MCP's revision, which MCPL leaves as it is
        capabilities: InitializeCapabilities {
            experimental: Some(ExperimentalCapabilities { mcpl: Some(mcpl) }),
            other: Default::default(),
        },
        client_info: ImplementationInfo { name: "my-host".into(), version: "0.1.0".into() },
    };
    let result = conn.send_request(method::INITIALIZE, Some(serde_json::to_value(&params)?)).await?;
    let result: McplInitializeResult = serde_json::from_value(result)?;
    conn.send_notification("notifications/initialized", None).await?;

    let server = result.capabilities.experimental.and_then(|e| e.mcpl);
    println!("server pushes events: {}", server.is_some_and(|m| m.has_push_events()));
    Ok(())
}
```

After `initialize`, MCP has the host send `notifications/initialized`, as the example does, and MCPL 0.4 asks it to send `featureSets/update`, which enables the server's feature sets. The server's requests, such as `push/event`, then arrive through `conn.next_message()`. `tests/connection_test.rs` has more examples: a capability negotiation, a push event, a channel registered and opened, an error response, and a notification kept while a request waits.

## Building and testing

```sh
cargo build
cargo test
```

## License

MIT, as declared in `Cargo.toml`.
