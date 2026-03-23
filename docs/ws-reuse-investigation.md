# WebSocket reuse investigation for Xray-core

Fork target: `edtxtongit/Xray-core`
Branch: `ws-reuse-investigation`
Date: 2026-03-23

## What I checked

I read the current WebSocket transport implementation and the existing multiplexing path.

Relevant files:

- `transport/internet/websocket/dialer.go`
- `transport/internet/websocket/hub.go`
- `transport/internet/websocket/connection.go`
- `infra/conf/transport_internet.go`
- `common/mux/*`
- `app/proxyman/outbound/handler.go`
- `app/proxyman/inbound/always.go`
- `transport/internet/splithttp/mux.go`

## Current state

### 1. WebSocket transport itself is one physical connection per dial

`transport/internet/websocket/dialer.go` currently creates a fresh `gorilla/websocket.Conn` for each `Dial(...)` call and wraps it as a `net.Conn`.

That means the WebSocket transport layer itself does **not** currently have a connection pool / xmux-style transport reuse.

### 2. Xray already has reuse over WebSocket through the generic mux layer

`app/proxyman/outbound/handler.go` builds a `mux.ClientManager` when outbound `mux.enabled` is on.
This mux layer is transport-agnostic and can run over a WebSocket transport.

So from the user-facing perspective, **multiple logical streams can already reuse one underlying WebSocket-based outbound channel** when the outbound enables `mux`.

### 3. Inbound side already has the corresponding mux server path

`app/proxyman/inbound/always.go` creates `mux.NewServer(ctx)`.
So the generic Xray mux path already exists on the receiving side too.

## Why transport-native WS reuse is not a tiny patch

XHTTP has its own transport-native xmux path (`transport/internet/splithttp/mux.go`).
WebSocket does not.

Adding an XHTTP-like native reuse layer for WebSocket would require at least:

1. A new config surface for WS reuse policy.
2. A global/shared WS client manager keyed by destination + stream settings.
3. A logical stream wrapper that can expose `net.Conn` semantics on top of one shared WS session.
4. A framing protocol for multiple logical streams over one WS channel.
5. A matching server-side demux path.
6. Careful interaction rules with the existing generic mux, XUDP, close semantics, deadlines, and stats.

In practice, that means this is **architecture work**, not just a small local edit in `websocket/dialer.go`.

## Practical conclusion

For now, the most realistic conclusion is:

- **WS transport-native pooling/xmux is not implemented**.
- **Functional WS reuse already exists via outbound `mux`**.
- If the goal is “let multiple requests share one WS tunnel”, the existing mux path may already satisfy that goal.
- If the goal is “add XHTTP-style native transport reuse specifically inside WS transport”, that is a larger feature and should be designed explicitly before coding.

## Recommended next steps

### Option A — document and expose the existing solution

Document clearly that for WebSocket-based outbounds, reuse should be achieved by enabling outbound `mux`.

This is the safest and lowest-risk path.

### Option B — design a real `ws xmux` feature

If the project explicitly wants transport-native WS reuse, create a design first:

- config shape
- compatibility with current mux
- stream frame format
- lifecycle / idle / reuse limits
- server-side demux behavior

Only after that should implementation start.

## My recommendation

Before writing invasive code, first confirm whether the desired feature is:

1. **existing mux over ws** (already supported), or
2. **new transport-native ws xmux** (new feature).

These are not the same thing.
