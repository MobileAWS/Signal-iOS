# SignalServiceKit `Network/` Layer

This directory documents the networking layer of Signal-iOS, located at
[`SignalServiceKit/Network/`](../../../SignalServiceKit/Network/).

All citations are `path:line` relative to the repository root. Each claim is
labeled with a confidence level:

- **High** — directly read from source; behavior is explicit in code.
- **Medium** — inferred from code structure, naming, or comments; cross-file
  behavior that is strongly implied but not fully traced.
- **Low** — depends on types/constants defined outside this directory
  (e.g. `TSConstants`, `libsignal`) that were not read.

## Documents

| Doc | Scope |
| --- | --- |
| [01-architecture.md](01-architecture.md) | Big-picture layering + how a request flows through the stack |
| [02-chat-websocket.md](02-chat-websocket.md) | Chat web socket (`OWSChatConnection`, `ChatConnectionManager`, `SSKWebSocket`, `WebSocketPromise`, `ConnectionLock`) |
| [03-rest-stack.md](03-rest-stack.md) | REST transport (`OWSUrlSession`, `OWSURLSessionProtocol`, `OWSURLSessionEndpoint`, `OWSMultipart`, `HttpHeaders`, `HttpSecurityPolicy`, `Certificates`, `OWSURLBuilderUtil`, mocks) |
| [04-api-layer.md](04-api-layer.md) | `API/` — `NetworkManager`, HTTP types/errors, request factories, provisioning, Giphy |
| [05-endpoints.md](05-endpoints.md) | Endpoint catalog: method, purpose, request/response shape, auth, retry per endpoint |
| [06-signal-service-censorship.md](06-signal-service-censorship.md) | `OWSSignalService`, censorship circumvention / domain fronting, country metadata |
| [07-proxy.md](07-proxy.md) | `SignalProxy/`, `ContentProxy`, `ProxiedContentDownloader` |
| [08-reachability-outage.md](08-reachability-outage.md) | `ReachabilityManager`, `OutageDetection`, `NetworkInterfaceSet` |
| [09-file-index.md](09-file-index.md) | Index of every first-party file in `Network/` with a one-line summary |

## Two transports

Signal-iOS speaks to its servers over **two transports**, both documented here:

1. **Chat web socket** (`libsignal`-backed): the long-lived connection used for
   the vast majority of authenticated/unauthenticated service requests and for
   receiving incoming message envelopes. See [02-chat-websocket.md](02-chat-websocket.md).
2. **REST (`URLSession`)**: one-shot HTTPS requests used for CDN
   uploads/downloads, storage service, registration, Giphy, and the background
   "wake-up" request. See [03-rest-stack.md](03-rest-stack.md).

`NetworkManager.asyncRequest(_:)` routes `TSRequest`s over the **web socket**
transport (via `ChatConnectionManager`), while `OWSSignalService` /
`OWSURLSession` are used for the REST transport.
(Confidence: **High** — `SignalServiceKit/Network/API/NetworkManager.swift:170-179`.)
