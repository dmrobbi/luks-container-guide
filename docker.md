# Docker — the LUKS2 vault container

**Verified live** end-to-end on the fleet host (2026-10-12): build → the locked vault
created inside a named volume → unlock with the operator's credential → the canary
read/write → lock → `docker compose stop` (the trap locks on the way down) → full
restart → `VAULT-LOCKED persisted` → re-unlock with the same credential → the canary
survived the lock cycle.

## The mechanic

- A container (`ubuntu:24.04` + cryptsetup/e2fsprogs) runs `--privileged LUKS2 +
  dm-crypt is a kernel-level encryption layer: loop attach + device-mapper. No
  unprivileged path keeps real LUKS; the trade is documented, not hidden.
- The LUKS2 image (`vault.img`) lives inside a NAMED Docker volume mounted at
  `/vault` — created at runtime, never baked into the image layer, persistent across
  stop/down/container removal (the volume outlives the container).
- pid1 = an interruptible idle loop whose SIGTERM/SIGINT trap unmounts +
  `luksClose`s before exit: `docker compose stop` LOCKS the vault on the way down.
  After that the image holds only ciphertext.
- Credentials travel by STDIN only: `unlock` verifies the credential with `open
  --test-passphrase` BEFORE the mapper opens; the staged key = a real 0600 file under
  `/run`, shredded right after; output = markers only (`VAULT-UNLOCK-OK`,
  `VAULT-CREDENTIAL-REJECTED`). Nothing secret lands in argv/env/logs/layers/volumes.

## Minimal self-contained version

Any `ubuntu:24.04` machine with Docker can run this without the fleet tree:

**Dockerfile**

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y --no-install-recommends \
        cryptsetup e2fsprogs util-linux bash coreutils \
    && rm -rf /var/lib/apt/lists/*
COPY vault_entry.sh /usr/local/bin/vault_entry
RUN chmod 0755 /usr/local/bin/vault_entry
ENTRYPOINT ["/usr/local/bin/vault_entry"]
CMD ["wait"]
```

**docker-compose.yml**

```yaml
services:
  luks-vault:
    build: .
    container_name: luks-vault
    privileged: true          # kernel-layer encryption: loop + device-mapper
    volumes:
      - luks-vault-vol:/vault
    stop_grace_period: 30s
volumes:
  luks-vault-vol:
    name: luks-vault-vol      # the LOCKED vault persists here, independent of the container
```

**vault_entry.sh** — the entry with all five modes + the stop trap:

```bash
#!/usr/bin/env bash
set -u
VAULT_DIR=/vault; IMG="$VAULT_DIR/vault.img"; MAPPER=vaultvol; MNT=/mnt/$MAPPER
CS=/usr/sbin/cryptsetup; MKFS=/usr/sbin/mkfs.ext4; TUNE=/usr/sbin/tune2fs

_lock_now() {
    mountpoint -q "$MNT" 2>/dev/null && umount "$MNT"
    "$CS" status "$MAPPER" >/dev/null 2>&1 && "$CS" luksClose "$MAPPER"
    return 0
}
trap '_lock_now; exit 0' TERM INT

case "${1:-}" in
  wait)     touch /run/vault-entry-ready; while :; do sleep 60 & wait $!; done ;;
  create)   # [stdin: passphrase] <size_mb>  -> the keyslot binds YOUR credential
            size="$2"; case "$size" in ''|[!0-9]*) exit 2;; esac
            [ -e "$IMG" ] && { echo exists >&2; exit 1; }
            cat > /run/stage.key; chmod 600 /run/stage.key
            [ -s /run/stage.key ] || exit 1
            dd if=/dev/urandom "of=$IMG" bs=1M "count=$size" status=none; chmod 600 "$IMG"
            "$CS" luksFormat --type luks2 --cipher aes-xts-plain64 --key-size 256 \
                --hash sha256 --pbkdf pbkdf2 --pbkdf-force-iterations 1000000 \
                --use-random "$IMG" --key-file=/run/stage.key --batch-mode
            shred -u /run/stage.key; echo "VAULT-CREATE-OK size_mb=$size" ;;
  unlock)   [ ! -e "$IMG" ] && { echo "none" >&2; exit 1; }
            cat > /run/unlock.key; chmod 600 /run/unlock.key
            if ! "$CS" open --test-passphrase "$IMG" --key-file=/run/unlock.key >/dev/null 2>&1; then
                shred -u /run/unlock.key; echo "VAULT-CREDENTIAL-REJECTED"; exit 1
            fi
            "$CS" luksOpen "$IMG" "$MAPPER" --key-file=/run/unlock.key; shred -u /run/unlock.key
            "$TUNE" -l "/dev/mapper/$MAPPER" >/dev/null 2>&1 \
                || "$MKFS" -q "/dev/mapper/$MAPPER"       # one-time payload fs
            mkdir -p "$MNT"; mount "/dev/mapper/$MAPPER" "$MNT"
            echo "VAULT-UNLOCK-OK mapper=$MAPPER mnt=$MNT" ;;
  lock)     _lock_now; echo "VAULT-LOCK-OK" ;;
  status)   [ ! -e "$IMG" ] && { echo "VAULT-NONE"; exit 1; }
            "$CS" isLuks "$IMG" && echo "VAULT-IMG-LUKS2-OK size=$(stat -c%s "$IMG")"
            "$CS" status "$MAPPER" >/dev/null 2>&1 && echo "VAULT-OPENED" || echo "VAULT-LOCKED" ;;
  selftest) # the full loop on a /tmp scratch image; the real vault untouched
            img=/tmp/st.img; rm -f "$img"
            dd if=/dev/urandom "of=$img" bs=1M count=40 status=none
            printf '%s' 'st-cred' > /tmp/sk; chmod 600 /tmp/sk
            "$CS" luksFormat --type luks2 --cipher aes-xts-plain64 --key-size 256 \
                --hash sha256 --pbkdf pbkdf2 --pbkdf-force-iterations 1000000 \
                --use-random "$img" --key-file=/tmp/sk --batch-mode
            "$CS" luksOpen "$img" stvol --key-file=/tmp/sk
            "$MKFS" -q /dev/mapper/stvol; mkdir -p /mnt/st; mount /dev/mapper/stvol /mnt/st
            printf 'canary\n' > /mnt/st/c.txt
            [ "$(cat /mnt/st/c.txt)" = canary ] || exit 1
            printf '%s' 'wrong' > /tmp/w; chmod 600 /tmp/w
            "$CS" open --test-passphrase "$img" --key-file=/tmp/w 2>/dev/null && exit 1
            umount /mnt/st && "$CS" luksClose stvol
            [ "$("$CS" status stvol 2>/dev/null)" ] && exit 1
            grep -a -q canary "$img" && { echo leaked >&2; exit 1; }
            rm -f "$img" /tmp/sk /tmp/w; echo "VAULT-SELFTEST-OK" ;;
  *)        exec usage-echo-with-your-text ;;
esac
```

(Replace the last line's placeholder with your own usage text; full field-tested
version = `scripts/ops/luks_vault/vault_entry.sh` in the fleet tree.)

## Drive it

```bash
docker compose build
docker compose up -d
docker compose exec -T luks-vault vault_entry selftest    # the scratch self-test
printf '%s' '<pass>' | docker compose exec -T luks-vault vault_entry create 64
docker compose exec -T luks-vault vault_entry status      # VAULT-LOCKED
printf '%s' '<pass>' | docker compose exec -T luks-vault vault_entry unlock
docker compose exec -T luks-vault vault_entry status      # VAULT-OPENED
# ... read/write /mnt/vaultvol via exec ...
docker compose exec -T luks-vault vault_entry lock        # VAULT-LOCK-OK
docker compose exec -T luks-vault vault_entry status      # VAULT-LOCKED (ciphertext at rest)
docker compose stop                                       # the trap locks (idempotent)
docker compose up -d
docker compose exec -T luks-vault vault_entry status      # VAULT-LOCKED persisted; unlock works again
docker compose down -v                                    # ONLY with intent: removes the volume (data gone)
```

## The two verified trapdoors (standing rules)

- stage the credential as a real 0600 file; sudo + process-substitution key files
  fail (sudo drops inherited fds — `/dev/fd/N` cannot resolve there).
- `--key-file` reads RAW BYTES, trailing newline included: `echo -n` vs `echo`
  produces two different keys by design.