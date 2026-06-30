# dc11 — worked example values

Concrete values used for the dc11 lab. Use these as a reference for your own cluster.

- `trilio-backup-configmap.yaml` — dc11's backup target: namespace `acm-enterprise-demo`,
  NFS `172.22.5.250:/export/sa-lab-nfs-share1` (SHARED lab NFS — do not wipe), threshold 300Gi.
- License: NOT stored. Copy `../template-trilio-license-configmap.yaml.example` to
  `trilio-license-configmap.yaml`, drop in the real key, apply to the hub `default` ns. Gitignored.

Apply on the ACM hub:
```
oc apply -f trilio-backup-configmap.yaml
oc apply -f trilio-license-configmap.yaml     # your filled copy (gitignored)
```
