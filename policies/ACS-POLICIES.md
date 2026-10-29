# Red Hat Advanced Cluster Security (ACS) configuration

ACS is the primary policy engine for the capstone. It gates images in the
pipeline (`roxctl image check` in the `acs-image-check` task) and at
admission (deploy-time enforcement). Configure these in the ACS console.

## 1. API token for the pipeline

Platform Configuration → Integrations → Authentication Tokens → API Token →
Generate. Role: **Continuous Integration**. Put it in the `roxsecrets` secret
(see `scripts/create-cicd-secrets.sh`).

## 2. Registry integration

Platform Configuration → Integrations → Image Integrations → add **Quay.io**
(or your Quay) with a robot account that can pull `catalogue-api`, so ACS can
scan the image the pipeline pushed.

## 3. Signature integration (Cosign public key)

Platform Configuration → Integrations → Signature Integrations → New →
name `catalogue-cosign` → Cosign public key → paste `cosign.pub`.

## 4. Policies

Clone the default policies below (Policy Management → clone), scope them to
the namespaces `catalogue-dev`, `catalogue-test`, `catalogue-prod`, and set
the enforcement shown.

| Policy (clone of default) | Lifecycle | Enforcement | Catches |
| --- | --- | --- | --- |
| Fixable Severity at least Important | Build, Deploy | Fail build; scale to zero at deploy | Critical/important CVEs with a fix (planted vulnerable dependency) |
| Trusted image signers *(new policy, criterion "Image signature verified by" = `catalogue-cosign`)* | Deploy | Block at admission | Unsigned or tampered images |
| Container using latest tag / Image tag not pinned | Build, Deploy | Fail build | Mutable tags |
| Containers running as root user *(enable "Run as root" criterion)* | Deploy | Block at admission | Planted root Deployment |
| No resource requests or limits specified | Deploy | Block at admission | Planted Deployment without limits |
| Privileged Container | Deploy | Block at admission | Privileged pods |
| Secret Mounted as Environment Variable | Deploy | Inform only | Visibility (the app intentionally uses env-var secrets) |
| Shell Spawned by Java/Node Application *(runtime)* | Runtime | Inform (or kill pod) | Runtime anomalies |

Admission enforcement needs the admission controller enabled on the secured
cluster (`SecuredCluster` CR → `admissionControl.listenOnCreates: true`,
`listenOnUpdates: true`, `enforceOnUpdates: true`).

## 5. Evidence to collect

```bash
# build-time: the policy table from the acs-image-check TaskRun log
tkn taskrun logs -n catalogue-cicd <run>-acs-image-check

# deploy-time: try to run an unsigned image - expect a denial from ACS
oc -n catalogue-dev run unsigned-test --image=registry.access.redhat.com/ubi9/ubi-minimal:latest -- sleep 60
```
