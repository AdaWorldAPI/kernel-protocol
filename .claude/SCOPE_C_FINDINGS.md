# Scope C: Jupyter Wire Protocol Findings

Source: `docs/messaging.rst` (protocol spec v5.4) and `docs/kernels.rst` (connection files, kernelspecs).

---

## 1. ZMQ Socket Types by Channel

| # | Channel       | Kernel-side socket | Client-side socket | Purpose |
|---|---------------|--------------------|--------------------|---------|
| 1 | **Shell**     | ROUTER             | DEALER             | Request/reply for execution, completion, inspection, kernel_info |
| 2 | **IOPub**     | PUB                | SUB                | Broadcast channel: stdout/stderr, display_data, execute_result, status, errors |
| 3 | **stdin**     | ROUTER             | DEALER             | Reverse request/reply: kernel asks frontend for user input (`input_request`/`input_reply`) |
| 4 | **Control**   | ROUTER             | DEALER             | Same as Shell but separate socket; used for shutdown, interrupt, debug (not queued behind execution) |
| 5 | **Heartbeat** | REP                | REQ                | Simple bytestring ping/pong to detect dead kernels |

**Key implementation notes:**
- The client's stdin DEALER socket MUST have the same ZMQ identity as the client's shell DEALER socket (required for `input_request` routing).
- IOPub subscribers typically subscribe to all topics (empty prefix). The topic prefix convention is `kernel.{uuid}.{msg_type}` or `stream.stdout`.
- Control channel should run in a **separate thread** from Shell so shutdown/interrupt aren't blocked by long-running execution.

---

## 2. Execute Message Flow — Full Structures

### Wire Protocol (ZMQ frames)

Every message is serialized as this sequence of ZMQ frames:

```
[
  b'zmq-identity',       # ZMQ routing prefix (zero or more frames)
  b'<IDS|MSG>',          # delimiter — literal bytes
  b'hmac-signature-hex', # HMAC-SHA256 hex digest (or empty string if auth disabled)
  b'{header}',           # JSON-serialized header dict
  b'{parent_header}',    # JSON-serialized parent header dict
  b'{metadata}',         # JSON-serialized metadata dict
  b'{content}',          # JSON-serialized content dict
  ...                    # zero or more extra binary buffers
]
```

For IOPub messages, the first frame (before `<IDS|MSG>`) is the **topic** string, e.g. `execute_result` or `stream.stdout`.

### Message Header (common to all messages)

```json
{
    "msg_id": "uuid-string",
    "session": "uuid-string",
    "username": "string",
    "date": "ISO-8601-timestamp",
    "msg_type": "execute_request",
    "version": "5.4"
}
```

### 2a. `execute_request` (Shell channel, client -> kernel)

```json
{
    "header": { "msg_type": "execute_request", "..." : "..." },
    "parent_header": {},
    "metadata": {},
    "content": {
        "code": "print('hello')\n1 + 1",
        "silent": false,
        "store_history": true,
        "user_expressions": {},
        "allow_stdin": false,
        "stop_on_error": true
    },
    "buffers": []
}
```

**Field types:**
- `code`: `str` — one or more lines of source code
- `silent`: `bool` — if true, no broadcast on IOPub, no execute_result, forces store_history=false
- `store_history`: `bool` — whether to increment execution counter (default: true when silent=false)
- `user_expressions`: `dict` — names->expressions evaluated after execution
- `allow_stdin`: `bool` — whether kernel may send `input_request`
- `stop_on_error`: `bool` — abort execution queue on exception

### 2b. `status` (IOPub channel, kernel -> all clients)

Sent **immediately** when kernel begins processing, and again when done.

```json
{
    "header": { "msg_type": "status", "..." : "..." },
    "parent_header": { "/* copy of execute_request header */": "" },
    "metadata": {},
    "content": {
        "execution_state": "busy"
    }
}
```

`execution_state`: one of `"busy"`, `"idle"`, `"starting"` (starting sent exactly once at process startup).

### 2c. `execute_input` (IOPub channel, kernel -> all clients)

Re-broadcast of the code being executed (so all connected frontends see it):

```json
{
    "header": { "msg_type": "execute_input", "..." : "..." },
    "parent_header": { "/* copy of execute_request header */": "" },
    "metadata": {},
    "content": {
        "code": "print('hello')\n1 + 1",
        "execution_count": 1
    }
}
```

### 2d. `stream` (IOPub channel, kernel -> all clients)

Stdout/stderr output:

```json
{
    "header": { "msg_type": "stream", "..." : "..." },
    "parent_header": { "/* copy of execute_request header */": "" },
    "metadata": {},
    "content": {
        "name": "stdout",
        "text": "hello\n"
    }
}
```

- `name`: `str` — `"stdout"` or `"stderr"`
- `text`: `str` — arbitrary text

### 2e. `display_data` (IOPub channel, kernel -> all clients)

Rich output produced by display calls (not the cell result):

```json
{
    "header": { "msg_type": "display_data", "..." : "..." },
    "parent_header": { "/* copy of execute_request header */": "" },
    "metadata": {},
    "content": {
        "data": {
            "text/plain": "a string representation",
            "text/html": "<table>...</table>"
        },
        "metadata": {
            "image/png": { "width": 640, "height": 480 }
        },
        "transient": {
            "display_id": "optional-id"
        }
    }
}
```

- `data`: `dict` — MIME-type keys -> raw representation values
- `metadata`: `dict` — per-MIME-type metadata
- `transient`: `dict` — optional, runtime-only (not persisted); `display_id` for updateable displays

### 2f. `execute_result` (IOPub channel, kernel -> all clients)

The cell's return value (the thing that would appear as `Out[N]`). Identical to `display_data` plus `execution_count`:

```json
{
    "header": { "msg_type": "execute_result", "..." : "..." },
    "parent_header": { "/* copy of execute_request header */": "" },
    "metadata": {},
    "content": {
        "execution_count": 1,
        "data": {
            "text/plain": "2",
            "text/html": "<strong>2</strong>"
        },
        "metadata": {}
    }
}
```

### 2g. `execute_reply` (Shell channel, kernel -> client)

Final reply to the original request:

```json
{
    "header": { "msg_type": "execute_reply", "..." : "..." },
    "parent_header": { "/* copy of execute_request header */": "" },
    "metadata": {},
    "content": {
        "status": "ok",
        "execution_count": 1,
        "payload": [],
        "user_expressions": {}
    }
}
```

On error:

```json
{
    "content": {
        "status": "error",
        "execution_count": 1,
        "ename": "NameError",
        "evalue": "name 'x' is not defined",
        "traceback": ["line 1, ...", "NameError: ..."]
    }
}
```

**Note:** `execution_count` is always present in `execute_reply` regardless of status.

### 2h. `error` (IOPub channel, kernel -> all clients)

Broadcast when execution raises an error:

```json
{
    "header": { "msg_type": "error", "..." : "..." },
    "parent_header": { "/* copy of execute_request header */": "" },
    "content": {
        "ename": "NameError",
        "evalue": "name 'x' is not defined",
        "traceback": ["..."]
    }
}
```

Same as the error fields in `execute_reply` but without `status` and `execution_count`.

---

## 3. HMAC Signing

**Algorithm:** HMAC-SHA256 (field `signature_scheme` in connection file = `"hmac-sha256"`).

**Key source:** The `key` field in the connection file (a UUID string, e.g. `"a0436f6c-1916-498b-8eb9-e81ab9368e84"`). If `key` is empty string, authentication is disabled (signature frame is empty).

**What gets signed:** The HMAC is computed over the **concatenation** of these four serialized (JSON bytes) frames, in order:
1. header
2. parent_header
3. metadata
4. content

**Pseudocode (Rust-relevant):**

```rust
let mut hmac = HmacSha256::new_from_slice(key.as_bytes());
hmac.update(header_bytes);
hmac.update(parent_header_bytes);
hmac.update(metadata_bytes);
hmac.update(content_bytes);
let signature: String = hex::encode(hmac.finalize().into_bytes());
```

The hex digest is placed in the signature frame between `<IDS|MSG>` and the header frame.

**Note:** Buffers are NOT included in the HMAC computation.

---

## 4. Connection File Format

Located at a path passed to the kernel as a command-line argument. File is readable only by the current user.

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

**All fields:**

| Field | Type | Description |
|-------|------|-------------|
| `transport` | `str` | Transport protocol, typically `"tcp"` (could be `"ipc"`) |
| `ip` | `str` | IP address to bind/connect, typically `"127.0.0.1"` |
| `shell_port` | `int` | Port for the Shell ROUTER socket |
| `iopub_port` | `int` | Port for the IOPub PUB socket |
| `stdin_port` | `int` | Port for the stdin ROUTER socket |
| `control_port` | `int` | Port for the Control ROUTER socket |
| `hb_port` | `int` | Port for the Heartbeat REP socket |
| `signature_scheme` | `str` | HMAC algorithm, always `"hmac-sha256"` |
| `key` | `str` | Shared secret for HMAC signing (UUID string); empty string disables auth |

**Address construction:** `{transport}://{ip}:{port}` — e.g. `tcp://127.0.0.1:57503`

---

## 5. Minimal R Execution — Full Message Trace

### Scenario
Connect to a running IRkernel, execute `df <- data.frame(x=1:3, y=c("a","b","c")); df`, receive both the text representation and any rich output.

### Prerequisites
1. Parse the connection file to get ports, key, transport, IP.
2. Create ZMQ sockets and connect:
   - Shell DEALER -> `tcp://127.0.0.1:{shell_port}`
   - IOPub SUB -> `tcp://127.0.0.1:{iopub_port}` (subscribe to all: empty prefix `""`)
   - Control DEALER -> `tcp://127.0.0.1:{control_port}` (for shutdown later)
   - (Optionally stdin DEALER and heartbeat REQ)
3. Generate a session UUID for all outgoing message headers.

### Message Sequence

```
CLIENT (Shell)                          KERNEL                          CLIENT (IOPub subscriber)
     |                                    |                                    |
     |  1. kernel_info_request ---------> |                                    |
     |                                    | ------ 2. status{busy} ----------> |
     |  <-------- 3. kernel_info_reply -- |                                    |
     |                                    | ------ 4. status{idle} ----------> |
     |                                    |                                    |
     |  5. execute_request --------------> |                                    |
     |     code="df <- data.frame(...);\n df"                                  |
     |     silent=false                   |                                    |
     |     store_history=true             |                                    |
     |     allow_stdin=false              |                                    |
     |                                    | ------ 6. status{busy} ----------> |
     |                                    | ------ 7. execute_input ----------> |
     |                                    |    code="df <- ...", exec_count=1   |
     |                                    |                                    |
     |                                    | ------ 8. execute_result ---------> |
     |                                    |    data={                           |
     |                                    |      "text/plain": "  x y\n1 1 a\n2 2 b\n3 3 c",
     |                                    |      "text/html": "<table>...</table>"
     |                                    |    }                                |
     |                                    |    execution_count=1                |
     |                                    |                                    |
     |  <-------- 9. execute_reply ------ |                                    |
     |     status="ok"                    |                                    |
     |     execution_count=1              |                                    |
     |                                    | ------ 10. status{idle} ---------> |
     |                                    |                                    |
```

### Step-by-step detail

**Step 1 — `kernel_info_request`** (optional but recommended; confirms kernel is alive):

```
Wire frames on Shell DEALER:
  [b'<IDS|MSG>', b'{hmac}',
   b'{"msg_id":"uuid1","session":"sess1","username":"","date":"...","msg_type":"kernel_info_request","version":"5.4"}',
   b'{}',   // parent_header
   b'{}',   // metadata
   b'{}'    // content
  ]
```

**Step 3 — `kernel_info_reply`** (confirms IRkernel, gives language_info):

```json
{
    "content": {
        "status": "ok",
        "protocol_version": "5.3",
        "implementation": "IRkernel",
        "implementation_version": "1.3.2",
        "language_info": {
            "name": "R",
            "version": "4.3.1",
            "mimetype": "text/x-r-source",
            "file_extension": ".r"
        },
        "banner": "R 4.3.1 -- IRkernel"
    }
}
```

**Step 5 — `execute_request`**:

```
Wire frames on Shell DEALER:
  [b'<IDS|MSG>', b'{hmac}',
   b'{"msg_id":"uuid2","session":"sess1","username":"","date":"...","msg_type":"execute_request","version":"5.4"}',
   b'{}',
   b'{}',
   b'{"code":"df <- data.frame(x=1:3, y=c(\"a\",\"b\",\"c\")); df","silent":false,"store_history":true,"user_expressions":{},"allow_stdin":false,"stop_on_error":true}'
  ]
```

**Steps 6-8 — IOPub messages** (received on SUB socket):

Each IOPub message has a topic prefix frame before `<IDS|MSG>`. The `parent_header` in each is a copy of the `execute_request`'s header, which is how the client correlates output to the request.

**Step 8 — `execute_result`** for a DataFrame:

IRkernel produces both `text/plain` and `text/html` representations:

```json
{
    "content": {
        "execution_count": 1,
        "data": {
            "text/plain": "  x y\n1 1 a\n2 2 b\n3 3 c",
            "text/html": "<table class=\"dataframe\">\n<thead><tr><th></th><th>x</th><th>y</th></tr></thead>\n<tbody>\n<tr><th>1</th><td>1</td><td>a</td></tr>\n<tr><th>2</th><td>2</td><td>b</td></tr>\n<tr><th>3</th><td>3</td><td>c</td></tr>\n</tbody>\n</table>"
        },
        "metadata": {}
    }
}
```

**Step 9 — `execute_reply`** on Shell:

```json
{
    "content": {
        "status": "ok",
        "execution_count": 1,
        "payload": [],
        "user_expressions": {}
    }
}
```

### Minimal message set summary

To execute R code and receive results, a client must handle these message types:

| Message type | Direction | Channel | Required? |
|---|---|---|---|
| `kernel_info_request` | client -> kernel | Shell | Recommended (validates connection) |
| `kernel_info_reply` | kernel -> client | Shell | Recommended |
| `execute_request` | client -> kernel | Shell | **Required** |
| `execute_reply` | kernel -> client | Shell | **Required** |
| `status` | kernel -> client | IOPub | **Required** (busy/idle bracketing) |
| `execute_input` | kernel -> client | IOPub | Expected (can ignore) |
| `stream` | kernel -> client | IOPub | Expected for print()/cat() output |
| `execute_result` | kernel -> client | IOPub | **Required** for cell return values |
| `display_data` | kernel -> client | IOPub | Expected for rich output (plots, HTML widgets) |
| `error` | kernel -> client | IOPub | Expected for error handling |

### Ordering guarantee

The `status: idle` message on IOPub signals that **all** IOPub messages for a given request have been sent. The client should collect IOPub messages associated with a request (matching `parent_header.msg_id`) until it sees `status: idle` with the same parent.

---

## Implementation Notes for Rust

1. **Crate choices:** `zeromq` (pure Rust) or `zmq` (libzmq bindings). The `hmac` and `sha2` crates for HMAC-SHA256. `serde_json` for serialization.

2. **ZMQ identity:** Set identity on DEALER sockets before connecting. The Shell and stdin DEALER sockets must share the same identity.

3. **Frame parsing:** Split received multipart message on the `<IDS|MSG>` delimiter (literal bytes `b"<IDS|MSG>"`). Everything before is routing/identity. Everything after: signature, header, parent_header, metadata, content, [buffers...].

4. **IOPub subscription:** After connecting the SUB socket, call `subscribe("")` to receive all messages. Topic filtering is optional.

5. **Message correlation:** Match IOPub messages to requests via `parent_header.msg_id == request.header.msg_id`.

6. **Shutdown:** Send `shutdown_request` on the Control channel (not Shell) with `{"restart": false}`.
