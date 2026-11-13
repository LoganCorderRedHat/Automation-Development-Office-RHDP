# AGENTS.md — ADO workshop for Red Hat Demo Platform

Read this first. It gives an AI assistant (or a new contributor) the context
needed to work in this repository.

## What this repository is

An RHDP **Field Sourced Content** catalog item that showcases ADO to customers
and partners. Each attendee gets a dedicated OpenShift cluster. The platform's
OpenShift GitOps deploys this repo's chart, and Showroom renders the lab guide
from this repo's root.

- Platform facts and sources: [docs/RHDP-PLATFORM.md](docs/RHDP-PLATFORM.md)
- Decisions, architecture, open items: [docs/DESIGN.md](docs/DESIGN.md)
- Upstream template this repo follows:
  <https://github.com/rhpds/field-sourced-content-template> (PR #5 layout:
  Showroom guide at the repo root, deployment chart under a path such as
  `examples/ansible`)

## What ADO is

**ADO (Automation Development Office)** is two projects:

1. **`infra.automation_development_office`** — an Ansible collection of roles
   and modules for building and operating platform automation across Ansible
   Automation Platform (AAP), OpenShift, RHEL, Satellite, identity services
   (IdM, RHBK), and tooling (Grafana, GitLab, Kafka, and more). It includes
   the bootstrap roles that generate a playbook repository and AAP objects.
   - Upstream: <https://github.com/redhat-cop/infra.automation-development-office>
2. **ADO Preflight UI** — a containerized web questionnaire. You answer
   environment questions (environment name, base domain, components such as
   OpenShift apps, Git repo, AAP connection). **Run Bootstrap** writes a
   preflight JSON and runs Ansible inside the container to generate a
   bootstrap Git repository (vars, vault files, playbooks), optionally push it
   to Git, and create the AAP projects, credentials, and job templates.
   - Upstream: <https://github.com/Automation-Development-Office/ado-preflight-ui>

The ADO story for the workshop: answer the Preflight questions → Run
Bootstrap → a Git repo and AAP job templates appear → run the job templates in
AAP to deploy OpenShift Day 1 apps.

## Decisions so far (2026-11-13)

- One dedicated cluster per attendee (ADO installs cluster-wide Operators).
- Storyline: Preflight → bootstrap → AAP, plus OpenShift Day 1 apps. Show
  all three ADO paths: push playbooks to Git, integrate with AAP, and run
  playbooks directly from the Preflight container. Goal: a robust lab that
  fully shows off ADO.
- Infrastructure: RHDP "Field Sourced Content - OpenShift (Dedicated
  Clusters)", **CNV multi-node** (not AWS).
- Everything else is open; see [docs/DESIGN.md](docs/DESIGN.md).

## Ground rules

- Commit or push only when the repo owner asks.
- Never commit secrets, tokens, pull secrets, kubeconfigs, or Preflight
  exports. Values the platform injects at order time stay out of git.
- Do not run installs or deploys against the owner's clusters; read-only
  inspection only when invited.
- Do not invent RHDP or ADO capabilities. Verify against the template, the
  `ocp4_workload_field_content` role, or the ADO repos, and note the source.
- Build the Showroom guide locally before pushing:

  ```bash
  podman run --rm -v "$PWD:/antora:z" -w /antora --entrypoint antora quay.io/rhpds/antora:v1.3.0 site.yml
  ```

  If that fails with `EACCES` under rootless podman, use the variant in
  [README.md](README.md).

## Local checkouts (repo owner's machine)

- This repo: `~/ado-rhdp/Automation-Development-Office-RHDP`
- Template reference clone: `~/ado-rhdp/field-sourced-content-template`
- ADO collection and Preflight UI working copies: `~/hcp/infra.automation-development-office`,
  `~/hcp/ado-preflight-ui` (branch `hcp`, uncommitted Shift on Shift work)
