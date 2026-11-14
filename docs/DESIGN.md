# Design: ADO workshop on RHDP

Status: **draft**. Decisions made so far, a proposed architecture, and the
questions still open. Platform details are in
[RHDP-PLATFORM.md](RHDP-PLATFORM.md).

## Goal

Show customers and partners what ADO does: answer a few questions in the
Preflight UI, and ADO generates a Git repository of playbooks and the AAP job
templates that deploy OpenShift Day 1 apps.

## Decisions (2026-11-13)

| Topic | Decision |
|---|---|
| Format | One dedicated cluster per attendee (ADO installs cluster-wide Operators, so attendees cannot share a cluster) |
| Storyline | Preflight → bootstrap → AAP, plus OpenShift Day 1 apps |
| ADO paths to show | All three: ADO pushes the generated playbooks to Git; ADO integrates them into AAP (project, credentials, job templates); and Preflight runs the playbooks directly from its container |
| Goal | A robust lab that fully shows off ADO |
| Catalog item | Field Sourced Content - OpenShift (Dedicated Clusters) |
| ADO collection | **No changes to the ADO collection.** Use ADO as released. |
| Preflight form | **Attendees fill out the Preflight form themselves** (that is the point of the workshop). No pre-filled JSON; the guide shows each seat's values to type. |
| AAP without Hub | Seat setup turns off the Controller setting *Enable Collection(s) Download* (`AWX_COLLECTIONS_ENABLED`); jobs use the collections inside `ado-ee` (open item 2, option a). |
| Infrastructure | **CNV, multi-node** (decided 2026-11-13; replaces AWS). About 15 minutes to provision instead of 60+, cheaper, sizing by worker count, CPU, and memory. Nothing in the storyline needs AWS. |
| Not using AAP module | Attendees select **Admin HTPasswd** (user `ado-admin`), **Console Banner**, **ACM**, **OpenShift Virtualization**, and **MTV**, then run them from the Preflight container. The guide has one page per Preflight step, in the form's order (decided 2026-11-13). All five work on the default RBD StorageClass (no RWX needed). |

## Proposed architecture (to be confirmed)

**Order settings:** CNV multi-node, AAP base workload, Existing GitOps Repo =
this repo, GitOps Path = `deploy`, Deploy Showroom = yes. Virtualization base:
see open item 1.

**Deployed per seat by this repo** (likely the Ansible Runner pattern,
because it needs waits, generated passwords, and API calls):

1. **ADO Preflight UI** as a Deployment plus Route, pre-filled where possible
   with the cluster API URL, AAP controller URL, and Git repo details.
2. **An in-cluster Git server** (for example Gitea) with an attendee account
   and an empty repository for the generated bootstrap repo. AAP syncs its
   project from there.
3. **A `userinfo` ConfigMap** with the Preflight URL and Git server login, so
   they appear in the order details and the Showroom guide.

**Attendee flow (live):**

1. Tour the environment: OpenShift console, AAP, Git server, Preflight.
2. In Preflight, pick OpenShift Day 1 apps and review the pre-filled settings.
3. Run Bootstrap: the playbook repo appears in Git, and the job templates
   appear in AAP.
4. Run the job templates in AAP to deploy the apps, and watch them come up in
   OpenShift.
5. Run playbooks straight from the Preflight container (no AAP), and compare
   with the AAP path.
6. Wrap-up: what changed, how ADO keeps it repeatable (re-run, day 2).

Candidate Day 1 apps, to pick by deploy time and cluster size: ACM, ACS,
Compliance, MTV, OpenShift Virtualization. On the earlier test cluster, ACM
took about 9 minutes to reach Running.

## Open items

1. **Virtualization base.** Checking it installs OpenShift Virtualization
   before ADO runs, so ADO's Virtualization deploy would only reuse it.
   Leave it unchecked if ADO should deploy Virtualization itself. VMs run on
   RHDP CNV-hosted clusters (the Shift on Shift test on an RHDP cluster booted
   hosted worker VMs), but confirm on a test order.
2. **No Automation Hub — answered (phase 1).** Bootstrap itself runs
   without a Hub: its "Hub mode" checks only look at the repo layout, and the
   Hub push/pull options default to off. The execution environment is set by
   the Preflight fields `aap.execution_environment` and
   `aap.execution_environment_image`; point them at
   `ghcr.io/automation-development-office/ado-ee:latest` (public). That EE
   already contains `infra.automation_development_office` 2.1.1, the
   `redhat.*` certified collections (openshift, openshift_virtualization,
   rhel_idm, satellite, rhel_system_roles, ...), `kubernetes.core`, and
   `infra.aap_configuration`.
   **Remaining problem:** the generated repo always has
   `collections/requirements.yml` (vendored mode was removed: "Controller
   collections always from Hub"), and `infra.automation_development_office`
   is **not on public Galaxy**, so Controller project sync would fail trying
   to install it. Options:
   - **(a) No ADO change — chosen.** The seat setup turns off the Controller
     setting *Enable Collection(s) Download* (`AWX_COLLECTIONS_ENABLED`,
     category `jobs`; AWX `awx/main/conf.py`: "Allows collections to be
     dynamically downloaded from a requirements.yml file for SCM projects").
     Project sync then skips `requirements.yml` and jobs use the EE's
     collections. ADO's bootstrap does not apply Controller settings, so it
     stays off. One API call with the AAP admin credentials
     (`PATCH .../api/controller/v2/settings/jobs/` on AAP 2.5+; the exact URL
     depends on the RHDP AAP version, open item 9).
   - (b) ADO enhancement ("no Hub: use the EE's collections"): not doing;
     the ADO collection stays unchanged.
3. **Preflight UI in-cluster — answered (phase 1).** Supported already:
   the Preflight repo ships `deploy/preflight.yaml` (namespace, 5Gi RWO PVC
   for `/workspace`, Deployment with probes, Service, edge-TLS Route with a
   15-minute timeout). The image is public:
   `ghcr.io/automation-development-office/ado-preflight-ui` (`latest`,
   versioned prerelease tags). It runs as a random UID (group-0 writable
   directories). Cluster access is the API URL and token typed in the form;
   no kubeconfig mount is needed. Related findings:
   - **Git server:** Preflight's SCM choices are GitLab, Bitbucket, GitHub,
     and "other". ADO only uses plain HTTPS git with a username and token (no
     SCM-specific API), so an in-cluster **Gitea works as "other"**. The
     bootstrap repo must already exist with a first commit; the seat setup
     creates it.
   - **Pre-filling the form:** only through **Upload JSON** (file import).
     There is no environment-variable or URL pre-fill. See open item 7.
   - **Running playbooks from the container:** supported. After Bootstrap,
     the Step 6 button changes to **Run Playbooks**, which runs the generated
     components inside the pod (no AAP) in workflow order, stopping at the
     first failure; **Preview commands** in the *Generated Playbooks* panel
     shows them first.
4. **Exact Helm value names — answered (first test order).** `aap.url`,
   `aap.adminUser`, `aap.adminPassword`, `bases.*`, `guid`, `openshift.*`,
   `keycloak.*`; see [RHDP-PLATFORM.md](RHDP-PLATFORM.md). The `aap` role
   still finds AAP on the cluster, which matched: `AnsibleAutomationPlatform`
   `aap` in namespace `aap` (AAP 2.6), Route `aap`, Secret
   `aap-admin-password`, API `/api/controller/v2/`.
5. **Timing.** Provisioning takes about 15 minutes on CNV multi-node; the
   live part must fit the session with deploys that take several minutes
   each.
6. **CNV checks** (first test order: default StorageClass
   `ocs-external-storagecluster-ceph-rbd`, CephFS available for
   ReadWriteMany; 3 control-plane nodes 16 vCPU / 64 GiB and 3 workers
   8 vCPU / 32 GiB; Showroom ran; sizing with Day 1 apps still to measure):
   the default StorageClass and
   whether it supports ReadWriteMany (Virtualization and ACS storage); how
   much CPU and memory the chosen Day 1 apps use, to set worker count and
   size (ACM alone took about 9 minutes to install on a test cluster); and
   that the AAP base and Showroom behave as described on CNV.

7. **How attendees fill in Preflight — decided.** They fill out the form
   themselves. The guide shows each seat's values (cluster API URL, token
   steps, AAP URL and login, Gitea URL, user, token, repo) as Showroom
   attributes from the `userinfo` ConfigMap.
8. **Collection versions.** Found on the first test order: the Preflight
   `latest` tag is app version 1.2.0 and bundles the **old `infra.ado`
   2.1.0** collection, so generated playbooks would call `infra.ado.*` roles
   that `ado-ee` lacks. Pinned `ado-preflight-ui:5.0.1-prerelease` (bundles
   `infra.automation_development_office` 2.1.4) and `ado-ee:2.1.1` (bundles
   2.1.1). Remaining gap: playbooks generated with 2.1.4 run with 2.1.1 roles
   in AAP; check in the dry run.
9. **AAP version on RHDP — answered.** AAP 2.6 (operator
   `aap-operator.v2.6.0`). In the Preflight form, pick AAP version 2.6.

10. **Job template EE image.** The generated job templates use the EE named
    in Preflight's *Execution Environment* field (`ee-supported-rhel9`), and
    its image defaults to Red Hat's `ee-supported-rhel9`, which lacks the ADO
    collection. The form has no image field, so without a Preflight change
    the AAP path needs the EE pointed at `ado-ee` (for example a guide step in
    AAP after Bootstrap). No Preflight edits for now (2026-11-13).

11. **Preflight form (image `5.0.1-prerelease`), mapped 2026-11-13** by
    running the pinned image locally and walking the form in a headless
    browser (no cluster changes). Six steps: Environment & Platform
    Selection, Git configuration, Platform Configuration, Components &
    Profiles, Review & Validate, Bootstrap Execution. Findings the guide
    relies on:
    - Step 1 defaults to **Disconnected** and **Using AAP**; attendees must
      change both. The vault password is in Step 1.
    - Step 2 *Automatically push generated automation to Git* is under
      *Advanced Git Settings* and is off by default.
    - Step 3 API URL defaults to an example value; *Skip TLS* is on by
      default (the RHDP API certificate is not publicly trusted; the Gitea
      route certificate is).
    - Admin HTPasswd is on Preflight's "during-bootstrap" list, but the form
      never sets `openshift.install_htpasswd_during_bootstrap`, so Bootstrap
      only generates `playbooks/openshift/ado-create-htpass-user-bootstrap.yml`
      and the runner applies it. The role appends its IdP next to `rhbk`.
    - Generated secrets are not pushed: `ado-preflight-<env>.json`,
      `ado-extra-vars.json`, and `.vault_pass` stay in the pod; the
      `vault_*.yml` files in Gitea are encrypted.
12. **To verify in the dry run (Not using AAP module):** total run time of
    the five playbooks (guide says about 20 to 30 minutes); console labels
    for ACM (*Fleet Management*), Virtualization, and MTV on OCP 4.22;
    the HTPasswd playbook in a real Bootstrap and the `htpasswd-admin`
    login; the wrap-up "change an answer" flow; worker headroom with AAP
    also installed.

## Next steps

1. ~~Answer open items 2 and 3 by reading the ADO code.~~ Done (phase 1).
2. Add a module outline with timing (open items 2 and 7 are decided).
3. ~~Replace the placeholder seat setup.~~ Done: `deploy/playbooks/site.yml`
   with roles `gitea`, `preflight`, `aap` (lint and syntax checks pass; not yet
   run on a cluster).
4. ~~First test order.~~ Done on cluster-5pxrp: seat setup succeeded after
   two fixes (Gitea CRD and RBAC name `gitea`; job clones the order's repo
   and revision). Preflight UI, Gitea (`lab-user`, `ado-bootstrap`, token),
   AAP collection download off, `userinfo` published, Showroom running.
   About 6 minutes for the seat setup on a clean run.
5. Replace the placeholder guide page with the workshop modules.

Done: the starter scaffold from the template (Showroom guide at the root,
`deploy/` Ansible Runner chart; `helm lint` passes and the guide builds).
