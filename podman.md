# Podman — the LUKS2 vault container

The SAME container contract as the verified Docker flow, driven by Podman. The fleet's
verified precedent so far: the compose file's Podman variant was validated with
`docker compose v2` (the obsolete-version warning is harmless), while Podman itself was
not installed on the fleet host at probe time — so treat this as the adapted contract
and run the loop on your own Podman host first.

## What stays the same

- The LUKS2 image lives inside RUNTIME storage (a named volume), created at runtime;
  nothing is baked into image layers.
- pid1 idles with a SIGTERM/SIGINT trap: at `podman stop` it unmounts + `luksClose`s
  first — after that the image holds only ciphertext (locked at rest).
- Credentials travel by stdin only (the same `vault_entry.sh` from the Docker guide;
  no fleet-specific changes to the script itself).

## What changes vs Docker (rootful mode, rootless host)

- **Volume**: `podman volume create luks-vault-vol` + mount it at `/vault`
  (`-v luks-vault-vol:/vault:Z` — Podman's SELinux relabel flag).
- **Privileged**: LUKS2 + dm-crypt needs kernel-level access (loop attach +
  device-mapper) exactly as in Docker. Plain `--privileged` works for both Podman run
  modes; for rootless Podman this activates Podman's user-namespace shims — the
  mapper/loop layer runs through them, so the same script contract holds. Treat the
  vault container with host-level trust either way.
- **Registry-free local build**: `podman build` + Podman's local storage. No
  docker-compatible daemon needed.
- **Compose driver**: podman-compose, or Podman's own quadlet/compose support; the
  simplest verified path = the direct `podman` commands (below). If you prefer
  compose: the SAME `docker-compose.yml` from the Docker guide works via
  `podman-compose up` — the fleet's validated precedent.

## Direct Podman flow (no compose)

```bash
# build (Dockerfile + vault_entry.sh identical to the Docker guide)
podman build -t luks-vault:1.0 .

# storage + entry
podman volume create luks-vault-vol
podman run -d --name luks-vault --privileged \
    -v luks-vault-vol:/vault:Z \
    --stop-timeout 30 \
    luks-vault:1.0

# the scratch self-test (the container-side /tmp loop)
podman exec -T luks-vault vault_entry selftest

# create the vault: the keyslot binds the operator's stdin credential
printf '%s' '<pass>' | podman exec -i -T luks-vault vault_entry create 64

podman exec -T luks-vault vault_entry status           # VAULT-LOCKED

# unlock + use
printf '%s' '<pass>' | podman exec -i -T luks-vault vault_entry unlock
podman exec -T luks-vault vault_entry status           # VAULT-OPENED
# ... read/write /mnt/vaultvol via podman exec ...

podman exec -T luks-vault vault_entry lock             # VAULT-LOCK-OK
podman stop luks-vault                                 # the trap locks (idempotent)
podman start luks-vault
podman exec -T luks-vault vault_entry status           # VAULT-LOCKED persisted

# gone for good (removes the storage too; refuses = your intent check first)
podman volume rm luks-vault-vol
```

- `podman exec -i -T` = the stdin pipe (the credential's only path, as in Docker's
  `exec -T`).
- `--stop-timeout 30` = Docker's `stop_grace_period` equivalent (the trap gets its
  time before SIGKILL).
- Podman preserves SIGTERM to pid1 at `podman stop` — so your trap receives it and
  locks.

## The two trapdoors + the rails (the same everywhere)

1. Stage the credential as a real 0600 file — sudo + process-substitution key files
   fail; and inside a rootful/rootless container, run the entry AS ROOT (the
   container's own user) rather than adding sudo: the script contract stays
   sudo-free.
2. `--key-file` = RAW BYTES incl. the trailing newline: `echo -n` vs `echo` changes
   the key.
3. No secret material committed anywhere; the placeholder is a placeholder.

## Verification checklist (run before trusting your host)

1. `podman exec luks-vault sh -c 'cryptsetup --version'` — cryptsetup present.
2. The selftest's full loop = `VAULT-SELFTEST-OK` (format/unlock/mount/canary/
   negatives/lock/leak-check).
3. `podman stop` → `podman start` → the status = `VAULT-LOCKED` and the re-unlock
   works with the SAME credential (the locked vault persisted).