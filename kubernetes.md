# Kubernetes — the LUKS2 vault pod

The SAME container contract as the verified Docker flow, driven by Kubernetes. Stated
plainly: the fleet's LIVE verification ran on Docker (see `docker.md`); this guide is
the adapted shape for Kubernetes — run the loop on your own cluster before trusting
it. The privileged-pod trade is the same one documented in every guide.

## The mapping

| Docker concept | Kubernetes concept |
|---|---|
| named volume `/vault` | a **PersistentVolumeClaim** (`ReadWriteOnce`) — outlives pods |
| `--privileged` container | `securityContext.privileged: true` on the pod's container |
| `docker compose stop` / `stop_grace_period` | pod deletion + `terminationGracePeriodSeconds` — k8s sends SIGTERM to pid1, and the trap locks first |
| `docker compose exec -T` | `kubectl exec -i` (stdin) / `kubectl exec -it` |
| container restart | a **Deployment, `replicas: 1`, strategy `Recreate`** (never two replicas on one RWO volume) |

## Prerequisites

1. An image the cluster can pull: build the vault image (the Dockerfile +
   `vault_entry.sh` from the Docker guide — identical, no changes) and PUSH it to a
   registry you control (e.g. `ghcr.io/<owner>/luks-vault:1.0` — the placeholder is a
   placeholder). Edit the manifest's image row to yours.
2. A namespace allowed to run privileged pods: the default PodSecurity is
   `baseline`/`restricted`, which REFUSES privileged; label the namespace
   `pod-security.kubernetes.io/enforce=privileged` (or use your own trusted
   namespace). This is admin-level intent — the vault pod deserves host-trust
   handling exactly like the Docker case.

## The manifest

`manifests/luks-vault.yaml` (next to this guide): Namespace + PVC (128Mi,
ReadWriteOnce) + a single-replica Deployment (strategy Recreate, privileged container,
the vault volume mounted at `/vault`, 30 s termination grace).

```bash
kubectl apply -f manifests/luks-vault.yaml
kubectl -n luks-vault get pod -w          # wait for Running/Ready
```

## Drive it (the same loop)

```bash
# the scratch self-test (container-side /tmp; the real vault untouched)
kubectl -n luks-vault exec deploy/luks-vault -- vault_entry selftest

# create the vault: the keyslot binds the operator's stdin credential
printf '%s' '<pass>' | kubectl -n luks-vault exec deploy/luks-vault -i -T -- vault_entry create 64

kubectl -n luks-vault exec deploy/luks-vault -- vault_entry status       # VAULT-LOCKED

# unlock + use
printf '%s' '<pass>' | kubectl -n luks-vault exec deploy/luks-vault -i -T -- vault_entry unlock
kubectl -n luks-vault exec deploy/luks-vault -- vault_entry status       # VAULT-OPENED
# ... read/write /mnt/vaultvol via kubectl exec ...

kubectl -n luks-vault exec deploy/luks-vault -- vault_entry lock         # VAULT-LOCK-OK
kubectl -n luks-vault exec deploy/luks-vault -- vault_entry status       # VAULT-LOCKED

# the locked-at-rest data survives pod deletion (the PVC outlives the pod):
kubectl -n luks-vault delete pod -l app=luks-vault
kubectl -n luks-vault get pod -w                                         # the new pod Running
kubectl -n luks-vault exec deploy/luks-vault -- vault_entry status       # VAULT-LOCKED persisted
printf '%s' '<pass>' | kubectl -n luks-vault exec deploy/luks-vault -i -T -- vault_entry unlock
```

## The k8s-specific caveats (the honest list)

- **SIGTERM is pod-lifecycle**: on delete/eviction/scale-down k8s sends SIGTERM to
  pid1 and waits `terminationGracePeriodSeconds` (the manifest = 30 s) before
  SIGKILL. The trap locks the vault in that window. A hard node crash = no SIGTERM:
  the mapper dies with the node — which still leaves the vault LOCKED (ciphertext on
  disk), the correct failure direction.
- **The PVC = the persistent boundary**: `delete pod` keeps the locked vault;
  `kubectl delete pvc` (or PV reclaim) is the data-destroying act — treat it like the
  Docker guide's `down -v`.
- **Privileged = admin-level intent**: the pod mounts the node's loop/device-mapper
  surface via the unprivileged-file contract only — but the securityContext itself is
  privileged; keep the namespace locked down, and audit who can exec into it.
- **Do not run more than one replica**: the RWO volume + a single mapper contract =
  `strategy: Recreate`; two replicas would fight over the loop/mapper.
- The two trapdoors + the raw-bytes rule apply unchanged (the Docker guide carries
  them); no secret material is committed anywhere — credentials at runtime stdin
  only.

## Verification checklist (run before trusting your cluster)

1. The pod reaches Ready (the entry's `/run/vault-entry-ready` touch confirms pid1
   came up).
2. `kubectl exec ... -- vault_entry selftest` = `VAULT-SELFTEST-OK`.
3. Pod deletion → the new pod → `VAULT-LOCKED` + the re-unlock with the SAME
   credential works (the locked vault persisted).