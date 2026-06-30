# components/gitops

Configuration for the OpenShift GitOps (Argo CD) instance that runs this repo.

- The operator itself is installed once via `bootstrap/01-openshift-gitops-operator.yaml`,
  which creates the Argo instance in namespace `openshift-gitops`.
- `argocd-cluster-admin.yaml` grants that instance the permissions it needs to install
  operators and ACM policies cluster-wide.

This component is intentionally NOT a child app in `apps/` (chicken-and-egg: Argo can't
manage the binding that grants it permission before it has permission). Apply it once,
right after the operator is Succeeded:
```
oc apply -f components/gitops/argocd-cluster-admin.yaml
```
