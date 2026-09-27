# Apodeixis Language & Proof Engine — Complete Specification & Developer Guide

## Overview

**Apodeixis** (from Ancient Greek *ἀπόδειξις*, meaning "demonstration, proof") is a domain-specific verifiable programming language, static typechecker, and embedded theorem-prover designed for zero-trust systems, microkernels, and autonomous security sidecars (such as **Mini Guardian**).

Apodeixis combines the low-level predictability of Assembly, the memory-safety of Rust, the formal proof verification of Lean, and the deterministic execution model of Holy C.

---

## 1. Release Architecture & Feature Status

| Feature / Subsystem | Implementation File | Status | Description |
| :--- | :--- | :--- | :--- |
| **Parser & AST** | `apodeixis/src/parser.rs` | **Shipped** | Parses `requires`, `ensures`, `fn`, `struct`, `enum`, `let`, `if`, `match`. |
| **Deterministic Interpreter** | `apodeixis/src/interpreter.rs` | **Shipped** | Zero-heap inner loop, 10,000 step budget limit, call-depth bounds. |
| **IFC Lattice Taint Engine** | `apodeixis/src/interpreter.rs` | **Shipped** | Enforces `@Secret` vs `@Public` taint lattice across expressions, calls, and outputs. |
| **Output Sink Enforcement** | `apodeixis/src/interpreter.rs` | **Shipped** | `print`/`println` builtins fail-closed on `@Secret` values. `declassify()` escapes. |
| **Piggyback Telemetry Codec** | `miniguard/src/piggyback_codec.rs` | **Shipped** | Verifies 8-byte fleet telemetry frames via embedded `PIGGYBACK_POLICY`. |
| **C / Python / Go FFI Bindings** | `apodeixis/src/ffi.rs` | **Shipped** | `apodeixis_run_mission` C ABI export for C, Python (`ctypes`), and Go (`cgo`). |
| **Standard Mission Library** | `apodeixis/src/mission_library.rs` | **Shipped** | Pre-compiled missions for JSON, auth, config, OTA updates, and fleet signals. |
| **Fleet Connection Guard** | `miniguard/src/connection_guard.rs` | **Shipped** | Evaluates node mTLS, HMAC, and auth delta signals; triggers auto-quarantine. |

---

## 2. Information Flow Control (IFC) Lattice

Apodeixis enforces a strict security lattice where:
$$\text{Public} \sqsubset \text{Secret}$$

### 2.1 Label Annotations
Variables can be annotated with security labels during binding:
```rust
let mut payload @ Secret = b"\x00\x01\x02\x03";
let safe_meta @ Public = "version_1.0";
```

### 2.2 Taint Propagation
Taint automatically flows to composite operations. Any expression containing a `@Secret` component evaluates to a `@Secret` result:
- **Binary/Unary Operations**: `secret_val + 5` $\rightarrow$ `@Secret`
- **Array & Struct Indexing**: `secret_bytes[0]` $\rightarrow$ `@Secret`
- **Casts & Builtins**: `to_string(secret_val)` $\rightarrow$ `@Secret`, `to_hex(secret_bytes)` $\rightarrow$ `@Secret`
- **Interprocedural Function Calls**: Passing a `@Secret` argument to a user function automatically taints the target parameter:
  ```rust
  fn process(data) {
      // data inherits @Secret from caller argument
      print(data) // FAILS CLOSED
  }
  ```

### 2.3 Output Sink Protection
To prevent accidental or malicious data exfiltration:
- Standard output builtins (`print`, `println`) check argument labels. If any argument carries a `@Secret` label, execution immediately halts with:
  ```text
  Security Violation: Information flow leak! Cannot print 'Secret' data to output
  ```
- **Declassification**: The explicit escape hatch `declassify(expr)` clears the security label when outputting sanitized or encrypted payloads:
  ```rust
  let safe_str = declassify(secret_val);
  print(safe_str); // Permitted
  ```

---

## 3. Apodeixis Builtin Functions Catalog

| Builtin | Signature | Description |
| :--- | :--- | :--- |
| `signal(name)` | `(Str) -> Num` | Reads host telemetry/signal input. |
| `emit_action(action, target)` | `(Str, Str) -> Unit` | Proposes a policy decision. |
| `to_string(val)` | `(Any) -> Str` | Converts value to string representation. |
| `to_hex(bytes)` | `(ByteArray) -> Str` | Converts byte array to 2-digit hex string. |
| `to_uint(val)` | `(Num) -> UInt` | Converts positive number to unsigned integer. |
| `declassify(val)` | `(Any) -> Any` | Clears `@Secret` security label from value. |
| `print(val)` / `println(val)` | `(Any) -> Unit` | Output sink (fail-closed on `@Secret`). |
| `len(arr)` | `(Array) -> Num` | Returns element count. |

---

## 4. Standard Mission Library (`mission_library.rs`)

Apodeixis ships with pre-compiled mission templates for quick developer integration:

### 4.1 `MISSION_VALID_JSON`
```rust
fn main() {
    let len = signal("json.len");
    if len > 0.0 && len <= 65536.0 {
        if signal("json.valid_syntax") == 1.0 {
            emit_action("accept_json", "valid")
        } else {
            emit_action("drop_json", "syntax_error")
        }
    } else {
        emit_action("drop_json", "invalid_length")
    }
}
```

### 4.2 `MISSION_AUTH`
```rust
fn main() {
    let authed = signal("auth.is_authenticated");
    let mfa = signal("auth.mfa_verified");
    if authed == 1.0 && mfa == 1.0 {
        emit_action("allow_access", "authenticated")
    } else {
        emit_action("deny_access", "unauthenticated")
    }
}
```

### 4.3 `MISSION_PIGGYBACK_SAFE`
```rust
fn main() {
    let fam = signal("pb.fam");
    let sub = signal("pb.sub");
    if fam >= 0.0 && fam <= 3.0 && sub >= 0.0 && sub <= 3.0 {
        emit_action("relay_signal", "safe")
    } else {
        emit_action("drop_signal", "invalid_frame")
    }
}
```

---

## 5. Cross-Language FFI Integration (C, Python, Go, Node.js)

Apodeixis exports a C ABI via `apodeixis/src/ffi.rs`.

### 5.1 C Integration Example
```c
#include <stdio.h>
#include "apodeixis.h"

int main() {
    CSignalPair signals[2] = {
        {"json.len", 128.0},
        {"json.valid_syntax", 1.0}
    };
    
    CApodeixisResult* res = apodeixis_run_mission(
        "fn main() { emit_action(\"accept\", \"ok\") }",
        signals, 2, 10000
    );

    if (res->success) {
        printf("Action: %s\n", res->proposed_action);
    }
    
    apodeixis_free_result(res);
    return 0;
}
```

### 5.2 Python Integration Example (`ctypes`)
```python
import ctypes

class CApodeixisResult(ctypes.Structure):
    _fields_ = [
        ("success", ctypes.c_int),
        ("has_action", ctypes.c_int),
        ("proposed_action", ctypes.c_char_p),
        ("proposed_target", ctypes.c_char_p),
        ("steps_used", ctypes.c_uint64),
        ("error_message", ctypes.c_char_p),
    ]

lib = ctypes.CDLL("target/release/libapodeixis.so")
lib.apodeixis_run_mission.restype = ctypes.POINTER(CApodeixisResult)

res_ptr = lib.apodeixis_run_mission(
    b"fn main() { emit_action('allow', 'ok') }",
    None, 0, 10000
)

res = res_ptr.contents
if res.success:
    print(f"Proposed Action: {res.proposed_action.decode('utf-8')}")

lib.apodeixis_free_result(res_ptr)
```

---

## 6. Verification & Test Suite

Run full language tests:
```bash
cargo test -p apodeixis
```

Run specific mission & FFI tests:
```bash
cargo test -p apodeixis --lib ffi::tests
cargo test -p apodeixis --lib mission_library::tests
```
