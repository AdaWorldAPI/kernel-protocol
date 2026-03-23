# Progress

## Phase 1: Understand
- [ ] Read messaging.rst fully
- [ ] Read kernels.rst fully
- [ ] Map message types to use cases
- [ ] Study existing implementations (jupyter_client Python, evcxr)

## Phase 2: Rust Crate
- [ ] ZMQ bindings (zeromq-rs or zmq crate)
- [ ] Message struct definitions
- [ ] Serialization/deserialization
- [ ] HMAC signing
- [ ] Connection file parser
- [ ] Basic client: connect, execute, receive

## Phase 3: Integration
- [ ] PyO3 wrapper for marimo
- [ ] Test against evcxr kernel
- [ ] Test against IRkernel
