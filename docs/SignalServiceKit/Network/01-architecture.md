# 01 · Architecture

High-level view of the `SignalServiceKit/Network/` layer and how a request
travels through it.

## Layering

```mermaid
flowchart TB
    subgraph callers["Callers (feature code)"]
        FC["Managers / services"]
    end

    subgraph api["API layer (API/)"]
        NM["NetworkManager\nasyncRequest(retryPolicy)"]
        RF["OWSRequestFactory\n+ sub-factories"]
        TSR["TSRequest\n(url, method, body, auth)"]
        HE["OWSHTTPError / HTTPUtils\nHTTPResponse"]
    end

    subgraph ws["Chat web socket transport"]
        CCM["ChatConnectionManager"]
        OCC["OWSChatConnection\n(identified / unidentified)"]
        LSN["libsignal Net\n(ChatConnection)"]
    end

    subgraph rest["REST transport"]
        SS["OWSSignalService\n(endpoint + session builder)"]
        OUS["OWSURLSession"]
        EP["OWSURLSessionEndpoint"]
        SEC["HttpSecurityPolicy / Certificates"]
    end

    subgraph infra["Cross-cutting infra"]
        CC["Censorship circumvention\n(OWSCensorshipConfiguration,\nOWSCountryMetadata)"]
        PX["SignalProxy / ContentProxy"]
        RCH["ReachabilityManager"]
        OUT["OutageDetection"]
    end

    FC --> RF --> TSR
    FC --> NM
    NM --> CCM --> OCC --> LSN
    NM -->|wraps results| HE
    FC -->|CDN / storage / registration| SS --> OUS --> EP
    OUS --> SEC
    SS --> CC
    OUS --> PX
    LSN --> PX
    RCH -. notifies .-> NM
    OCC -. success/failure .-> OUT
    OUS -. failure .-> OUT
```

(Confidence: **High** for the edges backed by the files cited below; **Medium**
for the exact caller fan-in, which lives outside this directory.)

## Request shape: `TSRequest`

Every service request is modeled by `TSRequest`
(`SignalServiceKit/Network/API/Requests/TSRequest.swift:9-44`), carrying:

- `url` (usually just a path + query, host resolved later), `method`, `headers`,
  `body` (`.parameters` / `.encodable` / `.data`),
- `timeoutInterval` (default `OWSRequestFactory.textSecureHTTPTimeOut` = **10s**;
  `TSRequest.swift:12`, `OWSRequestFactory.swift:19`),
- `maxResponseSize` (default `.max`; `TSRequest.swift:13`),
- an `auth` discriminator (`TSRequest.swift:97`).

(Confidence: **High**.)

## Two transports, one request type

- `NetworkManager.asyncRequest(_:retryPolicy:)` → `ChatConnectionManager.makeRequest`
  → chat web socket. See [02-chat-websocket.md](02-chat-websocket.md).
  (`SignalServiceKit/Network/API/NetworkManager.swift:170-179`.)
- `OWSSignalService` builds `OWSURLSession`s for REST calls (CDN, storage,
  registration, updates). See [03-rest-stack.md](03-rest-stack.md).
  (`SignalServiceKit/Network/OWSSignalServiceProtocol.swift:55-78`.)

Both transports apply the same auth rules via `TSRequest.applyAuth(to:socketAuth:)`
(`TSRequest.swift:127-146`) and surface the same `HTTPResponse` /
`OWSHTTPError` result types. See [04-api-layer.md](04-api-layer.md).

## End-to-end flow (web socket request)

```mermaid
sequenceDiagram
    autonumber
    participant C as Caller
    participant NM as NetworkManager
    participant CCM as ChatConnectionManager
    participant OCC as OWSChatConnection
    participant LS as libsignal ChatConnection
    participant OD as OutageDetection

    C->>NM: asyncRequest(TSRequest, retryPolicy)
    NM->>NM: Retry.performWithBackoff(maxAttempts)
    NM->>CCM: makeRequest(request)
    CCM->>CCM: route by auth.connectionType
    CCM->>OCC: makeRequest(request)
    OCC->>OCC: waitUntilReadyAndPerformRequest (30s)
    OCC->>LS: send(ChatConnection.Request)
    LS-->>OCC: ChatConnection.Response
    OCC->>OCC: handleRequestResponse (2xx vs preprocess error)
    OCC->>OD: reportConnectionSuccess()
    OCC-->>NM: HTTPResponse / throws OWSHTTPError
    NM-->>C: HTTPResponse / throws
```

Citations: `NetworkManager.swift:159-189`; `ChatConnectionManager.swift:184-187`;
`OWSChatConnection.swift` (`makeRequest` `406-424`, `waitUntilReadyAndPerformRequest`
`176-205`, `handleRequestResponse` `431-470`, `makeRequestInternal` for libsignal
`~690-760`). (Confidence: **High**.)
