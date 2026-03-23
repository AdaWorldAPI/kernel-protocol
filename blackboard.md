# Blackboard — kernel-protocol

> Single-binary architecture: Jupyter wire protocol implemented in Rust (for R only).

## What Exists

This is the **specification** for the Jupyter Kernel Messaging Protocol v5.4. No code — just RST docs.

## Protocol Summary

### 5 ZMQ Channels

| Channel | Socket Type | Direction | Purpose |
|---------|------------|-----------|---------|
| Shell | ROUTER/DEALER | Client→Kernel | Execute, inspect, complete |
| IOPub | PUB/SUB | Kernel→Client (broadcast) | Status, output, errors |
| Stdin | ROUTER/DEALER | Kernel→Client (reversed) | Input requests |
| Control | ROUTER/DEALER | Client→Kernel | Shutdown, interrupt, debug |
| Heartbeat | REQ/REP | Ping/pong | Keep-alive |

### Message Wire Format

```
[zmq-routing-prefix]
<IDS|MSG>           (delimiter)
[HMAC-SHA256 hex]   (signature)
[header JSON]
[parent_header JSON]
[metadata JSON]
[content JSON]
[binary-buffers]    (optional)
```

### Minimal Message Set for R Execution

1. `kernel_info_request` → `kernel_info_reply` (handshake)
2. `execute_request` → status:busy → execute_input → stream/display_data/execute_result → execute_reply → status:idle

### Connection File

```json
{
  "transport": "tcp",
  "ip": "127.0.0.1",
  "shell_port": 57503,
  "iopub_port": 40885,
  "stdin_port": 52597,
  "control_port": 50160,
  "hb_port": 42540,
  "signature_scheme": "hmac-sha256",
  "key": "a0436f6c-1916-498b-8eb9-e81ab9368e84"
}
```

## What Gets Implemented in Rust

Minimal kernel client for IRkernel:
- ZMQ via `zeromq` crate
- HMAC-SHA256 signing
- `execute_request` / `execute_reply` / `display_data` / `stream` / `status`
- Connection file parsing
- Arrow IPC for DataFrame exchange

**NOT needed**: completion, inspection, history, comms, debugging.

## Key Files

| File | Purpose |
|---|---|
| `docs/messaging.rst` | Complete wire protocol spec |
| `docs/kernels.rst` | Kernel lifecycle, kernelspec, connection files |
