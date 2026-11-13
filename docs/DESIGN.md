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
     **Run selected playbooks** runs the generated components inside the pod
     (no AAP); **Preview commands** shows them first.
4. **Exact Helm value names** for the AAP controller URL and credentials,
   `guid`, and `bases.*`: read them from a real order.
5. **Timing.** Provisioning takes about 15 minutes on CNV multi-node; the
   live part must fit the session with deploys that take several minutes
   each.
6. **CNV checks on the first test order:** the default StorageClass and
   whether it supports ReadWriteMany (Virtualization and ACS storage); how
   much CPU and memory the chosen Day 1 apps use, to set worker count and
   size (ACM alone took about 9 minutes to install on a test cluster); and
   that the AAP base and Showroom behave as described on CNV.

7. **How attendees fill in Preflight — decided.** They fill out the form
   themselves. The guide shows each seat's values (cluster API URL, token
   steps, AAP URL and login, Gitea URL, user, token, repo) as Showroom
   attributes from the `userinfo` ConfigMap.
8. **Collection versions.** The AAP path uses the collection inside `ado-ee`
   (2.1.1); the container path uses the collection baked into the Preflight
   image. Pin both image tags and check they carry the same collection
   version.
9. **AAP version on RHDP.** Preflight's default EE image is built from the AAP
   version field (default 2.7); we override the image anyway, but record the
   RHDP AAP base version and the credential value names from a real order.

10. **Job template EE image.** The generated job templates use the EE named
    in Preflight's *Execution Environment* field (`ee-supported-rhel9`), and
    its image defaults to Red Hat's `ee-supported-rhel9`, which lacks the ADO
    collection. The form has no image field, so without a Preflight change
    the AAP path needs the EE pointed at `ado-ee` (for example a guide step in
    AAP after Bootstrap). No Preflight edits for now (2026-11-13).

## Next steps

1. ~~Answer open items 2 and 3 by reading the ADO code.~~ Done (phase 1).
2. Add a module outline with timing (open items 2 and 7 are decided).
3. ~~Replace the placeholder seat setup.~~ Done: `deploy/playbooks/site.yml`
   with roles `gitea`, `preflight`, `aap` (lint and syntax checks pass; not yet
   run on a cluster).
4. First test order on RHDP to run the seat setup for real.
5. Replace the placeholder guide page with the workshop modules.

Done: the starter scaffold from the template (Showroom guide at the root,
`deploy/` Ansible Runner chart; `helm lint` passes and the guide builds).
