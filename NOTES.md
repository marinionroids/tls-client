# Fork notes: WebSocket over CFFI

Branch `cffi-websocket` = upstream `master` (v1.16.0 + 3) + revived PR #251.
Adds `wsConnect`, `wsRead`, `wsWrite`, `wsClose` to the shared library. The WS dial goes through
`tls_client.NewWebsocket` and the client's uTLS dialer, so it uses the same browser profile as `request`.

## Build (linux-amd64, glibc >= 2.31)

```bash
docker run --rm -v $PWD:/src -w /src/cffi_dist golang:1.24-bullseye \
  go build -buildvcs=false -buildmode=c-shared -o dist/tls-client-ws-1.16.0-ws.4-linux-amd64.so .
nm -D cffi_dist/dist/*.so | grep -E ' T ws'   # wsClose wsConnect wsRead wsStats wsWrite
```

`cffi_dist/go.mod` now has `replace github.com/bogdanfinn/tls-client => ../` because
`cffi_src/websocket.go` is not in any upstream release. Tests: `go test -race ./cffi_src/`.

## API

Same lifecycle as `request`: JSON string in, JSON string out, then `freeMemory(out.id)`.
Errors use the normal error shape (`status: 0`, message in `body`).

```jsonc
// wsConnect
{ "tlsClientIdentifier": "safari_ios_26_0", "url": "wss://example.com/ws",
  "headers": { "User-Agent": "...", "Origin": "https://www.chess.com" },
  "headerOrder": ["host", "upgrade", "connection", "user-agent", "origin"],
  "handshakeTimeoutMilliseconds": 10000, "proxyUrl": "http://user:pass@host:port" }
// -> { "id": "...", "connectionId": "ws-uuid", "status": 101 }

// wsRead    (messageType 1 = text, 2 = binary with base64 data)
{ "connectionId": "ws-uuid", "timeoutMilliseconds": 30000 }
// -> { "id": "...", "connectionId": "ws-uuid", "messageType": 1, "data": "..." }

// wsWrite
{ "connectionId": "ws-uuid", "messageType": 2, "data": "<base64>" }
// -> { "id": "...", "connectionId": "ws-uuid", "success": true }

// wsClose
{ "connectionId": "ws-uuid" }
// -> { "id": "...", "success": true }

// wsStats   (diagnostics, never touches the wire; tag v1.16.0-ws.4)
{ "connectionId": "ws-uuid" }
// -> { "id": "...", "ageMs": 1506, "messagesRead": 2, "messagesWritten": 1, "bytesRead": 34, "bytesWritten": 2,
//      "msSinceLastRead": 1464, "msSinceLastWrite": 1503, "lastWriteDurationMs": 0, "maxWriteDurationMs": 0,
//      "unreadPending": false, "localAddr": "...", "remoteAddr": "...",
//      "tcp": { "state": 1, "retransmits": 0, "backoff": 0, "rtoMs": 233, "rttMs": 32, "unacked": 0, "lost": 0,
//               "totalRetrans": 0, "notsentBytes": 0, "msSinceLastDataSent": 1503, "msSinceLastDataRecv": 1465,
//               "msSinceLastAckRecv": 1465 } }
```

Other `wsConnect` fields: `sessionId`, `customTlsClient`, `readBufferSize`, `writeBufferSize`,
`insecureSkipVerify`, `withRandomTLSExtensionOrder`.

## Things to know

- **permessage-deflate is on.** `websocket.Dialer.EnableCompression` is true so the upgrade
  sends `Sec-WebSocket-Extensions: permessage-deflate; server_no_context_takeover; client_no_context_takeover`
  (iOS URLSession / ktor-client offer deflate; capture 2026-10-02). Gorilla decompresses if the
  server accepts. Tag `v1.16.0-ws.3`.
- **Session reuse needs a separate HTTP/1.1 session.** `wsConnect` with `sessionId` uses that client
  as-is and never modifies it. Create it with `forceHttp1: true`; do not pass your normal HTTP/2 session,
  the handshake fails once the server negotiates h2. Without `sessionId` an inline HTTP/1.1 client is
  built for the connection and nothing is added to the session store.
- **Read timeout is not fatal.** On timeout `wsRead` returns an error whose `body` is exactly
  `websocket read timeout`; the connection stays open, call `wsRead` again. Any other read error
  (peer close, network) closes the connection and drops its `connectionId`.
- **Threading.** One reader goroutine per connection answers pings even when nobody is in `wsRead`.
  `wsWrite` is safe alongside `wsRead` and other `wsWrite` calls. Use one `wsRead` caller per
  connection (several would each get different messages). `wsClose` releases a blocked `wsRead`.
- **Node/koffi: raise `UV_THREADPOOL_SIZE`.** koffi `.async` calls run on the libuv thread pool
  (default 4 threads) and every open socket parks one thread in `wsRead`. With 4+ quiet sockets,
  `wsWrite` (and DNS, fs, crypto) queues behind the reads for up to the read timeout, so app-level
  keepalives go out late and the server drops the connection. Set `UV_THREADPOOL_SIZE` in the
  process environment to at least open sockets + 8 (max 1024).
- **`wsStats` for silent stalls.** `tcp` is linux `TCP_INFO` for the socket to the server (or to the
  proxy); it is omitted on other platforms and when the tunnel is not a plain TCP socket (h2 proxy).
  `unacked` / `retransmits` / `backoff` > 0 with a large `msSinceLastAckRecv` = our bytes are not being
  acknowledged (dead path, writes still "succeed" into the send buffer). `unreadPending: true` = a
  message is waiting and the caller is not in `wsRead`. Counters are data messages, not ping/pong.
  A connection that already failed or was closed is gone from the store, so call it before `wsClose`.
- A failed `wsWrite` closes the connection and drops its `connectionId`, same as a failed read.
- Unread messages are not buffered in the library: if the caller stops calling `wsRead`, the reader
  blocks and TCP backpressure applies.
- `destroySession` / `destroyAll` do not close WebSocket connections. Call `wsClose`.
- `status` in the `wsConnect` output is always 101; a failed handshake is an error response.

## Diff vs upstream PR #251

- Reader goroutine + channel instead of `SetReadDeadline` + `ReadMessage` per call. In #251 a read
  timeout permanently broke the connection (gorilla read errors are sticky), a deadline from one call
  leaked into the next call without a timeout, and pings were only answered during `wsRead`.
- Write mutex: concurrent `wsWrite` calls panicked in gorilla ("concurrent write").
- `wsClose` uses `LoadAndDelete` + `sync.Once` (no double close), sends a close frame, and unblocks
  a pending `wsRead`. Connections that fail on read are removed from the map instead of leaking.
- `wsWrite` rejects message types other than 1 and 2.
- `cffi_dist/go.mod` replace directive so the dist module actually compiles against `cffi_src`.
- Tests in `cffi_src/websocket_test.go` (local TLS echo server).

## Node (koffi) sketch

```js
const koffi = require('koffi');
const lib = koffi.load('./tls-client-ws-1.16.0-linux-amd64.so');
const free = lib.func('void freeMemory(const char *)');
const fn = Object.fromEntries(['wsConnect', 'wsRead', 'wsWrite', 'wsClose']
  .map((n) => [n, lib.func(`const char *${n}(const char *)`)]));
// .async keeps the blocking read off the event loop
const call = (name, params) => new Promise((resolve, reject) =>
  fn[name].async(JSON.stringify(params), (err, res) => {
    if (err) return reject(err);
    const out = JSON.parse(res);
    free(out.id);
    out.status === 0 ? reject(new Error(out.body)) : resolve(out);
  }));

const { connectionId } = await call('wsConnect', { tlsClientIdentifier: 'safari_ios_26_0', url: 'wss://echo.websocket.org' });
await call('wsWrite', { connectionId, messageType: 1, data: 'hi' });
console.log(await call('wsRead', { connectionId, timeoutMilliseconds: 30000 }));
await call('wsClose', { connectionId });
```
