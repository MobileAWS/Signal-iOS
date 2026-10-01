# 09 · File Index

Every first-party `.swift` file under `SignalServiceKit/Network/` (58 files), with
a one-line summary and a link to the section that covers it. Confidence is
per-file as described in the [README](README.md).

## Chat web socket → [02](02-chat-websocket.md)

| File | Summary |
| --- | --- |
| [`OWSChatConnection.swift`](../../../SignalServiceKit/Network/OWSChatConnection.swift) | Base + libsignal-backed identified/unidentified chat connections: tokens, state, reconnect backoff, keepalive, incoming envelopes |
| [`ChatConnectionManager.swift`](../../../SignalServiceKit/Network/ChatConnectionManager.swift) | Protocol + impl routing `TSRequest`s to the right connection; `withAuth/UnauthService`; mock |
| [`SSKWebSocket.swift`](../../../SignalServiceKit/Network/SSKWebSocket.swift) | `URLSessionWebSocketTask` wrapper, `WebSocketRequest` (wss), factories, `WebSocketError` |
| [`WebSocketPromise.swift`](../../../SignalServiceKit/Network/WebSocketPromise.swift) | Promise/request-response adapter over `SSKWebSocket` |
| [`ConnectionLock.swift`](../../../SignalServiceKit/Network/ConnectionLock.swift) | Cross-process `fcntl` connection lock with priorities + Darwin-notification interrupts |

## REST stack → [03](03-rest-stack.md)

| File | Summary |
| --- | --- |
| [`OWSUrlSession.swift`](../../../SignalServiceKit/Network/OWSUrlSession.swift) | Concrete `URLSession` wrapper: requests/uploads/downloads/websocket, 2xx gate, pinning, proxy, HTTP/3, task state |
| [`OWSURLSessionProtocol.swift`](../../../SignalServiceKit/Network/OWSURLSessionProtocol.swift) | Transport protocol, `HTTPMethod`, `OWSUrlDownloadResponse`, `OWSUrlFrontingInfo`, convenience + multipart extensions |
| [`OWSURLSessionEndpoint.swift`](../../../SignalServiceKit/Network/OWSURLSessionEndpoint.swift) | Endpoint model (baseUrl/fronting/securityPolicy/extraHeaders), request/URL building |
| [`OWSURLBuilderUtil.swift`](../../../SignalServiceKit/Network/OWSURLBuilderUtil.swift) | `joinUrl` — merges a path with a base URL, overriding scheme and clearing credentials |
| [`OWSMultipart.swift`](../../../SignalServiceKit/Network/OWSMultipart.swift) | Streams multipart/form-data bodies (AFNetworking-derived) |
| [`HttpHeaders.swift`](../../../SignalServiceKit/Network/HttpHeaders.swift) | Case-insensitive headers, default/User-Agent/Accept-Language, Basic auth, log allowlist |
| [`HttpSecurityPolicy.swift`](../../../SignalServiceKit/Network/HttpSecurityPolicy.swift) | TLS trust evaluation: `systemDefault` vs `signalCaPinned` |
| [`Certificates.swift`](../../../SignalServiceKit/Network/Certificates.swift) | Loads pinned certs from the SignalServiceKit bundle |
| [`BaseOWSURLSessionMock.swift`](../../../SignalServiceKit/Network/BaseOWSURLSessionMock.swift) | Test mock returning HTTP 200 |

## API layer → [04](04-api-layer.md) and [05](05-endpoints.md)

| File | Summary | Doc |
| --- | --- | --- |
| [`API/NetworkManager.swift`](../../../SignalServiceKit/Network/API/NetworkManager.swift) | Request entry point, `RetryPolicy`, libsignal/system proxy wiring, mocks | [04](04-api-layer.md) |
| [`API/HTTPUtils.swift`](../../../SignalServiceKit/Network/API/HTTPUtils.swift) | Error preprocessing/classification, `Retry-After` parsing, outage reporting | [04](04-api-layer.md) |
| [`API/HTTPResponse.swift`](../../../SignalServiceKit/Network/API/HTTPResponse.swift) | Response struct + JSON/string/param parsing + `asError()` | [04](04-api-layer.md) |
| [`API/OWSHTTPError.swift`](../../../SignalServiceKit/Network/API/OWSHTTPError.swift) | Error enum (wrapped/network/serviceResponse), retryability, 429 handling | [04](04-api-layer.md) |
| [`API/DeviceProvisioningService.swift`](../../../SignalServiceKit/Network/API/DeviceProvisioningService.swift) | Provisioning code request + device provisioning | [04](04-api-layer.md) / [05](05-endpoints.md) |
| [`API/DeviceLimitExceededError.swift`](../../../SignalServiceKit/Network/API/DeviceLimitExceededError.swift) | Localized error for HTTP 411 | [04](04-api-layer.md) |
| [`API/AccountDataReport.swift`](../../../SignalServiceKit/Network/API/AccountDataReport.swift) | Parses the account data report (JSON + text) | [04](04-api-layer.md) |
| [`API/Giphy/GiphyAPI.swift`](../../../SignalServiceKit/Network/API/Giphy/GiphyAPI.swift) | Giphy trending/search via a dedicated `OWSURLSession` + `ContentProxy` | [04](04-api-layer.md) |
| [`API/Giphy/GiphyDownloader.swift`](../../../SignalServiceKit/Network/API/Giphy/GiphyDownloader.swift) | `ProxiedContentDownloader` subclass for GIFs | [04](04-api-layer.md) / [07](07-proxy.md) |
| [`API/Giphy/GiphyImageInfo.swift`](../../../SignalServiceKit/Network/API/Giphy/GiphyImageInfo.swift) | Giphy result model + `GiphyError` | [04](04-api-layer.md) |

### `API/Requests/` (request types, auth, factories) → [05](05-endpoints.md)

| File | Summary |
| --- | --- |
| [`TSRequest.swift`](../../../SignalServiceKit/Network/API/Requests/TSRequest.swift) | The request model: url/method/body/headers/timeout + `Auth` + `applyAuth` + log redaction |
| [`ChatServiceAuth.swift`](../../../SignalServiceKit/Network/API/Requests/ChatServiceAuth.swift) | Implicit vs explicit chat credentials |
| [`AuthedAccount.swift`](../../../SignalServiceKit/Network/API/Requests/AuthedAccount.swift) | Implicit vs explicit account identity → `ChatServiceAuth` |
| [`RemoteAttestationAuthFetcher.swift`](../../../SignalServiceKit/Network/API/Requests/RemoteAttestationAuthFetcher.swift) | Fetches CDSI/SVR2 attestation auth (`v2/directory/auth`, `v2/svr/auth`) |
| [`AccountDataReportRequestFactory.swift`](../../../SignalServiceKit/Network/API/Requests/AccountDataReportRequestFactory.swift) | `GET v2/accounts/data_report` |
| [`OWSRequestFactory.swift`](../../../SignalServiceKit/Network/API/Requests/OWSRequestFactory.swift) | Main factory: config, calling relays, auth, challenges, keys, profiles, donations, device provisioning, reglock |
| [`OWSRequestFactory+Usernames.swift`](../../../SignalServiceKit/Network/API/Requests/OWSRequestFactory+Usernames.swift) | Username hash reserve/confirm/lookup + username link |
| [`OWSRequestFactory+Spam.swift`](../../../SignalServiceKit/Network/API/Requests/OWSRequestFactory+Spam.swift) | Report spam |
| [`OWSRequestFactory+BoostPayments.swift`](../../../SignalServiceKit/Network/API/Requests/OWSRequestFactory+BoostPayments.swift) | One-time boost Stripe/PayPal payments |
| [`OWSRequestFactory+Backups.swift`](../../../SignalServiceKit/Network/API/Requests/OWSRequestFactory+Backups.swift) | `v1/archives/*` backup endpoints |
| [`WhoAmI/WhoAmIManager.swift`](../../../SignalServiceKit/Network/API/Requests/WhoAmI/WhoAmIManager.swift) | `GET v1/accounts/whoami` + `WhoAmI` response + mock |
| [`Provisioning/ProvisioningRequestFactory.swift`](../../../SignalServiceKit/Network/API/Requests/Provisioning/ProvisioningRequestFactory.swift) | `PUT v1/devices/link` (verify secondary device) |
| [`Provisioning/ProvisioningServiceResponses.swift`](../../../SignalServiceKit/Network/API/Requests/Provisioning/ProvisioningServiceResponses.swift) | Link-device response codes (200/409/411) + response model |
| [`Registration/RegistrationRequestFactory.swift`](../../../SignalServiceKit/Network/API/Requests/Registration/RegistrationRequestFactory.swift) | Session API, SVR2 auth check, create account, change number |
| [`Registration/RegistrationServiceResponses.swift`](../../../SignalServiceKit/Network/API/Requests/Registration/RegistrationServiceResponses.swift) | Response-code enums for the registration session API |
| [`AccountAttributes/AccountAttributes.swift`](../../../SignalServiceKit/Network/API/Requests/AccountAttributes/AccountAttributes.swift) | `Codable` account attributes + `Capabilities` |
| [`AccountAttributes/AccountAttributesGenerator.swift`](../../../SignalServiceKit/Network/API/Requests/AccountAttributes/AccountAttributesGenerator.swift) | Builds `AccountAttributes` for a primary device |
| [`AccountAttributes/AccountAttributesRequestFactory.swift`](../../../SignalServiceKit/Network/API/Requests/AccountAttributes/AccountAttributesRequestFactory.swift) | `PUT v1/accounts/attributes` / `PUT v1/devices/capabilities` |
| [`AccountAttributes/AccountAttributesUpdater.swift`](../../../SignalServiceKit/Network/API/Requests/AccountAttributes/AccountAttributesUpdater.swift) | Protocol to schedule an attributes update |
| [`AccountAttributes/AccountAttributesUpdaterImpl.swift`](../../../SignalServiceKit/Network/API/Requests/AccountAttributes/AccountAttributesUpdaterImpl.swift) | Cron-driven periodic/on-change/on-request updater |

## Signal service & censorship → [06](06-signal-service-censorship.md)

| File | Summary |
| --- | --- |
| [`OWSSignalService.swift`](../../../SignalServiceKit/Network/OWSSignalService.swift) | REST session factory + censorship-circumvention state + CDN session cache |
| [`OWSSignalServiceProtocol.swift`](../../../SignalServiceKit/Network/OWSSignalServiceProtocol.swift) | Protocol + `SignalServiceType`/`SignalServiceInfo` + per-service session builders |
| [`OWSSignalServiceMock.swift`](../../../SignalServiceKit/Network/OWSSignalServiceMock.swift) | Test mock with injectable builders |
| [`OWSCensorshipConfiguration.swift`](../../../SignalServiceKit/Network/OWSCensorshipConfiguration.swift) | Fronting hosts, pinning policies, SNI, censored calling-code map |
| [`OWSCountryMetadata.swift`](../../../SignalServiceKit/Network/OWSCountryMetadata.swift) | Country list mapping country codes → fronting domains |

## Proxy support → [07](07-proxy.md)

| File | Summary |
| --- | --- |
| [`SignalProxy/SignalProxy.swift`](../../../SignalServiceKit/Network/SignalProxy/SignalProxy.swift) | User-configured TLS proxy facade: state, validation, persistence, libsignal wiring |
| [`SignalProxy/SignalProxy+RelayServer.swift`](../../../SignalServiceKit/Network/SignalProxy/SignalProxy+RelayServer.swift) | Local loopback HTTP proxy listener + backoff |
| [`SignalProxy/SignalProxy+RelayClient.swift`](../../../SignalServiceKit/Network/SignalProxy/SignalProxy+RelayClient.swift) | Per-connection CONNECT parser bridging to the proxy client |
| [`SignalProxy/SignalProxy+ProxyClient.swift`](../../../SignalServiceKit/Network/SignalProxy/SignalProxy+ProxyClient.swift) | Connects to the Signal TLS Proxy; 200/503 handshake |
| [`ContentProxy.swift`](../../../SignalServiceKit/Network/ContentProxy.swift) | Fixed `contentproxy.signal.org` proxy config + request padding |
| [`ProxiedContentDownloader.swift`](../../../SignalServiceKit/Network/ProxiedContentDownloader.swift) | Generic segmented/cached proxied content downloader (base of `GiphyDownloader`) |

## Reachability & outage → [08](08-reachability-outage.md)

| File | Summary |
| --- | --- |
| [`ReachabilityManager.swift`](../../../SignalServiceKit/Network/ReachabilityManager.swift) | `SCNetworkReachability`-based reachability + wake-up download + mock |
| [`OutageDetection.swift`](../../../SignalServiceKit/Network/OutageDetection.swift) | DNS-probe-based Signal outage detection |
| [`NetworkInterfaceSet.swift`](../../../SignalServiceKit/Network/NetworkInterfaceSet.swift) | `OptionSet` of cellular/wifi interfaces |

## Coverage

All **58** `.swift` files under `SignalServiceKit/Network/` are listed above and
documented in one of docs [02](02-chat-websocket.md)–[08](08-reachability-outage.md).
