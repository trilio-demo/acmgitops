# acmgitops

A **reusable GitOps repo** that, on any OpenShift cluster, stands up the management
plane for **Trilio for Kubernetes (T4K)** data protection, driven from **Red Hat
Advanced Cluster Management (ACM)**.

Clone it, point it at your Git repo, drop in a few values, apply **one** manifest.
From there Argo CD installs OpenShift Pipelines, ACM, and a set of ACM Governance
Policies that install T4K and run namespace backups across every managed cluster you
label — all from the hub. `dc11` ships as a worked example (`values/dc11/`).

---

## 1. How the pieces fit

Three layers, each installed by the one below it:

| Layer | What it is | Role here |
|-------|-----------|-----------|
| **OpenShift GitOps** (Argo CD) | Continuous delivery engine | Syncs everything in this repo. You apply one root Application; it renders all the rest. |
| **OpenShift Pipelines** (Tekton) | CI | A sample pipeline that validates the manifests. Optional but wired in. |
| **ACM** (Advanced Cluster Management) | Multi-cluster hub | Governs managed clusters via **Policies** + **Placement**. The T4K install and backups are ACM Policies, not raw manifests — so the hub keeps every managed cluster in the desired state. |
| **T4K** | The data-protection product | Installed on each managed cluster by an ACM Policy (OLM Subscription → operator → `TrilioVaultManager` → `License`). Backups are governed by a second Policy. |

### The flow

```
 you ──apply──> bootstrap/app-of-apps.yaml        (the ONE manifest, on the hub)
                        │  Argo CD "app of apps"
                        ▼
        ┌───────────────┼───────────────────────────┐
   apps/pipelines   apps/acm                    apps/t4k-policies
     (wave 10)      (wave 20)                      (wave 30)
   Tekton operator  ACM operator + MultiClusterHub  ACM Policies + Placement
                        │                               │
                        │ ACM hub comes up              │ policy propagator pushes
                        ▼                               ▼   to labelled clusters
                  managed clusters  ◄────────────  install-trilio  +  create-trilio-ns-backup
                  (label to opt in)                     │
                                                        ▼
                                         T4K operator → TVM → License (Active)
                                         + per-namespace Target/Retention/Schedule/BackupPlan
```

### Two labels drive everything

| Label | Put it on | Effect |
|-------|-----------|--------|
| `protected-by=trilio` (+ `vendor=OpenShift`) | a **ManagedCluster** | The `install-trilio` Placement selects it → T4K gets installed there. |
| `trilio-backup=true` | a **Namespace** (on a managed cluster) | The backup Policy discovers it (`lookup`+`range`) → creates Target + Retention + Schedule + BackupPlan **in that namespace**. |

Label a new cluster or namespace and it's protected on the next sync — no per-object edits in Git.

---

## 2. Repo layout

```
bootstrap/
  01-openshift-gitops-operator.yaml   # one-time: install Argo CD
  app-of-apps.yaml                    # the single manifest you apply afterwards
apps/                                 # child Argo Applications, by sync-wave
  pipelines.yaml     (wave 10)
  acm.yaml           (wave 20)
  t4k-policies.yaml  (wave 30)        # note: directory recurse=true (nested dirs)
components/
  gitops/     # Argo instance RBAC (cluster-admin binding) — applied once, by hand
  pipelines/  # Tekton operator + a sample validation pipeline
  acm/        # ACM: namespace + OperatorGroup + Subscription + MultiClusterHub
  t4k/
    placement/  # PlacementRule + PlacementBinding for each policy
    policies/   # install-trilio  +  create-trilio-ns-backup
    configmaps/ # *.example only (real values live under values/, gitignored)
values/
  ocp-4.20/   # OCP profile: certified operator via OperatorHub
  ocp-4.22/   # OCP profile: custom grpc catalog (no certified bundle yet)
  dc11/       # environment values (NFS export, schedule, retention)
  template-*.example
```

---

## 3. Prerequisites

On the **hub** cluster:

- `cluster-admin` (`oc whoami` = an admin, e.g. `kube:admin`).
- **OpenShift GitOps** operator (installed by `bootstrap/01-...`, below).
- Your Git repo **reachable by Argo CD**: either public, or a repository credential
  added to `openshift-gitops`. (Argo runs in-cluster and has no laptop creds.)

On each **managed** cluster that will run backups:

- A **CSI driver with snapshot support** and a `VolumeSnapshotClass`
  (`oc get volumesnapshotclass`). T4K uses CSI snapshots; without one it can't
  snapshot PVCs.
- A reachable **NFS export** (or S3 bucket) for backup storage.
- A valid **T4K license key**.

---

## 4. Sync-wave flow (what "wave" means here)

`bootstrap/app-of-apps.yaml` is a root Argo Application pointing at `apps/`. Each child
Application carries `argocd.argoproj.io/sync-wave`: pipelines `10` → acm `20` →
t4k-policies `30`. The waves order **creation** of the children; once created they sync
**independently and in parallel**, with retry. So it's normal to see `t4k-policies`
report healthy before ACM has finished — the policies just sit inert until ACM is up and
a cluster is labelled.

Two child components install an operator **and** a CR that the operator's CRD defines
(ACM: Subscription + `MultiClusterHub`; Pipelines: Subscription + a Tekton `Pipeline`).
Argo dry-runs every object up front, and the CR fails that dry-run because its CRD does
not exist yet. Both CRs therefore carry:

```yaml
annotations:
  argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
```

so wave-0 installs the operator, OLM lands the CRD, and the CR applies on retry.

---

## 5. Pick your OCP profile (this is the 4.20 vs 4.22 decision)

Where the T4K **operator** comes from depends on your OpenShift version, because Red
Hat's `certified-operators` catalog is **per-OCP-version**:

| | **OCP 4.20** (and any version with a certified bundle) | **OCP 4.22** (newer than the certified bundle) |
|---|---|---|
| Operator source | `certified-operators` (OperatorHub) | Trilio's own **grpc CatalogSource** (`quay.io/triliovault/k8s-triliovault-catalog`, RC builds) |
| Custom CatalogSource created? | **No** | **Yes** (by the policy) |
| Values file | `values/ocp-4.20/trilio-operator-configmap.yaml` | `values/ocp-4.22/trilio-operator-configmap.yaml` |
| Version you get | latest **GA** for 4.20 — the **5.2.x** line per Trilio's [compatibility matrix](https://docs.trilio.io/kubernetes/overview/compatibility-matrix) | latest **RC** on the grpc index (4.22 is not in the matrix yet) |

The `install-trilio` policy is **parameterized** — it reads `catalogSource`,
`catalogSourceImage`, `channel`, and `startingCSV` from a hub ConfigMap
`trilio-operator-configmap` (via ACM hub templating). The CatalogSource is created
**only when `catalogSourceImage` is non-empty**, so the certified/OperatorHub path
creates no extra catalog.

**Always confirm the channel + CSV against the target cluster's catalog** — the CSV name
is authoritative from the cluster, not from docs:

```
oc get packagemanifest k8s-triliovault -n openshift-marketplace \
  -o jsonpath='{.status.defaultChannel}{"\n"}{range .status.channels[*]}{.name}={.currentCSV}{"\n"}{end}'
```

Put `defaultChannel` into `channel` and the matching `currentCSV` into `startingCSV` in
your profile's `trilio-operator-configmap.yaml`. (If the command returns nothing on a new
OCP like 4.22, the certified bundle isn't published for that version yet — use the custom
catalog profile.)

---

## 6. Install, step by step

**1. Point the repo at your Git URL** (placeholder `__REPO_URL__`, if forking):
```
grep -rl __REPO_URL__ . | xargs sed -i 's#__REPO_URL__#https://github.com/<you>/acmgitops.git#g'
```

**2. Install Argo CD and grant it permissions** (one-time):
```
oc apply -f bootstrap/01-openshift-gitops-operator.yaml
oc get csv -A | grep gitops        # wait: Succeeded
oc apply -f components/gitops/argocd-cluster-admin.yaml
```

**3. Put the three hub ConfigMaps in `default`** (the policies read these via hub templating):
```
# operator source — pick your OCP profile
oc apply -f values/ocp-4.20/trilio-operator-configmap.yaml     # or values/ocp-4.22/...

# backup tunables (NFS export, schedule, retention) — copy from dc11 or the template
oc apply -f values/dc11/trilio-backup-configmap.yaml

# license (real key — gitignored; copy the .example and fill it)
cp values/template-trilio-license-configmap.yaml.example /tmp/license-cm.yaml   # edit key
oc apply -n default -f /tmp/license-cm.yaml
```

**4. Make the repo readable by Argo** — public repo, or add a repository credential to
`openshift-gitops`. (Symptom if you skip this: the root app shows
`ComparisonError … authentication required: Repository not found`.)

**5. Apply the one manifest:**
```
oc apply -f bootstrap/app-of-apps.yaml
```
Watch it:
```
oc get applications -n openshift-gitops
oc get multiclusterhub -A          # wait: Running (ACM is the long pole, several minutes)
```

**6. Opt a managed cluster into protection:**
```
oc label managedcluster <name> protected-by=trilio vendor=OpenShift
```
On a single-cluster hub, ACM self-imports it as `local-cluster` — label that. T4K then
installs; verify:
```
oc get csv -n trilio-system                              # k8s-triliovault…: Succeeded
oc get triliovaultmanager -n trilio-system               # STATUS: Deployed
oc get license -n trilio-system                          # STATUS: Active
oc get policy install-trilio -n default                  # COMPLIANCE: Compliant
```

**7. Opt namespaces into backup** (on the managed cluster):
```
oc label namespace <ns> trilio-backup=true
oc get target,policy.triliovault.trilio.io,backupplan -n <ns>   # Target Available, BackupPlan Available
```

**8. (Optional) prove it** — the schedule is periodic (e.g. weekly), so trigger one now:
```
# Backup
oc apply -n <ns> -f - <<'EOF'
apiVersion: triliovault.trilio.io/v1
kind: Backup
metadata: { name: smoke-backup }
spec:
  type: Full
  backupPlan: { name: <ns>-backupplan, namespace: <ns> }
EOF
oc get backup smoke-backup -n <ns> -o jsonpath='{.status.status} {.status.percentageCompletion}{"\n"}'

# Restore into a fresh namespace
oc create namespace <ns>-restore
oc apply -n <ns>-restore -f - <<'EOF'
apiVersion: triliovault.trilio.io/v1
kind: Restore
metadata: { name: smoke-restore }
spec:
  source: { type: Backup, backup: { name: smoke-backup, namespace: <ns> } }
  cleanupConfig: { enabled: false }
  restoreFlags: { skipIfAlreadyExists: true, useOCPNamespaceUIDRange: true }
EOF
```
The Restore restores into the namespace where the CR lives.

---

## 7. What the T4K policies do (`components/t4k/`)

- **install-trilio** — on each labelled cluster, enforces: the `trilio-system`
  namespace, an **OperatorGroup** (required, or OLM never builds an InstallPlan),
  optionally a **CatalogSource** (only when a custom catalog image is set), the T4K
  operator **Subscription** (source/channel/CSV from `trilio-operator-configmap`), the
  **TrilioVaultManager**, and the **License** (key from the `trilio-license` ConfigMap).
- **create-trilio-ns-backup** — **label-driven, multi-namespace**. Discovers every
  namespace labelled `trilio-backup=true` and, in each, enforces an NFS `Target`, a
  Retention `Policy` (`retainLatest`), a Schedule `Policy` (`backupSchedule` cron), and a
  `BackupPlan` wiring them. It creates **no** standalone `Backup` objects — T4K's own
  scheduler fires and prunes, so ACM never fights T4K over individual backups. Tunables
  come from `trilio-backup-configmap`.

---

## 8. Troubleshooting

Issues you may hit in your own environment (these depend on your cluster/network/repo,
not on this repo's manifests):

| Symptom | Cause | What to do |
|---|---|---|
| Root app: `ComparisonError … authentication required: Repository not found` | Repo is private; Argo runs in-cluster with no creds | Make the repo public, or add a repository credential to `openshift-gitops` |
| Subscription: `ResolutionFailed … no operators found in package k8s-triliovault` | The chosen catalog has no T4K bundle for this OCP (e.g. `certified-operators` on 4.22) | Use the right OCP profile (`values/ocp-4.22` for the custom catalog), and confirm `channel`/`startingCSV` from `oc get packagemanifest k8s-triliovault -n openshift-marketplace` |
| CatalogSource pod `ImagePullBackOff … manifest unknown` | Wrong image **tag** (the catalog index versions differently from the app) | Pick a real tag from `quay.io/triliovault/k8s-triliovault-catalog` and set `catalogSourceImage` in `trilio-operator-configmap` |
| `Target` stuck `InProgress` | NFS export unreachable/unwritable from the cluster | Check the export in `trilio-backup-configmap`; watch the `tvk-target-validation-*` pod in `trilio-system` |

## 9. Design decisions (why the manifests look the way they do)

These are already handled in the repo — noted so the choices aren't a mystery, and so
you don't "simplify" them back into breakage:

- **`apps/t4k-policies.yaml` sets `directory: { recurse: true }`.** `components/t4k` is
  nested (placement/, policies/); without recurse Argo reads zero manifests from the flat
  root and falsely reports Synced/Healthy while deploying nothing.
- **`install-trilio` creates an OperatorGroup in `trilio-system`.** Without one, OLM never
  generates an InstallPlan and the Subscription hangs silently.
- **`MultiClusterHub` and the sample `Pipeline` carry `SkipDryRunOnMissingResource=true`.**
  Their CRDs don't exist until the operator installs (same sync), so Argo's up-front
  dry-run would otherwise reject them.
- **The `TrilioVaultManager` does not set `logLevel`.** Recent T4K webhooks reject the
  deprecated `logLevel`/`datamoverLogLevel` fields (use `logConfig` if you must tune logs).

## 10. Notes / verify before relying on it

- **Operator channels** (`latest` for GitOps/Pipelines, `release-2.16` for ACM) — confirm
  for your OCP version; right after a GA some operators lag.
- The 4.22 custom-catalog profile pins an **RC** build (no GA bundle for 4.22 yet).
  Switch that profile back to `certified-operators` once a 4.22-certified GA ships.
- `cluster-admin` for the Argo controller is broad — scope it down in hardened environments.
- Official Trilio references: OCP install and ACM-policy deployment guides on
  `docs.trilio.io/kubernetes`.
