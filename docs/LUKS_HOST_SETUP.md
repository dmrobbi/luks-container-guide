# LUKS2 host setup — the verified guide

A condensed, verified host-side guide to the LUKS2 container method. The workflow arc
is credited to cyberanchor's LUKS gist (`6535a5b38ceb4c57986d4c8324e0a94c`, no
license — nothing copied). Verified on Ubuntu 24.04 with cryptsetup 2.7.0 /
e2fsprogs 1.47.0; file-backed containers only unless you have consent for block
devices.

## Tooling notes (Ubuntu)

- `cryptsetup` and `mkfs.ext4` live in `/usr/sbin` — the user PATH usually omits
  sbin, so call the binaries by absolute path (or via sudo). A bare `cryptsetup`
  "command not found" is a PATH artifact, not a missing package.
- File-backed container ops need no root: `luksFormat`, `luksAddKey`,
  `luksKillSlot`, `luksDump`, `luksHeaderBackup/Restore`, `open --test-passphrase`.
- Root (or a privileged container) is only needed for mapper activation:
  `luksOpen`/`luksClose`, `mkfs`, `mount`.

## Create a container — two verified profiles

```bash
dd if=/dev/urandom of=vault.img bs=1M count=64 status=none   # real bulk bytes, not sparse
chmod 600 vault.img

# FAST: PBKDF2-SHA256, 256-bit, 1,000,000 force-iterations
/usr/sbin/cryptsetup luksFormat --type luks2 \
    --cipher aes-xts-plain64 --key-size 256 --hash sha256 \
    --pbkdf pbkdf2 --pbkdf-force-iterations 1000000 --use-random \
    vault.img --key-file=key --batch-mode

# SECURE: Argon2id, 512-bit — the 1 GiB memory default
/usr/sbin/cryptsetup luksFormat --type luks2 \
    --cipher aes-xts-plain64 --key-size 512 --hash sha512 \
    --pbkdf argon2id --pbkdf-memory 1048576 --pbkdf-parallel 8 --iter-time 5000 \
    --use-random --sector-size 4096 \
    --keyslot-cipher aes-xts-plain64 --keyslot-key-size 512 \
    --luks2-keyslots-size 16m --luks2-metadata-size 256k \
    vault.img --key-file=key --batch-mode
```

- FAST = quick format/unlock on any CPU; SECURE = slow and memory-hard (punishes
  GPU/offline guessing). Default Argon2id memory = 1 GiB; 4 GiB (`--pbkdf-memory
  4194304`) is an explicit RAM-rich variant only — it must fit a single unlock's
  memory spike.

## Lifecycle

```bash
printf '%s' '<passphrase>' > key && chmod 600 key     # real 0600 file, NOT process substitution
/usr/sbin/cryptsetup luksOpen vault.img vaultvol --key-file=key
/usr/sbin/mkfs.ext4 -q /dev/mapper/vaultvol                # first unlock only
mkdir -p /mnt/vaultvol && mount /dev/mapper/vaultvol /mnt/vaultvol
# ... use ...
umount /mnt/vaultvol && /usr/sbin/cryptsetup luksClose vaultvol
```

## Keyslots: add / test / rotate / kill

- LUKS2 carries up to 32 slots; any single populated slot activates the volume;
  convention = slot 0 as the primary, higher slots for recovery keys.
- Add: authenticated with an existing key, new credential on stdin or a file:

```bash
printf '%s' '<new-passphrase>' | /usr/sbin/cryptsetup luksAddKey vault.img \
    --key-slot 3 --key-file=key_existing --new-keyfile=- --batch-mode
```

- Test any slot anytime:

```bash
/usr/sbin/cryptsetup open --test-passphrase vault.img --key-slot 3 --key-file=key
```

- Rotation ("replace slot 0") is destructive by shape — stage + TEST first, then
  confirm before the kill:

```bash
/usr/sbin/cryptsetup luksKillSlot vault.img 0 --key-file=key_slot3 --batch-mode
```

- Rules that keep you locked out less: keep a recovery key on a free slot BEFORE
  rotating/killing anything; never kill the last working slot; a kill authenticates
  with a DIFFERENT slot's credential than the slot being killed.

## Complex passphrases

Machine-injected credentials with metacharacters: pipe exact bytes on stdin
(`--key-file=-` / `--new-keyfile=-`), never inline-quote into argv, and verify the
byte stream first:

```bash
printf '%s' '<complex-passphrase>' | hexdump -C
printf '%s' '<complex-passphrase>' | /usr/sbin/cryptsetup luksAddKey vault.img ... --new-keyfile=-
```

No example passphrases belong in any repo; material stays out of the tree.

## Header geometry — why "wipe the first 4 MB" is wrong

LUKS2's header = two metadata copies + the keyslots area, then the payload. On the
verified FAST-profile container: metadata 16384 bytes x2 + keyslots 16744448 bytes ≈
16.5 MiB. The reference sheet's `bs=1M count=4` overwrite leaves most of the header —
and its keyslots — on disk and recoverable. Derive the length from geometry:

```bash
HDR_BYTES=$(/usr/sbin/cryptsetup luksDump vault.img | awk '
  /Metadata area:/  { meta = $3 }  /Keyslots area:/ { ks = $3 }
  END { printf "%d", 2 * meta + ks + 1048576 }')        # + 1 MiB margin
dd if=/dev/zero of=vault.img bs=1M count=$(( HDR_BYTES / 1048576 )) conv=notrunc status=none
```

(Deleting a header makes the content unreachable by design; back up first if the
wipe should stay reversible.)

## Header backup / restore

```bash
/usr/sbin/cryptsetup luksHeaderBackup vault.img --header-backup-file header.bak
/usr/sbin/cryptsetup luksHeaderRestore vault.img --header-backup-file header.bak
```

A header backup is a full unlock oracle for every keyslot it carries: keep it
OFFLINE, chmod 600, on media separate from the container.

## Detached headers

```bash
/usr/sbin/cryptsetup luksFormat --type luks2 --header header.bin vault.img --key-file=key --batch-mode
/usr/sbin/cryptsetup luksOpen --header header.bin vault.img vaultvol
/usr/sbin/cryptsetup luksHeaderBackup vault.img --header-backup-file header.bin
/usr/sbin/cryptsetup luksDump --header header.bin vault.img
```

Without the header file the container is inert. Two copies on separate media; test a
RESTORE (restore-then-`--test-passphrase`) before relying on detached storage.

## Image-derived key files (methodology + honesty)

A key file can be derived from staged image bytes in a fixed byte envelope
(prefix-marker || image || suffix-marker), with SHA-512 pins on BOTH the image and
the derived key. Classify it as **obscurity, not secrecy** — anyone with the envelope
parameters and the image reproduces the key. It is a decoy/deception layer around a
real key, never a replacement; never re-download third-party media at use time.

## BIP-39/seed-phrase passphrases — methodology + warning

The reference derives disk passphrases from BIP-39 mnemonics through a web tool. Do
NOT couple wallet-recovery material to disk encryption: it doubles the blast radius
of either secret leaking and routes a high-value secret through a browser context. If
a flow ever REQUIRES it: offline live-USB host, a purpose-generated mnemonic, local
on-box derivation, never a mnemonic in use as a wallet. No mnemonic/seed/passphrase
material is committed anywhere.

## The six corrections vs the reference sheet

| # | Reference behavior | This guide |
|---|---|---|
| 1 | Secure profile pins 4 GiB memory; its own note says 1 GiB | 1 GiB default; 4 GiB = explicit variant |
| 2 | "Add to slot 0" begins with `luksKillSlot 0` | Renamed rotation: stage + test + confirm gate |
| 3 | Fixed 4 MB header wipe | Wipe length from `luksDump` geometry (+ margin) |
| 4 | BIP-39 → web tool → passphrase as routine | Methodology + warning; no material committed |
| 5 | Arch/pacman assumed | Host facts verified (Ubuntu; sbin PATH artifact) |
| 6 | Inconsistent newline handling on stdin keys | Raw-bytes rule stated + verified by rejection |

## Full-loop verification

`scripts/ops/luks_verify.sh` (in the fleet tree) runs a 21-check PASS/FAIL matrix
over this entire arc — format both profiles, open/format/mount, canary round-trip,
key/slot ops incl. rotation, per-type test-passphrases, header backup/restore
round-trips, detached-header cycles, cleanup. Recorded run: 21 PASS / 0 FAIL / 0 SKIP.