# ADO workshop for Red Hat Demo Platform

A hands-on workshop that shows ADO (Automation Development Office) automating
an OpenShift cluster with Ansible Automation Platform. Built on the RHDP
**Field Sourced Content - OpenShift (Dedicated Clusters)** catalog item.

| Path | What it is |
|---|---|
| `site.yml`, `ui-config.yml`, `content/` | Showroom lab guide (deployed by RHDP when **Deploy Showroom?** is checked) |
| `deploy/` | GitOps entry point: per-seat setup chart (Ansible Runner job) |
| `docs/` | Platform notes and design |
| `AGENTS.md` | Context for AI assistants and contributors |

Build the guide locally (output in `www/`):

```bash
podman run --rm -v "$PWD:/antora:z" -w /antora --entrypoint antora quay.io/rhpds/antora:v1.3.0 site.yml
```

With rootless podman (for example on Fedora) that command can fail with
`EACCES: permission denied, mkdir '/antora/www'`. Run it as your own user and
keep the Antora cache in the repo (ignored by git):

```bash
podman run --rm --security-opt label=disable --user "$(id -u):$(id -g)" --userns=keep-id \
  -e ANTORA_CACHE_DIR=/antora/.cache/antora -v "$PWD:/antora" -w /antora \
  --entrypoint antora quay.io/rhpds/antora:v1.3.0 site.yml
```

Open `www/index.html` to view it.
