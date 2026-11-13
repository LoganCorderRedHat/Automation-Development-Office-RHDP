# RHDP platform: Field Sourced Content - OpenShift (Dedicated Clusters)

How the catalog item this workshop builds on works. Facts are from the
sources listed at the end; anything not confirmed is marked **TBD**.

## The catalog item

- **Dedicated Clusters:** one full cluster per order or per workshop seat.
  Each person logs in as the cluster admin. Workshops are limited to 10 seats
  (ask for approval for more). The **Shared Clusters** variant gives numbered
  users on one cluster, up to 30.
- **Cloud provider:**
  - **CNV:** cheapest and fastest (~5 min SNO, ~15 min multi-node).
  - **AWS:** slower (60+ min, listed as about 53 min), needed for AWS
    integration. Optional **Open Environment** gives AWS account credentials
    and full control of auto-stop and auto-destroy.
- **Size and version:** SNO or multi-node; OpenShift 4.20, 4.21, or 4.22.
  Worker count tops out at 5.
- **Always included:** an authentication provider (RHBK) and OpenShift GitOps.
- **Optional base workloads** (installed before GitOps runs):
  - **OpenShift Virtualization:** Operator plus HyperConverged. On AWS
    multi-node, workers are `m7i` instances with EC2 nested virtualization,
    so VMs run with hardware KVM. On SNO, VMs run with QEMU software
    emulation. With this base on AWS, worker types are `m7i.2xlarge`
    (default) or `m7i.4xlarge`.
  - **OpenShift AI 3:** Operator plus Data Science Cluster, CPU only by default.
  - **Ansible Automation Platform:** a licensed **Automation Controller**.
    Automation Hub, EDA, and AAP Lightspeed are **not** included.
- **Other options:** OpenShift Lightspeed; LiteMaaS/MaaS temporary model API
  keys (3, 7, 10, or 30 days).
- **Lifetime:** auto-stop 6 hours, auto-destroy 48 hours.

## How our repo gets deployed

1. When ordering, check **Existing GitOps Repo?** and give the repo URL,
   revision, and **GitOps Path** (the directory that holds the Helm chart to
   start from). Private GitHub repos work with an access token.
2. The `ocp4_workload_field_content` role (AgnosticD v2 core workloads)
   creates one Argo CD Application named `field-content` in
   `openshift-gitops`, pointing at that path. Automated sync, no prune, no
   self-heal, `CreateNamespace=true`.
3. The chart at that path is either:
   - **Helm app-of-apps** (`examples/helm`): each component under
     `components/` becomes its own Argo Application and can be switched on or
     off.
   - **Ansible Runner Job** (`examples/ansible`): a Job clones the repo,
     installs the listed Python packages and collections, writes a kubeconfig
     from its service account, and runs the playbook. The template recommends
     this pattern when you need waits, generated secrets, API calls, or
     conditional logic.
4. The platform does **not** monitor or verify what GitOps deploys. A bad
   chart does not fail the order.

### Values passed to the chart

Confirmed in `ocp4_workload_field_content` (`tasks/workload.yml`):

| Value | Notes |
|---|---|
| `deployer.domain` | Cluster ingress domain |
| `deployer.apiUrl` | OpenShift API URL |
| `gitops.repoURL`, `gitops.revision`, `gitops.path` | The order's GitOps settings |
| `litemaas.enabled`, `litemaas.apiUrl`, `litemaas.apiKey`, `litemaas.model` | When LiteMaaS is enabled |
| `multi_user.num_users`, `multi_user.users` | Shared Clusters only |

Merged from the catalog item's Helm values (set in the private AgnosticV
configuration). The catalog description lists these, but the **exact key
names are TBD** until read from a real order
(`oc get application field-content -n openshift-gitops -o yaml`):

- `guid`, `user.count` (always 0 for Dedicated Clusters)
- `bases.virt`, `bases.rhoai`, `bases.aap` (booleans for the selected base workloads)
- OpenShift console URL and admin credentials
- Keycloak console URL and admin credentials
- AAP controller URL and admin credentials (when AAP is selected)
- OpenShift AI dashboard URL (when AI is selected)

### Passing data back to RHDP

- Label resources `demo.redhat.com/application: "<name>"` for health
  monitoring.
- A ConfigMap labeled `demo.redhat.com/userinfo: ""` passes its `data` back
  to AgnosticD user data. It appears in the order details and as Showroom
  attributes (for example `{preflight_url}` in the guide).

## Showroom lab guide

- Check **Deploy Showroom?** when ordering. Showroom builds from the **repo
  root**: `site.yml`, `ui-config.yml`, and `content/` (with `antora.yml`).
- The terminal tab is the OpenShift web terminal (`/terminal`, set in
  `ui-config.yml`). Embedded OpenShift console tabs do not work.
- A bad layout does not fail the order and the order does not wait for
  Showroom. A clone error or missing `site.yml`/`content/` gives a warning;
  AsciiDoc or `antora.yml` errors give none. Check the `git-cloner` and
  `antora-builder` container logs in the `showroom-<guid>` namespace.
- For a private repo, use a read-only token valid until the environment is
  retired. Showroom users can read it, and it is reused when the pod
  restarts.
- Build locally before pushing:

  ```bash
  podman run --rm -v "$PWD:/antora:z" -w /antora --entrypoint antora quay.io/rhpds/antora:v1.3.0 site.yml
  ```

## RHDP team announcement (FSC update rollout)

Points from the RHDP maintainers' announcement, beyond what is above:

- **Each user gets their own Showroom link and guide**, including in a
  Dedicated Clusters workshop. Both Dedicated and Shared Clusters support the
  **workshop UI**.
- **Content-only guides are recommended.** Integrated terminals work;
  embedded OpenShift consoles are not supported in this version. For this
  workshop: attendees open Preflight, AAP, Gitea, and the OpenShift console in
  their own browser tabs, and the guide links to them.
- **Do not deploy Showroom yourself** when **Deploy Showroom?** is checked;
  that is the one case that is not compatible with older FSC repos.
- **Responsibility split:** RHDP supports cluster provisioning and the core
  workloads. Whether our automation succeeds in setting up the workshop is
  **entirely our responsibility**; RHDP does not support it.
- **Onboarding path:** FSC is planned to become the way to onboard an item to
  RHDP for others to order.
- **FSC for RHEL** is a separate catalog item for RHEL-based content
  (possible future home for ADO's RHEL, Satellite, and IdM story).
- The MaaS model list was refreshed.

## Sources

- RHDP team announcement of the FSC changes, shared by the repo owner on
  2026-11-13.
- RHDP catalog description for "Field Sourced Content - OpenShift (Dedicated
  Clusters)", shared by the repo owner on 2026-11-13.
- <https://github.com/rhpds/field-sourced-content-template/pull/5> (merged;
  GPTEINFRA-18402) and the template's `main` at commit `6721b95`.
- `ocp4_workload_field_content` in <https://github.com/rhpds/core_workloads>
  (`templates/application.yaml.j2`, `tasks/workload.yml`, `defaults/main.yml`).
