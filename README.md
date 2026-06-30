# acmgitops

A **reusable GitOps repo** that, on any OpenShift cluster, stands up the management
plane for **Trilio for Kubernetes (T4K)** data protection driven from **Red Hat
Advanced Cluster Management (ACM)**:

1. **OpenShift GitOps** (Argo CD) — the engine that syncs everything in this repo.
2. **OpenShift Pipelines** (Tekton) — CI to validate/roll out changes.
3. **ACM** — installed and brought up as the multi-cluster hub.
4. **T4K policies** — ACM Governance Policies that install T4K and drive namespace
   backups across every managed cluster you label, all from the hub.

Clone it, point it at your repo, fill in your values, apply one manifest. dc11 ships as
a worked example under `values/dc11/`.

> This repo is **only** GitOps + ACM + T4K. It deliberately does NOT carry a cluster's
> full platform operator set (CNV, RHOAI, logging, ...) -- that belongs to each cluster's
> own config.

## Layout
```
bootstrap/
  01-openshift-gitops-operator.yaml   # one-time: install Argo CD
  app-of-apps.yaml                    # the single manifest you apply afterwards
apps/                                 # child Argo Applications, by sync-wave
  pipelines      (wave 10)
  acm            (wave 20)
  t4k-policies   (wave 30)
components/
  gitops/        # Argo instance RBAC (cluster-admin binding) -- applied once, by hand
  pipelines/     # Tekton operator + a sample validation pipeline
  acm/           # ACM: namespace + OperatorGroup + Subscription + MultiClusterHub
  t4k/           # Trilio governance (adapted to 5.3.x): configmaps / placement / policies
values/
  dc11/          # worked example values (NFS, backup namespace) -- reference
  template-*.example  # blank templates you copy and fill
```

## Replicate it on your cluster
1. **Fork / create your repo** and push this tree.
2. **Set the repo URL** everywhere (placeholder `__REPO_URL__`):
   ```
   grep -rl __REPO_URL__ . | xargs sed -i 's#__REPO_URL__#https://github.com/<you>/acmgitops.git#g'
   ```
3. **Install Argo CD and grant it permissions** (one-time):
   ```
   oc apply -f bootstrap/01-openshift-gitops-operator.yaml
   oc get csv -n openshift-operators | grep gitops          # wait: Succeeded
   oc apply -f components/gitops/argocd-cluster-admin.yaml
   ```
4. **Provide your values on the hub** (the T4K policies read these via ACM hub templating):
   ```
   cp values/template-trilio-backup-configmap.yaml.example  /tmp/backup-cm.yaml   # edit NFS + namespace
   cp values/template-trilio-license-configmap.yaml.example /tmp/license-cm.yaml  # edit real license key
   oc apply -n default -f /tmp/backup-cm.yaml -f /tmp/license-cm.yaml
   ```
   (Or copy `values/dc11/` and adapt.)
5. **Apply the app-of-apps** -- Argo does the rest:
   ```
   oc apply -f bootstrap/app-of-apps.yaml
   ```
   Sync order: pipelines -> ACM operator + MultiClusterHub -> T4K policies. Argo retries
   while the ACM CSV/MultiClusterHub come up, then the policy propagator distributes the
   T4K install + backup policies.
6. **Opt a managed cluster into protection** (label-based placement):
   ```
   oc label managedcluster <name> protected-by=trilio vendor=OpenShift
   ```
7. **Opt namespaces into backup** on each managed cluster (label-driven discovery):
   ```
   oc label namespace <ns> trilio-backup=true
   ```
   The backup policy picks up every namespace carrying this label automatically — no
   per-namespace edits in Git. Label a new namespace and it gets protected on next sync.

## What the T4K policies do (`components/t4k/`)
- **install-trilio** -- namespace, T4K operator Subscription (**channel 5.3.x**,
  `startingCSV k8s-triliovault-stable.5.3.1`), `TrilioVaultManager`, and `License`
  (from your hub ConfigMap).
- **create-trilio-ns-backup** -- **label-driven, multi-namespace**. Discovers every
  namespace labelled `trilio-backup=true` (via `lookup`+`range` templating) and, in each,
  enforces a NFS `Target`, a Retention `Policy` (`retainLatest`), a Schedule `Policy`
  (`backupSchedule` cron), and a `BackupPlan` wiring them together. It creates **no**
  standalone `Backup` objects: T4K's scheduler fires backups on the cron and prunes to
  `retainLatest`, so ACM never fights T4K over individual backups. All tunables come from
  the hub ConfigMap `trilio-backup-configmap`.

CRD apiVersions verified against the k8s-triliovault source: `TrilioVaultManager`,
`License`, `Target`, `BackupPlan`, `Policy`, `Backup` all served at
`triliovault.trilio.io/v1` in 5.3.x.

## Verify before relying on it
- Operator **channels** (`latest` for GitOps/Pipelines, `release-2.16` for ACM) -- confirm
  for your OCP version; right after a GA some operators lag.
- `cluster-admin` for the Argo controller is broad -- scope down in hardened environments.
- The sample pipeline is illustrative (validates manifests); adapt it to your CI needs.
