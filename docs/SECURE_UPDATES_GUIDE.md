# MiniGuardian Secure Updates Guide

Signed, proof-checked over-the-air updates for sealed MiniGuardian deployments.

## What it does

- Pulls a signed update manifest from a channel URL.
- Verifies the manifest with a pinned Ed25519 public key.
- Downloads binaries, verifies SHA-256 hashes.
- Runs an embedded **Apodeixis** mission that permits the update only when the host is calm and sealed.
- Applies the update atomically: backup → swap → re-seal → restart → health gate → automatic rollback on failure.

Updates are **disabled by default**. Nothing happens until you configure `[updates]`.

---

## 1. Enable updates on the host

Edit `/etc/miniguard/miniguard.toml` (as root):

```toml
[updates]
enabled = true
channel_url = "https://releases.example.com/miniguard/stable/manifest.json"
public_key = "a1b2c3d4e5f6..."   # 64 hex chars, Ed25519 verifying key
auto_check = true                  # check on startup / periodically
auto_apply = false                 # recommended: require operator approval
allow_rollback = true
staging_dir = "/var/lib/jayce/miniguard/updates"
health_gate_secs = 60
```

Restart the daemon to load the new config:

```bash
sudo systemctl restart miniguard
```

Verify the subsystem is alive:

```bash
sudo sh -c '{
  printf "%s\n" "$(cat /run/jayce-operator/miniguard.sock.token)"
  echo update status
} | socat - UNIX-CONNECT:/run/jayce-operator/miniguard.sock'
```

Expected:

```json
{"enabled":true,"current_version":"0.1.0","staged_version":null,...}
```

---

## 2. Publish an update

### Build the release binaries

```bash
cd /path/to/MiniGuardian/miniguard
cargo build --release --package miniguard
```

Artifacts:

```
../target/x86_64-unknown-linux-musl/release/miniguard
../target/x86_64-unknown-linux-musl/release/miniguard-sentinel
```

### Hash the binaries

```bash
sha256sum ../target/x86_64-unknown-linux-musl/release/miniguard
sha256sum ../target/x86_64-unknown-linux-musl/release/miniguard-sentinel
```

### Build the unsigned manifest

`manifest.json`:

```json
{
  "version": "0.2.0",
  "release_date": 1786320000,
  "binaries": {
    "miniguard": {
      "url": "https://releases.example.com/miniguard/0.2.0/miniguard",
      "hash": "sha256=YOUR_DAEMON_HASH",
      "size": 17555472
    },
    "miniguard-sentinel": {
      "url": "https://releases.example.com/miniguard/0.2.0/miniguard-sentinel",
      "hash": "sha256=YOUR_SENTINEL_HASH",
      "size": 512784
    }
  }
}
```

### Sign the manifest

Use the same Ed25519 secret key whose public key is pinned in `miniguard.toml`.

```python
import json, base64
from nacl.signing import SigningKey

with open("manifest.json", "rb") as f:
    manifest_bytes = f.read()

with open("secret.key", "rb") as f:
    signing_key = SigningKey(f.read())

signature = signing_key.sign(manifest_bytes).signature

signed = {
    "manifest": json.loads(manifest_bytes),
    "signature": base64.b64encode(signature).decode("ascii")
}

with open("manifest.json", "w") as f:
    json.dump(signed, f, indent=2)
```

> The inner `manifest` object is what gets signed. Do not include `signature` in the signed bytes.

Upload the two binaries and the signed `manifest.json` to your `channel_url` path.

---

## 3. Operator commands

All commands require the auth token first, like every other management-socket command.

### Check for an update

```bash
sudo sh -c '{
  printf "%s\n" "$(cat /run/jayce-operator/miniguard.sock.token)"
  echo update check
} | socat - UNIX-CONNECT:/run/jayce-operator/miniguard.sock'
```

On success the daemon downloads the binaries to `/var/lib/jayce/miniguard/updates/<version>/` and replies with the staged path.

### Apply a staged update

```bash
sudo sh -c '{
  printf "%s\n" "$(cat /run/jayce-operator/miniguard.sock.token)"
  echo update apply
} | socat - UNIX-CONNECT:/run/jayce-operator/miniguard.sock'
```

The daemon:

1. Re-verifies signature + hashes.
2. Runs the Apodeixis `safe-update` mission.
3. Stops `miniguard` and `miniguard-sentinel`.
4. Remounts `/usr/local/bin` read-write.
5. Rotates backups (`miniguard.bak.1` ← current, `.bak.1` → `.bak.2`, etc.).
6. Copies staged binaries into `/usr/local/bin/`.
7. Re-applies `chattr +i` and read-only mount.
8. Restarts services.
9. Health-gates on socket + heartbeat.
10. Rolls back automatically if the health gate fails.

### Roll back

```bash
sudo sh -c '{
  printf "%s\n" "$(cat /run/jayce-operator/miniguard.sock.token)"
  echo update rollback
} | socat - UNIX-CONNECT:/run/jayce-operator/miniguard.sock'
```

Restores the most recent backup pair and restarts services.

---

## 4. Security notes

- **Pinned key**: Only the public key in `miniguard.toml` is trusted. A manifest signed by any other key is rejected.
- **Fail-closed policy**: The Apodeixis mission must return `emit_action("permit", "safe_update")` exactly. Any tampered or over-permissive policy is rejected as `MissionFailed`.
- **Sealed requirement**: Updates are rejected unless both binaries and `/etc/miniguard/seal.hashes` are immutable. This prevents downgrades to an unsealed state.
- **Calm requirement**: Updates are delayed (not rejected) when the threat level is not `calm`.
- **Atomic with rollback**: The old binaries remain as `.bak.1` until the next successful update, so a bad release can be undone.
- **No auto-apply by default**: Keep `auto_apply = false` so a compromised channel cannot silently push code.

---

## 5. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `updates are disabled in config` | `[updates].enabled = false` | Enable in `miniguard.toml` and restart. |
| `bad public key hex` | `public_key` is not 64 hex chars. | Re-check the pinned key. |
| `manifest signature verification FAILED` | Wrong signing key or tampered manifest. | Sign with the key whose pubkey is pinned. |
| `hash mismatch for miniguard` | Binary on the server differs from manifest hash. | Re-hash and re-sign. |
| `update policy veto: reject` | Host not sealed, not calm, or signature/hash invalid. | Check seal posture with `seal` socket command. |
| `health gate failed; rollback executed` | New daemon did not start or write heartbeat. | Check `journalctl -u miniguard -n 50`; inspect `.bak.1`. |

---

## Files involved

- `miniguard/src/update_manager.rs` — core logic, Apodeixis policy, signature/hash verification.
- `miniguard/src/socket_server.rs` — `update status|check|apply|rollback` commands.
- `miniguard/src/config.rs` — `[updates]` config section.
- `AGENTS.md` — terse reference.

---

## 6. Master Console Design Checklist (personal notes)

Cloud-based fleet control plane that uses the same MiniGuardian architecture and intelligence, learns from connected leaf nodes, and pushes updates/security definitions back out.

### Phase 1 — Fleet telemetry ingestion
- [ ] Leaf nodes send compressed telemetry to the master over gossip or HTTPS.
- [ ] Telemetry includes: threat alerts, blocked IPs, memory patterns, impossible-state hits, freeze events, seal posture.
- [ ] Master stores aggregated corpus in `/var/lib/jayce/miniguard/fleet/`.
- [ ] Per-node identity + auth token issued at join time.

### Phase 2 — Collective learning
- [ ] Master runs existing detectors over the aggregated corpus.
- [ ] Identify new attack patterns / IoCs from the fleet.
- [ ] Produce signed security-definition bundle:
  - blocked IPs
  - trusted/untrusted hashes
  - detection threshold tweaks
  - new Apodeixis policy revisions
- [ ] Human-in-the-loop approval before publishing critical defs.

### Phase 3 — Distribution
- [ ] Master serves:
  - `/manifest.json` — signed binary update manifest (done).
  - `/defs.json` — signed security-definition bundle.
- [ ] Leaves pull `manifest.json` and `defs.json` periodically.
- [ ] Apply defs without requiring daemon restart where possible.
- [ ] Emergency revocation path for compromised nodes or bad defs.

### Phase 4 — Master console binary / UI
- [ ] Build `miniguard-master` or `--master` mode.
- [ ] Fleet dashboard:
  - connected nodes
  - seal posture per node
  - threat level per node
  - pending updates
  - aggregate attack memory
- [ ] Operator commands:
  - approve/block leaf node
  - push emergency revocation
  - publish new manifest
  - publish new defs bundle

### Open questions to resolve
- [ ] Gossip-only vs HTTPS API for leaf-master control?
- [ ] Should leaves peer with each other, or only with the master (star topology)?
- [ ] How to handle offline nodes that come back later (delta sync)?
- [ ] Key rotation strategy for the signing key?
- [ ] Quorum/consensus before auto-publishing learned defs?

---

## 7. Config editing notes (so I don't mess it up later)

`/etc/miniguard/miniguard.toml` is sealed with `chattr +i` by `scripts/seal.sh`. You must release the immutable flag before editing and re-apply it after.

### Add `[updates]` for the first time

Run this **as one paste** (do not paste the TOML lines by themselves — the shell will try to run them):

```bash
sudo chattr -i /etc/miniguard/miniguard.toml && \
sudo tee -a /etc/miniguard/miniguard.toml > /dev/null <<'EOF'

[updates]
enabled = true
channel_url = "https://your-real-domain.com/stable/manifest.json"
public_key = "YOUR_REAL_64_HEX_ED25519_PUBKEY"
auto_check = true
auto_apply = false
allow_rollback = true
EOF
sudo chattr +i /etc/miniguard/miniguard.toml && \
sudo systemctl restart miniguard
```

### Update an existing `[updates]` section

```bash
sudo chattr -i /etc/miniguard/miniguard.toml && \
sudo python3 - <<'PY'
import re
path = "/etc/miniguard/miniguard.toml"
with open(path) as f:
    txt = f.read()
txt = re.sub(r'public_key = ".*"', 'public_key = "YOUR_REAL_64_HEX_ED25519_PUBKEY"', txt)
txt = re.sub(r'channel_url = ".*"', 'channel_url = "https://your-real-domain.com/stable/manifest.json"', txt)
with open(path, "w") as f:
    f.write(txt)
PY
sudo chattr +i /etc/miniguard/miniguard.toml && \
sudo systemctl restart miniguard
```

### Disable updates temporarily

```bash
sudo chattr -i /etc/miniguard/miniguard.toml && \
sudo sed -i 's/^enabled = true/enabled = false/' /etc/miniguard/miniguard.toml && \
sudo chattr +i /etc/miniguard/miniguard.toml && \
sudo systemctl restart miniguard
```

### Verify after any change

```bash
sudo sh -c '{
  printf "%s\n" "$(cat /run/jayce-operator/miniguard.sock.token)"
  echo update status
} | socat - UNIX-CONNECT:/run/jayce-operator/miniguard.sock'
```

Expected: `"enabled":true` (or `false` if you disabled it) and `"current_version":"0.1.0"`.

