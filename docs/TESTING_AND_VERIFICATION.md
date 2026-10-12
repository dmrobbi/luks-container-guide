# Testing and verification — how this vault was proven to work

The complete story of the live testing behind these guides: what was tested, on what,
what failed along the way, what was measured, and what is — and is not — proven.
Written 2026-10-12. Every verdict below is a real command with a real exit code on a
real host — never assumed, never paraphrased into confidence.

## The test environment

- One fleet host (Ubuntu, kernel 6.8) running Docker. A REAL TPM 2.0 is present
  (`/dev/tpm0` + `/dev/tpmrm0`).
- The vault container: `ubuntu:24.04` + `cryptsetup`/`e2fsprogs`, plus `systemd` +
  `tpm2-tools` for the TPM arc (their binaries and the `systemd-tpm2` libcryptsetup
  token plugin are what enrollment and the unseal go through).
- Run **privileged** (LUKS2 + dm-crypt is a kernel-level layer: loop attach +
  device-mapper — no unprivileged path keeps real LUKS), with a named volume at
  `/vault` that outlives the container.
- The vault under test: a 64 MB file, LUKS2, `aes-xts-plain64` with a 256-bit key,
  `pbkdf2` at 1,000,000 iterations; the FAST profile header geometry (keyslots area
  ~16 MiB + 2x16 KiB metadata) was measured in the host-setup review.
- The entry contract under test: pid1 idles with a SIGTERM/SIGINT trap that unmounts
  and `luksClose`s before exit (that is the locked-at-rest mechanic); modes
  `create` / `unlock` / `lock` / `status` / `selftest`, plus the TPM pair
  `tpm-enroll` / `tpm-unlock`. Credentials travel by STDIN only — staged as a real
  0600 file under `/run`, shredded immediately after use — and the output is markers
  only (`VAULT-...` tokens). Nothing secret lands in argv, env, logs, image layers,
  or volumes.
- The credentials used in testing are disposable generic values created for the runs;
  the vaults were destroyed at the end of every cycle. No secret material appears
  anywhere in this repo.

## Arc 1 — the container contract, before anything landed

Timestamps are UTC. The full lifecycle ran live before the first commit, every leg
individually exit-code checked:

destroy (clean start) → build → up → the entry's readiness marker → **selftest** —
the full loop on a throwaway 40 MB scratch image (format, open, one-time filesystem,
canary write/read-back, BOTH negative controls: the wrong credential refused and a
close + leak-check that scans the at-rest image for the canary's plaintext), with the
real vault untouched → **create** (64 MB, the operator's stdin credential) →
**status** = `VAULT-IMG-LUKS2-OK` + `VAULT-LOCKED`, no mapper → **unlock** with the
SAME credential (its `open --test-passphrase` pre-check runs BEFORE the mapper
opens) → mapper + mount → the **canary** write + read-back → **lock** →
**locked-at-rest proof**: the LUKS magic present in the image, NO plaintext of the
canary in the image → the **wrong-credential negative** rejected again on the real
vault → **stop** (the trap locks on the way down) → **restart** →
`VAULT-LOCKED persisted` → re-unlock works → **the canary survived the lock cycle**
→ final stop and teardown.

Every verdict in that paragraph is an exit code, not prose.

## Arc 2 — the recorded demo, itself verified

The same arc was recorded as an asciinema cast, deliberately beat by beat: fresh
start → build → deploy → state probes → create → locked → a WRONG credential shown
hitting cryptsetup's raw line ("No key available with this passphrase.") and getting
the rejection marker → the at-rest ciphertext probe → unlock → the canary → lock →
the locked-connect errors (status inactive, no mapper, no mount) → stop → destroy +
image removal → the final state = GONE.

The recording was then verified ITSELF: the raw artifact serves with HTTP 200 and
CORS allowances, a mirror byte-compares (md5 equality) against the local cast, and a
headless-chromium check clicked the playlist entry and observed the playback state
and timestamps actually advance.

## Arc 3 — the TPM machine-binding

The ask behind it: the containers should only be usable on the specific machines —
and an HA story for a Proxmox cluster or Kubernetes was wanted too. The HA designs
live in `kubernetes.md` (run-first, not yet cluster-proven); this arc is the
machine-binding, which DID run live.

### The design

`systemd-cryptenroll --tpm2-device=auto` seals the payload's disk key to the host's
chip. Measured with `luksDump` after enrollment: a `systemd-tpm2` token —
`tpm2-hash-pcrs: 7` (the SecureBoot / measured-boot state), `tpm2-pcr-bank: sha256`,
`tpm2-primary-alg: ecc`, `tpm2-srk: true` — plus a second keyslot holding the sealed
key. The passphrase slot stays untouched: that is the off-machine and recovery
credential.

The unseal runs through the cryptsetup CLI's `open --token-only` (the systemd-tpm2
plugin consults the chip; nothing on stdin; a closed stdin makes a failed consult
fail fast instead of hanging).

### What failed first — and the fixes

Each catch carries a journal entry in the in-house defect ledger; the patterns
matter more than the list, so the root causes are stated plainly:

1. **The orchestrator never got the dispatch rows.** The entry grew the TPM modes,
   but the host-side orchestrator (the case block that execs into the container) had
   no rows for them: every call printed the usage text and exited 2. Fixed by adding
   the two case rows and the usage rows — and by the habit that any new mode lands
   at BOTH layers in the same change.

2. **The enroll leg hung for minutes.** `systemd-cryptenroll` called without an
   existing credential PROMPTS for it; under a non-tty exec with an (open but empty)
   stdin, that prompt waits forever. The container's process list showed the wedged
   enroll alive but idle. The fix is the same discipline the create/unlock modes
   already follow: the credential arrives by STDIN, is staged 0600, is handed to
   cryptenroll with `--unlock-key-file`, and is destroyed with `shred -u` when done —
   ALWAYS, including on the failure path.

3. **The unseal legs produced FALSE PASSES.** The cycle log showed the mount's
   "bad superblock" error printed, and THEN the success marker — and the canary
   read-back "worked". Both were lies with a mechanism: the TPM path mounted a raw
   payload (the one-time filesystem creation lived only in the passphrase unlock),
   the mount failed, the entry's `set -u` without `-e` let the script continue,
   and the canary "passed" because the file sat in the container's plain
   filesystem instead of the cipher. Fixed three ways at once: the one-time
   filesystem is mirrored into the TPM path, every mount step is explicitly guarded
   and exits on failure with an honest marker, and the whole entry moved to
   `set -euo pipefail` so no failed leg can ever print a success marker again. This
   false-pass arc is the reason the "definitive cycle" below exists: only a run in
   which every leg is honestly counted counts.

4. **The loop pool was starved.** Later legs failed with "Attaching loopback device
   failed (loop device with autoclear flag is required)" — cryptsetup must expose a
   file through a loop device to open a mapper, and none were available. The host's
   loop table: 20 loops attached — 19 snap squashfses (legitimate) and ONE STALE
   LOOP leaked by earlier killed test runs, still bound to a DELETED vault file.
   Detaching the stale loop restored a free slot; a live probe (create a loop,
   release it) confirmed the pool allocates dynamically. Lesson: leaked device
   state from crashed test chains starves real work — the debris belonged to this
   project and was cleaned.

5. **Two self-inflicted edit corruptions.** One stranded a code fence mid-document
   (caught by the post-write read-back), one inserted a spurious line into a
   committed file (caught by `git diff` against the pushed commit and restored from
   it; the corrupted copy never landed). The discipline that caught both: read back
   what you wrote, and verify the tree against the commit you pushed.

### The measured semantics that re-defined the verify chain

After enrollment, the "wrong-credential" open SUCCEEDED. That was investigated
rather than shrugged off:

- Probe 1 — with the token present: `open --test-passphrase` with the wrong
  credential = exit 0.
- Probe 2 — with the token PLUGIN moved aside: STILL exit 0 — the kernel keyring
  carried the volume key from an earlier successful open and satisfies later opens.

Conclusion: post-enrollment, the machine IS the authority. The token path unseals
on the enrolled chip; after any successful open, the volume key sits in the kernel
keyring and re-opens succeed without the token. The wrong-credential rejection is a
PRE-enrollment and OFF-machine property. This is the requirement ("only usable on
the specific machines"), not a bug — and it is why the negatives below are staged
BEFORE enrollment.

One more measured fact fixed the tooling: the `systemd-cryptsetup attach` helper
WEDGES on its ask-password fallback (a FIFO wait) even with stdin closed, after a
failed consult. The unseal therefore uses the cryptsetup CLI's `--token-only`
instead — deterministic, no ask daemon, fails fast.

### THE DEFINITIVE CYCLE — every leg green

| # | Leg | Verdict |
|---|---|---|
| 1 | destroy (clean start) | exit 0 |
| 2 | build; up | 0, 0 |
| 3 | pid1 readiness marker | ENTRY-READY |
| 4 | selftest | `VAULT-SELFTEST-OK`, exit 0 — honestly (under `-e`, healthy loop pool) |
| 5 | create 64 MB (stdin credential) | `VAULT-CREATE-OK` |
| 6 | status | `VAULT-IMG-LUKS2-OK size=67108864`, `VAULT-LOCKED no-mapper` |
| 7 | unlock with the WRONG credential | `VAULT-CREDENTIAL-REJECTED` |
| 8 | tpm-enroll (stdin credential) | "New TPM2 token enrolled as key slot 1." → `VAULT-TPM-ENROLL-OK` |
| 9 | tpm-unlock — NO input on stdin | `VAULT-TPM-UNLOCK-OK` (attach + one-time filesystem + mount) |
| 10 | canary write + read-back | `CANARY_RW-OK` (inside the payload's filesystem) |
| 11 | lock | `VAULT-LOCK-OK`; status `VAULT-LOCKED` |
| 12 | tpm-unlock again | `VAULT-TPM-UNLOCK-OK` |
| 13 | canary read-back | `CANARY-PERSISTED-OK` |
| 14 | lock + leak-check | `NO-PLAINTEXT-AT-REST` — scan of the at-rest image; non-vacuous because the canary verifiably lived inside the cipher |
| 15 | tpm-enroll with an EMPTY credential | "Unlocking via keyfile failed: Operation not permitted" → `VAULT-TPM-ENROLL-FAILED` — the honest negative |
| 16 | foreign-sim: kill the token slot | `TOKEN-SLOT-KILLED`; the header dump shows the `systemd-tpm2` token (PCR7/sha256/ecc/SRK) |
| 17 | tpm-unlock | `VAULT-TPM-UNSEAL-FAILED`, exit 1, IMMEDIATE — the foreign-machine behavior |
| 18 | status | `VAULT-LOCKED` — the vault intact after the simulation |
| 19 | unlock (the passphrase) | `VAULT-UNLOCK-OK` — the recovery credential works |
| 20 | lock; stop; destroy; image removal | the standing cleanup — every verdict 0 |

## What is proven — and what is not

**Proven on a real host with a real chip:**
- the container contract end-to-end: create / credential-gated unlock / use / lock /
  persistence across stop+restart; ciphertext-at-rest with no plaintext leak;
- the TPM seal onto a real TPM 2.0, and the CREDENTIALLESS unseal (twice: fresh and
  after a lock/relock);
- the passphrase recovery after the TPM slot's death;
- the foreign-machine REJECTION via the missing-binding simulation: the unseal fails
  fast (exit 1), the vault stays locked and intact;
- both negatives: the wrong credential (pre-enrollment) and the empty credential.

**Not proven — stated honestly (updated after the cross-chip pair landed):**
- ROOT CAUSE FOUND for the earlier "second stack refuses tool contexts" reading:
  the machines were missing the tpm2-tss runtime libraries (libtss2-esys/mu/rc/
  tcti-device). On the first host an strace caught systemd dlopen'ing
  libtss2-rc.so.0 and friends and finding none — "TPM2 support is not installed";
  installing the runtime set flipped the same tool from that error to listing
  the chip (tpm_crb via /dev/tpmrm0). The second host had already gained the set
  as a side effect of installing tpm2-tools — the refusal stopped reproducing at
  the same moment. The chip, the kernel, and systemd were never the problem.
- The definitive cross-chip pair is now MEASURED (2026-10-12): (a) an enrollment
  on the second host's chip SUCCEEDS (systemd-cryptenall rc=0, "New TPM2 token
  enrolled as key slot 1"); (b) the same image unseals credentiallessly on the
  second host with its own chip (rc=0, mapper up, closed cleanly); (c) shipped
  to the first host, the token-only open REFUSES at the real Esys_Load (TPM
  error 0x1df, "Failed to unseal secret using TPM2: State not recoverable",
  rc=1) — the two chips refuse each other's seals; (d) the passphrase fallback
  opens the same image on both hosts (rc=0). Earlier, still standing: the image
  validates on the other host with its passphrase (the positive control) and the
  token consult fails fast with no mapper there.
- Two process lessons from this arc (reproducible): [1] systemd-cryptenall with
  non-tty stdin does NOT read the unlock credential from stdin — it creates a
  /run/systemd/ask-password session and waits indefinitely (a 90s timeout kill
  exposed the wait); hand the credential via a 0600 stage file +
  --unlock-key-file, then shred it. [2] cryptsetup's stdin passphrase loses its
  trailing newline (line semantics) while --key-file consumes the whole file —
  a "pass\n" keyfile does not match a "pass" bound at luksFormat time and
  surfaces as "Unlocking via keyfile failed: Operation not permitted". Create
  and every later unlock must derive the credential identically.
- The vTPM variant: the fleet's Ubuntu ships the swtpm socket backend but NO
  swtpm_cuse character-device frontend, and the systemd TPM2 tooling consumes
  device TCTIs only — a systemd-usable virtual chip is not buildable here without
  the VM/QEMU route (a separate, heavier leg).

- the HA shapes in `kubernetes.md`: they are documented designs (the physical-chip
  active-passive fleet; the KubeVirt vTPM carrier whose persistent TPM state is a
  since-v1.0 feature with actively maintained live-migration fixes). Neither has
  been through a live cluster failover in this testing. Run-first applies.

## Reproduce it

The drive-it sequence in `docker.md` (with the TPM section) is the reproduce path:
build → selftest → create → the pre-enroll negative → enroll → the credentialless
unseal → the canary → lock/unlock persistence → the leak-check → the slot-kill
foreign-sim → the recovery. Use your own credentials; run in a throwaway volume; the
`destroy --yes` at the end is the deliberate data-destroying act — with intent, by
the operator, every time.