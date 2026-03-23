# kernel-protocol — Jupyter Kernel Wire Protocol

## Role in Stack
The spec. This repo defines the ZMQ message format that all Jupyter kernels
speak. Our fork is the reference for implementing the kernel protocol bridge
in marimo.

## Current State (upstream jupyter)
- Sphinx docs describing the wire protocol
- Message types: execute_request/reply, inspect, complete, kernel_info, etc.
- ZMQ socket topology: shell, iopub, stdin, control, heartbeat
- Connection files: JSON with ports, key, transport

## Our Fork's Mission
1. Reference spec for implementing marimo's kernel protocol client
2. Possibly: Rust implementation of the protocol (zmq bindings + message ser/de)
3. Test harness: verify our implementation matches the spec

## Key Files
- docs/messaging.rst — THE spec (message types, fields, semantics)
- docs/kernels.rst — kernel discovery, kernelspec, lifecycle

## Integration Points
- **marimo** → implements client side of this protocol
- **evcxr** → existing Rust kernel that speaks this protocol
- **IRkernel** → existing R kernel that speaks this protocol
- **graph-notebook magics** → route through this protocol
