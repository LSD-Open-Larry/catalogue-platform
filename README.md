# catalogue-platform

Everything the platform team runs on the cluster, managed by Argo CD
(OpenShift GitOps). After the one-time bootstrap, every change here is a pull
request; nobody runs `oc apply`.

```
bootstrap.yaml               root Application "cluster-bootstrap" - the ONE manual apply
argocd-instance-patch.yaml   one-time patch: Application health check + Argo CD RBAC
apps/                        synced by cluster-bootstrap, in sync-wave order
  00-projects.yaml           wave -1  AppProject "platform"
  10-operators.yaml          wave 0   platform-operators (operators/), sealed-secrets (Helm)
  20-cluster-config.yaml     wave 1   cluster-config  -> cluster/
  30-cicd.yaml               wave 2   cicd            -> cicd/
  40-policies.yaml           wave 3   policies        -> policies/
  50-catalogue.yaml          wave 4   catalogue-root  -> catalogue-config repo, apps/
operators/                   OLM Subscriptions (Red Hat ACS)
cluster/                     namespaces, RBAC, user workload monitoring
cicd/                        Tekton Tasks and Pipelines, PaC Repository CRs, (sealed CI secrets)
policies/                    admission policies (Kyverno option; ACS notes)
```

## One-time steps (cannot be GitOps)

1. OpenShift GitOps installed (default instance in `openshift-gitops`).
2. Repository secrets so Argo CD can read `catalogue-platform` and
   `catalogue-config` (if private) - see the header of `bootstrap.yaml`.
3. Cluster-scoped permissions for the Argo CD application controller
   (`cluster-admin` in a lab, a narrower ClusterRole in production).
4. `oc patch argocd openshift-gitops -n openshift-gitops --type merge --patch-file argocd-instance-patch.yaml`
5. `oc apply -f bootstrap.yaml`
6. Once the Sealed Secrets controller is running: create and seal the CI and
   app secrets (`scripts/create-cicd-secrets.sh`, `scripts/seal-secrets.sh <env>`)
   and commit the sealed files.

## Safety choices

- `prune: false` on `platform-operators` and `cluster-config`: deleting a file
  never uninstalls an operator or deletes a namespace with its data.
- No resources finalizer on the platform apps: deleting an Application by
  mistake does not cascade-delete what it deployed.
- Retries with backoff everywhere, because later waves wait on operators and
  CRDs from earlier ones.
