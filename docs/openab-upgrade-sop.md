# OpenAB Version Upgrade SOP

| | |
|---|---|
| **Document Version** | 1.3 |
| **Last Updated** | 2026-04-14 |

## Environment Reference

| Item | Details |
|---|---|
| Deployment Method | Kubernetes (Helm Chart) |
| Deployment Name | `<release-name>-kiro` (default: `openab-kiro`) — see note below |
| Pod Label (precise) | `app.kubernetes.io/instance=openab,app.kubernetes.io/name=openab-kiro` |
| Helm Repo (OCI, recommended) | `oci://ghcr.io/openabdev/charts/openab` |
| Helm Repo (GitHub Pages, fallback) | `https://openabdev.github.io/openab` |
| Image Registry | `ghcr.io/openabdev/openab` |
| Git Repo | `github.com/openabdev/openab` |
| Agent Config | `/home/agent/.kiro/agents/default.json` |
| Steering Files | `/home/agent/.kiro/steering/` |
| kiro-cli Auth DB | `/home/agent/.local/share/kiro-cli/data.sqlite3` |
| PVC Mount Path | `/home/agent` (Helm); `.kiro` / `.local/share/kiro-cli` (raw k8s) |
| KUBECONFIG | `~/.kube/config` (must be set explicitly — default k3s config has insufficient permissions) |
| Namespace | `default` (adjust to match your actual deployment namespace) |

> **Deployment naming pattern:** The deployment name follows `<release-name>-kiro`. For the default setup (`helm install openab …`), the deployment is `openab-kiro`. If you used a different release name (e.g. `my-bot`), the deployment is `my-bot-kiro`. Verify with:
> ```bash
> RELEASE_NAME=$(helm list -o json | jq -r '.[0].name')
> DEPLOYMENT="${RELEASE_NAME}-kiro"
> echo "Deployment: $DEPLOYMENT"
> ```

> ⚠️ The local kubectl defaults to reading `/etc/rancher/k3s/k3s.yaml`, which will result in a permission denied error. Before running any command, always set:
> ```bash
> export KUBECONFIG=~/.kube/config
> ```

> 💡 **Namespace setup (recommended):** If OpenAB is deployed in a non-default namespace, set the following at the start of your session to avoid having to append `-n <namespace>` to every command:
> ```bash
> export NS=openab          # replace with your actual namespace
> export KUBECONFIG=~/.kube/config
> alias kubectl="kubectl -n $NS"
> alias helm="helm -n $NS"
> ```
> All `kubectl` and `helm` commands in this SOP assume either the default namespace or that the above aliases are in effect.

> ⚠️ **Data loss warning:** `helm uninstall` **deletes the PVC** and all persistent data (steering files, auth database, agent config) unless the chart has an explicit resource policy annotation. Always use `helm rollback` instead of uninstall + reinstall. If you need to uninstall, back up the PVC data first.

---

## Upgrade Process Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Pre-Upgrade Preparation                  │
│  Check version info → Read Release Notes → Announce outage  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                          Backup                              │
│  Helm values / Agent config / Steering / hosts.yml / PVC    │
│  → Verification gate (all files exist & non-empty) ✅        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  Upgrade Execution (2 Phases)                │
│                                                             │
│  Step 1: Pre-release Validation                             │
│    helm upgrade --version x.x.x-beta.1                     │
│    └─ Automated smoke test (kubectl wait + pgrep + logs)    │
│         ├─ Pass ──────────────────────────┐                 │
│         └─ Fail → rollback, stop          │                 │
│                                           ▼                 │
│  Step 2: Promote to Stable                                  │
│    helm upgrade --version x.x.x                            │
│    └─ kubectl rollout status                               │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    Post-Upgrade Verification                  │
│  Pod Ready? → Version check → Log check → Discord E2E test  │
│       │                                                     │
│       ├─ All pass → Send completion notice ✅               │
│       └─ Issues   → Proceed to rollback ↓                  │
└────────────────────────┬────────────────────────────────────┘
                         │ (on failure)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                       Rollback                               │
│                                                             │
│  Diagnose symptom                                           │
│   ├─ Pod won't start    → helm rollback <REVISION>          │
│   ├─ Broken / Pod OK    → rollback or hotfix                │
│   ├─ Config lost        → restore from backup               │
│   └─ Bot unresponsive   → kubectl rollout restart → rollback │
│                                                             │
│  Rollback done → Re-run verification → Send rollback notice │
└─────────────────────────────────────────────────────────────┘
```

---

## I. Pre-Upgrade Preparation

### 1. Resolve Environment Variables

> **Agent note:** Run this block first. All subsequent steps depend on these variables.
>
> **Step 1 output:** `RELEASE_NAME`, `DEPLOYMENT`, `POD`, `CURRENT_VERSION`, `TARGET_VERSION` → used in all subsequent steps.

```bash
export KUBECONFIG=~/.kube/config

# Resolve release name and deployment name
RELEASE_NAME=$(helm list -o json | jq -r '.[0].name')
DEPLOYMENT="${RELEASE_NAME}-kiro"
echo "Release: $RELEASE_NAME  |  Deployment: $DEPLOYMENT"

# Get current running pod (precise label selector — avoids matching other agents)
POD=$(kubectl get pod \
  -l "app.kubernetes.io/instance=${RELEASE_NAME},app.kubernetes.io/name=${DEPLOYMENT}" \
  -o jsonpath='{.items[0].metadata.name}')
echo "Pod: $POD"
if [ -z "$POD" ]; then echo "❌ Pod not found. Check label selectors."; fi

# Get current deployed chart version
CURRENT_VERSION=$(helm list -f "$RELEASE_NAME" -o json | jq -r '.[0].chart' | sed 's/openab-//')
echo "Current chart version: $CURRENT_VERSION"

# List available versions via OCI (no repo add needed)
helm show chart oci://ghcr.io/openabdev/charts/openab 2>/dev/null | grep ^version

# List all published versions (requires GitHub Pages repo to be added)
# helm repo add openab https://openabdev.github.io/openab && helm repo update
# helm search repo openab --versions

# Check the actual image the Pod is running
kubectl get deployment "$DEPLOYMENT" \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

After running the above, set the target version:

```bash
# Set this to the version you are upgrading to (e.g. 0.7.5)
TARGET_VERSION="0.7.5"
echo "Upgrading to: $TARGET_VERSION"
```

### 2. Read the Release Notes

- Go to `https://github.com/openabdev/openab/releases/tag/v${TARGET_VERSION}`
- Pay special attention to:
  - Breaking changes
  - Helm Chart values changes
  - Added or deprecated environment variables
  - Any migration steps

### 3. Check Node Resources

> Skipping this step risks the new Pod getting stuck in `Pending` if the node lacks capacity.

```bash
# Check allocatable resources on all nodes
kubectl describe nodes | grep -A 5 "Allocatable:"

# Check current resource requests across the cluster
kubectl top nodes
```

### 4. Announce the Upgrade

> ⚠️ **Downtime is expected during every upgrade.** The deployment strategy is `Recreate` because the PVC is ReadWriteOnce, which does not support RollingUpdate. The old Pod is terminated before the new one starts — the Discord bot will be unavailable during this window, and this is expected behaviour.

- Notify all users via Discord channel / Slack / email:
  - Scheduled upgrade time and estimated downtime (typically 1–3 minutes)
  - Scope of impact (Discord bot will be offline during the upgrade)
  - Emergency contact

---

## II. Backup

> **Agent note — dependency chain:**
> - Step 0 must run first (resolves `BACKUP_DIR` and `POD`)
> - Steps 1–7 depend on `POD` from Step 0
> - The Verification Gate must pass before proceeding to Section III

### Agent-Executable Backup (Linear Sequence)

This section is written as a machine-executable runbook with no ambiguous branches. Run steps in order.

#### Step 0 — Resolve variables and create backup directory

> **Output:** `BACKUP_DIR`, `POD` → used in Steps 1–7 and the Verification Gate.

```bash
export KUBECONFIG=~/.kube/config

RELEASE_NAME=$(helm list -o json | jq -r '.[0].name')
DEPLOYMENT="${RELEASE_NAME}-kiro"
BACKUP_DIR="openab-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"
echo "Backup directory: $BACKUP_DIR"

POD=$(kubectl get pod \
  -l "app.kubernetes.io/instance=${RELEASE_NAME},app.kubernetes.io/name=${DEPLOYMENT}" \
  -o jsonpath='{.items[0].metadata.name}')
echo "Pod: $POD"

# Gate: abort if pod not found
if [ -z "$POD" ]; then
  echo "❌ Pod not found. Cannot proceed with backup."
  exit 1
fi

# Gate: verify tar is available (required for directory kubectl cp)
if ! kubectl exec "$POD" -- which tar > /dev/null 2>&1; then
  echo "❌ tar not found in container. kubectl cp of directories will fail. Aborting."
  exit 1
fi
```

#### Step 1 — Backup Helm values

> **Output:** `$BACKUP_DIR/values.yaml`

```bash
helm get values "$RELEASE_NAME" -o yaml > "$BACKUP_DIR/values.yaml"
echo "✅ Helm values backed up"
```

#### Step 2 — Backup agent config

> **Input:** `POD` from Step 0 · **Output:** `$BACKUP_DIR/agents/`

```bash
kubectl cp "$POD:/home/agent/.kiro/agents/" "$BACKUP_DIR/agents/"
echo "✅ Agent config backed up"
```

#### Step 3 — Backup steering files

> **Input:** `POD` from Step 0 · **Output:** `$BACKUP_DIR/steering/`

```bash
kubectl cp "$POD:/home/agent/.kiro/steering/" "$BACKUP_DIR/steering/"
echo "✅ Steering files backed up"
```

#### Step 4 — Backup skills (optional)

> **Input:** `POD` from Step 0 · **Output:** `$BACKUP_DIR/skills/` (may be skipped)

```bash
if kubectl exec "$POD" -- test -d /home/agent/.kiro/skills 2>/dev/null; then
  kubectl cp "$POD:/home/agent/.kiro/skills/" "$BACKUP_DIR/skills/"
  echo "✅ Skills directory backed up"
else
  echo "⚠️ skills/ not found in container — skipping (this is normal if no custom skills are installed)"
fi
```

#### Step 5 — Backup GitHub CLI credentials and kiro-cli auth

> **Input:** `POD` from Step 0 · **Output:** `$BACKUP_DIR/hosts.yml`, `$BACKUP_DIR/kiro-auth.sqlite3`

```bash
kubectl cp "$POD:/home/agent/.config/gh/hosts.yml" "$BACKUP_DIR/hosts.yml"
echo "✅ hosts.yml backed up"

# kiro-cli auth database — required for bot to resume without re-authentication
kubectl cp "$POD:/home/agent/.local/share/kiro-cli/data.sqlite3" "$BACKUP_DIR/kiro-auth.sqlite3"
echo "✅ kiro-cli auth DB backed up"
```

#### Step 6 — Backup Kubernetes Secret

> **Output:** `$BACKUP_DIR/secret.yaml` ⚠️ SENSITIVE

```bash
kubectl get secret "${DEPLOYMENT}" -o yaml > "$BACKUP_DIR/secret.yaml"
echo "✅ Secret backed up"
echo "🔐 SECURITY: $BACKUP_DIR/secret.yaml contains credentials. Do NOT commit."
echo "   Encrypt if storing: gpg --symmetric $BACKUP_DIR/secret.yaml"
```

#### Step 7 — Backup Helm release history and PVC data

> **Input:** `POD` from Step 0 · **Output:** `$BACKUP_DIR/helm-history.txt`, `$BACKUP_DIR/pvc-data/`

```bash
helm history "$RELEASE_NAME" > "$BACKUP_DIR/helm-history.txt"
echo "✅ Helm history backed up"

# PVC backup via kubectl cp (default path — /home/agent is the full PVC mount)
# Check size first to avoid timeout
PVC_SIZE=$(kubectl exec "$POD" -- du -sh /home/agent 2>/dev/null | cut -f1)
echo "PVC size: $PVC_SIZE"
# Proceed with kubectl cp (recommended for < 500 MB; use VolumeSnapshot for larger volumes)
kubectl cp "$POD:/home/agent/" "$BACKUP_DIR/pvc-data/"
echo "✅ PVC data backed up"
```

> **Advanced option — VolumeSnapshot (for large PVCs or CSI-enabled clusters):**
> ```bash
> # First, resolve the PVC name
> PVC_NAME=$(kubectl get pod "$POD" \
>   -o jsonpath='{.spec.volumes[?(@.persistentVolumeClaim)].persistentVolumeClaim.claimName}')
> echo "PVC name: $PVC_NAME"
>
> # List available VolumeSnapshotClasses
> SNAPSHOT_CLASS=$(kubectl get volumesnapshotclass -o jsonpath='{.items[0].metadata.name}')
> echo "Snapshot class: $SNAPSHOT_CLASS"
>
> # Create the snapshot
> kubectl apply -f - <<EOF
> apiVersion: snapshot.storage.k8s.io/v1
> kind: VolumeSnapshot
> metadata:
>   name: openab-pvc-snapshot-$(date +%Y%m%d)
> spec:
>   volumeSnapshotClassName: ${SNAPSHOT_CLASS}
>   source:
>     persistentVolumeClaimName: ${PVC_NAME}
> EOF
> ```

#### Verification Gate — must pass before proceeding to upgrade

> **Agent instruction:** Run this gate after all backup steps. If any check fails, **stop and do not proceed** with the upgrade. A failed backup means that data is unprotected.

```bash
echo "=== Backup Verification Gate ==="
GATE_PASS=true

check_file() {
  local path="$1"
  local label="$2"
  if [ -s "$path" ]; then
    echo "  ✅ $label ($path)"
  else
    echo "  ❌ MISSING or EMPTY: $label ($path)"
    GATE_PASS=false
  fi
}

check_dir() {
  local path="$1"
  local label="$2"
  if [ -d "$path" ] && [ -n "$(ls -A "$path" 2>/dev/null)" ]; then
    echo "  ✅ $label ($path)"
  else
    echo "  ❌ MISSING or EMPTY: $label ($path)"
    GATE_PASS=false
  fi
}

check_file "$BACKUP_DIR/values.yaml"           "Helm values"
check_dir  "$BACKUP_DIR/agents/"               "Agent config"
check_dir  "$BACKUP_DIR/steering/"             "Steering files"
check_file "$BACKUP_DIR/hosts.yml"             "GitHub CLI credentials"
check_file "$BACKUP_DIR/kiro-auth.sqlite3"     "kiro-cli auth DB"
check_file "$BACKUP_DIR/secret.yaml"           "Kubernetes Secret"
check_file "$BACKUP_DIR/helm-history.txt"      "Helm history"
check_dir  "$BACKUP_DIR/pvc-data/"             "PVC data"

echo ""
if [ "$GATE_PASS" = true ]; then
  echo "✅ GATE PASSED — all backup files present and non-empty. Safe to proceed with upgrade."
else
  echo "❌ GATE FAILED — one or more backup files are missing or empty."
  echo "   Do NOT proceed with the upgrade until all checks pass."
  exit 1
fi
```

### One-Click Backup Script

The script below combines Steps 0–7 and the Verification Gate into a single file.

```bash
#!/bin/bash
# Note: set -e is intentionally omitted.
# Failures are recorded per step and reported at the end,
# so that a single failure does not prevent remaining items from being backed up.

export KUBECONFIG=~/.kube/config

RELEASE_NAME=$(helm list -o json | jq -r '.[0].name')
DEPLOYMENT="${RELEASE_NAME}-kiro"
BACKUP_DIR="openab-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

POD=$(kubectl get pod \
  -l "app.kubernetes.io/instance=${RELEASE_NAME},app.kubernetes.io/name=${DEPLOYMENT}" \
  -o jsonpath='{.items[0].metadata.name}')
if [ -z "$POD" ]; then
  echo "❌ OpenAB Pod not found. Aborting backup." && exit 1
fi

# Pre-check: kubectl cp directory operations require tar inside the container
if ! kubectl exec "$POD" -- which tar > /dev/null 2>&1; then
  echo "❌ tar not found in container. kubectl cp of directories will fail."
  echo "   Use VolumeSnapshot for PVC backup instead."
  exit 1
fi

FAILED_STEPS=()

backup_step() {
  local desc="$1"; shift
  echo "=== $desc ==="
  if ! "$@"; then
    echo "⚠️  Failed: $desc (continuing with remaining steps...)"
    FAILED_STEPS+=("$desc")
  fi
}

backup_step "Backup Helm values"       bash -c "helm get values '$RELEASE_NAME' -o yaml > $BACKUP_DIR/values.yaml"
backup_step "Backup Agent config"      kubectl cp "$POD:/home/agent/.kiro/agents/" "$BACKUP_DIR/agents/"
backup_step "Backup Steering files"    kubectl cp "$POD:/home/agent/.kiro/steering/" "$BACKUP_DIR/steering/"
backup_step "Backup hosts.yml"         kubectl cp "$POD:/home/agent/.config/gh/hosts.yml" "$BACKUP_DIR/hosts.yml"
backup_step "Backup kiro-cli auth DB"  kubectl cp "$POD:/home/agent/.local/share/kiro-cli/data.sqlite3" "$BACKUP_DIR/kiro-auth.sqlite3"
backup_step "Backup full Secret"       bash -c "kubectl get secret '${DEPLOYMENT}' -o yaml > $BACKUP_DIR/secret.yaml"
backup_step "Backup Helm history"      bash -c "helm history '$RELEASE_NAME' > $BACKUP_DIR/helm-history.txt"
backup_step "Backup PVC data"          kubectl cp "$POD:/home/agent/" "$BACKUP_DIR/pvc-data/"

if kubectl exec "$POD" -- test -d /home/agent/.kiro/skills 2>/dev/null; then
  backup_step "Backup skills" kubectl cp "$POD:/home/agent/.kiro/skills/" "$BACKUP_DIR/skills/"
else
  echo "⚠️ skills/ not found — skipping (normal if no custom skills installed)"
fi

echo ""
echo "=== Backup Summary: $BACKUP_DIR ==="
ls -la "$BACKUP_DIR/"

if [ ${#FAILED_STEPS[@]} -gt 0 ]; then
  echo ""
  echo "⚠️  The following backup steps FAILED:"
  for step in "${FAILED_STEPS[@]}"; do
    echo "   - $step"
  done
  echo ""
  echo "   Review the failures above before proceeding with the upgrade."
  echo "   A failed backup step means the corresponding data is NOT protected."
else
  echo "✅ All backup steps completed successfully."
fi

echo ""
echo "🔐 SECURITY REMINDER: $BACKUP_DIR/secret.yaml contains sensitive credentials"
echo "   (Discord token, STT key, etc.). Do NOT commit this file."
echo "   Consider encrypting it before storing:"
echo "     gpg --symmetric $BACKUP_DIR/secret.yaml"
echo "     # or: age -p -o $BACKUP_DIR/secret.yaml.age $BACKUP_DIR/secret.yaml"
```

### Backup Checklist (Human Reference)

| Item | Command | Notes |
|---|---|---|
| Helm values | `helm get values $RELEASE_NAME -o yaml > $BACKUP_DIR/values.yaml` | Current Helm deployment parameters |
| Agent config | `kubectl cp $POD:/home/agent/.kiro/agents/ $BACKUP_DIR/agents/` | Custom agent settings (model, prompt, tools, etc.) |
| Steering files | `kubectl cp $POD:/home/agent/.kiro/steering/ $BACKUP_DIR/steering/` | Steering docs such as IDENTITY.md |
| Skills | `kubectl cp $POD:/home/agent/.kiro/skills/ $BACKUP_DIR/skills/` | Custom agent skills (if any; see Step 4 for conditional check) |
| hosts.yml | `kubectl cp $POD:/home/agent/.config/gh/hosts.yml $BACKUP_DIR/hosts.yml` | GitHub CLI credentials (including multi-account configs) |
| kiro-cli auth | `kubectl cp $POD:/home/agent/.local/share/kiro-cli/data.sqlite3 $BACKUP_DIR/kiro-auth.sqlite3` | Bot auth DB — required to avoid re-authentication after PVC loss |
| Full Secret export | `kubectl get secret ${DEPLOYMENT} -o yaml > $BACKUP_DIR/secret.yaml` | ⚠️ **SENSITIVE** — contains Discord token, STT key, and other credentials. Store securely, **never commit to version control**. |
| PVC data | `kubectl cp $POD:/home/agent/ $BACKUP_DIR/pvc-data/` | Default: kubectl cp. See Step 7 for VolumeSnapshot (advanced). |
| Helm release history | `helm history $RELEASE_NAME > $BACKUP_DIR/helm-history.txt` | Useful reference for rollback |

---

## III. Upgrade Execution

> **Agent note — dependency chain:**
> - Requires `RELEASE_NAME`, `DEPLOYMENT`, `BACKUP_DIR`, `TARGET_VERSION` from Section I.
> - Requires the Verification Gate (Section II) to have passed.

### Pre-check: Resolve Upgrade Variables

```bash
export KUBECONFIG=~/.kube/config

# Resolve release name and deployment
RELEASE_NAME=$(helm list -o json | jq -r '.[0].name')
DEPLOYMENT="${RELEASE_NAME}-kiro"

# Resolve backup directory (most recent backup)
BACKUP_DIR=$(ls -td openab-backup-* 2>/dev/null | head -1)
BACKUP_VALUES="${BACKUP_DIR}/values.yaml"
echo "Using backup: $BACKUP_DIR"
echo "Values file: $BACKUP_VALUES"

# Confirm the values file exists and is non-empty
if [ ! -s "$BACKUP_VALUES" ]; then
  echo "❌ values.yaml not found or empty at $BACKUP_VALUES. Run backup first."
  exit 1
fi

# Set target version (e.g. 0.7.5 — check https://github.com/openabdev/openab/releases)
TARGET_VERSION="0.7.5"

# List available chart versions via OCI (no helm repo add required)
helm show chart oci://ghcr.io/openabdev/charts/openab --version "$TARGET_VERSION" 2>/dev/null \
  | grep -E "^(name|version|appVersion):"
```

### Pre-check: Confirm Helm OCI Access

```bash
# Verify OCI registry is reachable and the target version exists
helm show chart oci://ghcr.io/openabdev/charts/openab --version "${TARGET_VERSION}" > /dev/null \
  && echo "✅ Chart version ${TARGET_VERSION} available via OCI" \
  || echo "❌ Chart version ${TARGET_VERSION} not found. Check version string."

# Also verify the pre-release beta.1 version exists (required for Step 1)
helm show chart oci://ghcr.io/openabdev/charts/openab --version "${TARGET_VERSION}-beta.1" > /dev/null \
  && echo "✅ Pre-release ${TARGET_VERSION}-beta.1 available" \
  || echo "⚠️ beta.1 not found — check GitHub releases for available pre-release tags"
```

### Step 1: Pre-release Validation (Required)

> ⚠️ Per project convention, **a stable release must be preceded by a validated pre-release**. Do not skip directly to Step 2.
>
> **When can Step 1 be skipped?** Only if the release notes for the target stable version explicitly contain `pre-release-validated: true`, indicating that the corresponding pre-release has already been validated in another environment (e.g. a staging cluster). In all other cases, run Step 1 first.
>
> **Agent note — pass/fail criteria:**
> - **Pass:** `kubectl wait` exits 0 AND `pgrep -x openab` exits 0 AND log scan returns no `panic` or `fatal` lines.
> - **Fail:** Any of the above fails, OR a human operator reports a functional regression in Discord within the monitoring window. On failure, run `helm rollback` (see Section IV) and stop — do not proceed to Step 2.

```bash
# Dry-run first to catch values conflicts
helm upgrade "$RELEASE_NAME" oci://ghcr.io/openabdev/charts/openab \
  --version "${TARGET_VERSION}-beta.1" \
  -f "$BACKUP_VALUES" \
  --dry-run

# Deploy the pre-release
helm upgrade "$RELEASE_NAME" oci://ghcr.io/openabdev/charts/openab \
  --version "${TARGET_VERSION}-beta.1" \
  -f "$BACKUP_VALUES"

kubectl rollout status "deployment/${DEPLOYMENT}" --timeout=300s

# Automated smoke test
POD=$(kubectl get pod \
  -l "app.kubernetes.io/instance=${RELEASE_NAME},app.kubernetes.io/name=${DEPLOYMENT}" \
  -o jsonpath='{.items[0].metadata.name}')
kubectl wait --for=condition=Ready "pod/${POD}" --timeout=120s
kubectl exec "$POD" -- pgrep -x openab
PANIC_LINES=$(kubectl logs "deployment/${DEPLOYMENT}" --tail=100 | grep -icE "panic|fatal" || true)
if [ "$PANIC_LINES" -gt 0 ]; then
  echo "❌ Panic/fatal lines found in logs. Do NOT proceed. Run rollback."
  exit 1
fi
echo "✅ Automated smoke test passed. Proceed with Discord functional validation."

# After automated smoke test — manual Discord check required:
# → Send a test message in the Discord channel
# → Confirm the bot responds and basic conversation / tool calls work
# → If bot is unresponsive or broken: run helm rollback (Section IV) and stop
```

### Step 2: Promote to Stable

> **Agent note:** Only run this after Step 1 has passed both automated and Discord validation.

```bash
# Dry-run the stable version
helm upgrade "$RELEASE_NAME" oci://ghcr.io/openabdev/charts/openab \
  --version "${TARGET_VERSION}" \
  -f "$BACKUP_VALUES" \
  --dry-run

# Deploy stable (short downtime is expected due to Recreate strategy)
helm upgrade "$RELEASE_NAME" oci://ghcr.io/openabdev/charts/openab \
  --version "${TARGET_VERSION}" \
  -f "$BACKUP_VALUES"

# Wait for the Pod to be ready
kubectl rollout status "deployment/${DEPLOYMENT}" --timeout=300s
```

### Post-Upgrade Verification

> **Agent note — pass/fail criteria:**
> - **Pass:** All commands below exit 0 AND image tag matches `TARGET_VERSION` AND no panic/fatal in logs.
> - **Fail:** Any command exits non-zero, or image tag does not match. → Proceed to Section IV Rollback.

```bash
POD=$(kubectl get pod \
  -l "app.kubernetes.io/instance=${RELEASE_NAME},app.kubernetes.io/name=${DEPLOYMENT}" \
  -o jsonpath='{.items[0].metadata.name}')

# 1. Check Pod status (must be Running and READY)
kubectl get pod -l "app.kubernetes.io/instance=${RELEASE_NAME},app.kubernetes.io/name=${DEPLOYMENT}"
kubectl wait --for=condition=Ready "pod/${POD}" --timeout=120s

# 2. Verify deployed chart version matches target
DEPLOYED=$(helm list -f "$RELEASE_NAME" -o json | jq -r '.[0].chart')
echo "Deployed chart: $DEPLOYED  |  Expected: openab-${TARGET_VERSION}"
if [ "$DEPLOYED" != "openab-${TARGET_VERSION}" ]; then
  echo "❌ Version mismatch. Investigate before proceeding."
fi

# 3. Verify image tag
kubectl get "deployment/${DEPLOYMENT}" \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

# 4. Confirm the openab process is running
kubectl exec "$POD" -- pgrep -x openab

# 5. Check logs for errors
PANIC_LINES=$(kubectl logs "deployment/${DEPLOYMENT}" --tail=100 | grep -icE "panic|fatal" || true)
ERROR_LINES=$(kubectl logs "deployment/${DEPLOYMENT}" --tail=100 | grep -icE "error|warn" || true)
echo "Panic/fatal lines: $PANIC_LINES  |  Error/warn lines: $ERROR_LINES"
if [ "$PANIC_LINES" -gt 0 ]; then
  echo "❌ Panic/fatal found. Rollback recommended."
fi

# 6. Verify PVC data (steering files and agent config) are still present
kubectl exec "$POD" -- ls /home/agent/.kiro/steering/
kubectl exec "$POD" -- cat /home/agent/.kiro/agents/default.json | head -5
# If either path is missing, restore from backup (see Section IV: Restore Custom Config)

# 7. Discord E2E verification (final check — requires human operator)
# → Send a test message in the Discord channel
# → Confirm the bot responds and conversation works correctly
```

### Completion Notice

- Once all verifications pass, notify users:
  - Upgrade complete, service restored
  - New version number and summary of key changes
  - Contact channel for reporting any issues

---

## IV. Rollback

### Decision Tree

> **Agent note — machine-readable branch criteria:**
>
> | Observed condition | Action |
> |---|---|
> | `kubectl get pod` shows `CrashLoopBackOff` or `Pending` | `helm rollback` immediately |
> | Pod is `Running` AND `pgrep -x openab` exits non-zero | `helm rollback` |
> | Pod is `Running`, process OK, but logs contain `panic` or `fatal` | `helm rollback` |
> | Pod is `Running`, process OK, logs clean, but no Discord response after 60s | `kubectl rollout restart` first; if still no response after 60s → `helm rollback` |
> | Pod is `Running`, process OK, logs clean, Discord responds, but config is missing | Restore config from backup → `kubectl rollout restart` |
> | Quick fix is clearly identified (e.g. a known bad config key) | Hotfix — escalate to human engineer |

```
Issue detected after upgrade
         │
         ▼
    Pod status?
         │
         ├─ CrashLoopBackOff / Pending ──→ helm rollback <REVISION>
         │
         ├─ Running, pgrep exits non-zero OR panic in logs
         │         └─ helm rollback <REVISION>
         │
         ├─ Running, logs clean, bot unresponsive
         │         └─ kubectl rollout restart deployment/${DEPLOYMENT}
         │                   │
         │                   ├─ Responds within 60s ──→ Continue monitoring
         │                   └─ Still unresponsive   ──→ helm rollback <REVISION>
         │
         └─ Running, bot OK, config missing
                   └─ Restore config from backup → kubectl rollout restart
```

| Symptom | Action |
|---|---|
| Pod fails to start (CrashLoopBackOff) | Helm rollback |
| Functionality broken, Pod is running | Rollback or hotfix  |
| Custom config lost | Restore config files from backup |
| Bot unresponsive | Restart Pod first; rollback if it persists |

### Helm Rollback

```bash
export KUBECONFIG=~/.kube/config
RELEASE_NAME=$(helm list -o json | jq -r '.[0].name')
DEPLOYMENT="${RELEASE_NAME}-kiro"

# 1. View release history and identify the previous revision
helm history "$RELEASE_NAME"

# 2. Get the previous revision number automatically
PREV_REVISION=$(helm history "$RELEASE_NAME" --output json \
  | jq -r 'sort_by(.revision) | .[-2].revision')
echo "Rolling back to revision: $PREV_REVISION"

# 3. Roll back
helm rollback "$RELEASE_NAME" "$PREV_REVISION"

# 4. Wait for the Pod to be ready
kubectl rollout status "deployment/${DEPLOYMENT}" --timeout=300s

# 5. Confirm rollback succeeded
kubectl get pod -l "app.kubernetes.io/instance=${RELEASE_NAME},app.kubernetes.io/name=${DEPLOYMENT}"

# 6. Run full post-rollback verification
POD=$(kubectl get pod \
  -l "app.kubernetes.io/instance=${RELEASE_NAME},app.kubernetes.io/name=${DEPLOYMENT}" \
  -o jsonpath='{.items[0].metadata.name}')
kubectl wait --for=condition=Ready "pod/${POD}" --timeout=120s
kubectl exec "$POD" -- pgrep -x openab
kubectl logs "deployment/${DEPLOYMENT}" --tail=100 | grep -iE "error|warn|panic|fatal"
# → Send a test message in the Discord channel to confirm the bot responds
```

### Restore Custom Config

```bash
export KUBECONFIG=~/.kube/config
RELEASE_NAME=$(helm list -o json | jq -r '.[0].name')
DEPLOYMENT="${RELEASE_NAME}-kiro"

POD=$(kubectl get pod \
  -l "app.kubernetes.io/instance=${RELEASE_NAME},app.kubernetes.io/name=${DEPLOYMENT}" \
  -o jsonpath='{.items[0].metadata.name}')

# Resolve backup directory
BACKUP_DIR=$(ls -td openab-backup-* 2>/dev/null | head -1)
echo "Restoring from: $BACKUP_DIR"

# Restore agent config
kubectl cp "$BACKUP_DIR/agents/default.json" "$POD:/home/agent/.kiro/agents/default.json"

# Restore steering files
# ⚠️ kubectl cp directory behaviour varies across versions — trailing slash matters.
# Use the tar pipe method below to avoid accidentally creating a nested directory
# (e.g. steering/steering/) which can happen with some kubectl versions.
kubectl exec "$POD" -- mkdir -p /home/agent/.kiro/steering
tar c -C "$BACKUP_DIR/steering" . | kubectl exec -i "$POD" -- tar x -C /home/agent/.kiro/steering

# Restore GitHub CLI credentials
kubectl cp "$BACKUP_DIR/hosts.yml" "$POD:/home/agent/.config/gh/hosts.yml"

# Restore kiro-cli auth database
kubectl exec "$POD" -- mkdir -p /home/agent/.local/share/kiro-cli
kubectl cp "$BACKUP_DIR/kiro-auth.sqlite3" "$POD:/home/agent/.local/share/kiro-cli/data.sqlite3"

# Restart Pod to apply changes
kubectl rollout restart "deployment/${DEPLOYMENT}"
```
