# kernel-protocol — Transcode to Rust

Read the spec in this repo. Read `.claude/SCOPE_C_FINDINGS.md`.
Implement the Jupyter kernel wire protocol as a standalone Rust crate.

ZMQ sockets. Message serialization. HMAC signing. Connection files.
Enough to drive IRkernel (R) and evcxr (Rust).

The crate must work on its own. No dependency on marimo,
graph-notebook, quarto, lance-graph, ndarray, or rs-graph-llm.

Output: a Rust crate in this repo that speaks Jupyter kernel protocol.
Someone adds it to their Cargo.toml, they can connect to any Jupyter kernel.

Read first. Implement. Test.
