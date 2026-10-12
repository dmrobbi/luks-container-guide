# luks-container-guide — LUKS2 setup + container instructions (Docker / Podman / Kubernetes)

In-house, verified instructions for building, unlocking, and re-locking a LUKS2
container image inside Docker, Podman, and Kubernetes runtimes, plus the host-level
LUKS2 setup behind it.

The workflow arc is credited to cyberanchor's public LUKS gist
`6535a5b38ceb4c57986d4c8324e0a94c` ("CryptSetup | LUKS cryptocontainer CheatSheet",
2025-04-30 / updated 2026-10-10, **no license**): nothing is copied from it — every
command here is re-derived and re-verified in-house, and the shapes below carry the
six corrections the in-house review found in that sheet.

**Verification status, stated plainly:**

| Guide | Status |
|---|---|
| `docker.md` | LIVE-verified end-to-end on the fleet host (build → create → unlock → lock → stop → restart → the locked vault persisted → re-unlock) |
| `podman.md` | Adapted equivalents of the verified Docker flow (the compose validation precedent); run the loop on your Podman host first |
| `kubernetes.md` | The manifest + command shapes adapted from the verified flow; privileged-pod caveats stated; verify on your cluster first |

## The guide map

| File | Covers |
|---|---|
| [`docs/LUKS_HOST_SETUP.md`](docs/LUKS_HOST_SETUP.md) | LUKS2 itself: the on-disk method, the two profiles, the lifecycle, keyslots/rotation, header geometry + wipes, backup/restore, detached headers |
| [`docker.md`](docker.md) | The vault container on Docker: locked-at-rest / open-while-running, the stop mechanic, credentials-by-stdin |
| [`podman.md`](podman.md) | The same contract on Podman (rootless-host, root-container): volumes, --privileged, the TERM trap |
| [`kubernetes.md`](kubernetes.md) | The same contract on Kubernetes: a privileged single-replica pod + a PVC; `kubectl exec -i` for credentials |

## The one-paragraph method

LUKS2 is Linux's standard on-disk encryption for a block-like container: a random
master key encrypted by XTS-AES wraps your payload, and up to 32 keyslots in the
header each store the master key under a slow-KDF-derived binding of one credential
(passphrase or key file). Any single working credential opens the volume; the kernel's
dm-crypt layer then presents a plaintext mapper device that behaves like a normal
filesystem. Destroy or detach the header and the data is unreachable by design.

## The container contract (all three runtimes)

1. The LUKS2 image lives inside runtime storage (a named volume / a named volume or
   directory / a PVC).
2. While the container runs, an operator can unlock it (credential in, mapper open,
   filesystem mounted rw) and read/write through the mapper.
3. When the container stops — by hand, by crash, or by platform lifecycle — the
   entry's SIGTERM trap unmounts + `luksClose`s first. After that the image holds
   only ciphertext; the master key exists solely in kernel dm-crypt state while open,
   and the credentials solely in the header.
4. Credentials travel by stdin only: never in argv, env, logs, image layers, or
   manifests.

## Safety rails that apply everywhere

- Privileged container/pod is required: LUKS2 + dm-crypt is a kernel-level layer
  (loop attach + device-mapper). No unprivileged path keeps real LUKS. That trade is
  documented, not hidden — treat the vault pod/container with the same trust as the
  host.
- File-backed containers only in the guides; block-device ops need owner-signed
  consent.
- No secret material is committed anywhere; the passphrase placeholder is a
  placeholder.
- The two verified trapdoors: sudo + process-substitution key files fail (stage a
  real 0600 file); `--key-file` reads raw bytes including any trailing newline.

## License

No LICENSE file yet: an owner call (the standing convention across the fleet's repos).