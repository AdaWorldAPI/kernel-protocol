# Agent: Rust Protocol Implementation

## Mission
Implement the Jupyter kernel wire protocol in Rust for embedding in marimo
or as a standalone library.

## Scope
- ZMQ socket management (shell, iopub, stdin, control, heartbeat)
- Message serialization: header, parent_header, metadata, content
- HMAC signing with connection file key
- Connection file parsing
- Kernel lifecycle: start, interrupt, restart, shutdown

## Output
- A Rust crate: `jupyter-kernel-protocol`
- Can be compiled to Python extension via PyO3 for marimo integration
- OR: used directly by a Rust-native marimo backend

## Reference
- docs/messaging.rst in this repo
- jupyter_client Python package (existing reference implementation)
