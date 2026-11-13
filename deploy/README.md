# deploy/ — per-seat setup chart

The GitOps entry point for this workshop. When ordering the RHDP item, set:

- **GitOps URL:** this repository
- **GitOps Path:** `deploy`
- **Base workloads:** Ansible Automation Platform

Argo CD renders this chart: a namespace, RBAC, and an Ansible Runner Job (from
`examples/ansible` in <https://github.com/rhpds/field-sourced-content-template>).
The Job clones this repository and runs [`playbooks/site.yml`](playbooks/site.yml),
which sets up the seat without filling in the Preflight form:

| Step | Role | Result |
|---|---|---|
| 1 | [`gitea`](playbooks/roles/gitea) | RHDP Gitea operator, a Gitea instance in `gitea`, attendee account `lab-user`, empty repo `ado-bootstrap` (first commit only), and an access token saved in Secret `ado-workshop/ado-workshop-gitea-token` |
| 2 | [`preflight`](playbooks/roles/preflight) | ADO Preflight UI in `ado-workshop` (published image, workspace PVC, Route with a 15-minute timeout) |
| 3 | [`aap`](playbooks/roles/aap) | Finds the AAP instance from the base workload and turns off *Enable Collection(s) Download* (`AWX_COLLECTIONS_ENABLED`), so jobs use the collections in the `ado-ee` image (there is no Automation Hub) |
| 4 | `site.yml` | `userinfo` ConfigMap `ado-workshop/ado-workshop-userinfo`: Preflight URL, Gitea URL, user, password, token, bootstrap repo URL, AAP URL and admin login, EE image |

Re-running is safe: existing objects are kept, and the Gitea token is reused
from its Secret.

**Updating an existing seat.** The setup Job is an Argo CD Sync hook, and Argo
CD does not compare hooks:

- Changing `ansible.extraVars` (for example an image tag) also changes the
  tracked ConfigMap, so auto-sync re-runs the setup.
- Changing only the playbooks or roles does **not** trigger a sync. Re-run the
  setup with **Sync** on the `field-content` application in the Argo CD
  console.

The Job clones the repository URL and revision from the RHDP order
(`gitops.repoURL`, `gitops.revision`), so it always runs the same revision as
the chart.

Settings are in [`values.yaml`](values.yaml) under `ansible.extraVars`
(`workshop_namespace`, `preflight_image_override`, `ado_ee_image`). Pin tested
image tags before running the workshop.

Debug a seat:

```bash
oc logs job/field-content-ado-workshop-setup -n ado-workshop-setup
oc get configmap ado-workshop-userinfo -n ado-workshop -o yaml
```

Check the chart renders:

```bash
helm template field-content deploy/ --set deployer.domain=apps.example.com --set deployer.apiUrl=https://api.example.com:6443
```
