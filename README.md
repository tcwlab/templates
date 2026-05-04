# tcwlab/templates

Forgejo workflow templates for CI pipelines. Drop-in starting point for repos across The Chameleon Way.

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

---

## Quick start

Copy the template that matches your repo type to `.forgejo/workflows/ci.yml`, then replace the values marked with `# ── ADJUST ──`.

```bash
# For IaC/OpenTofu repos
cp iac-ci.yml myrepo/.forgejo/workflows/ci.yml

# For Kotlin/Micronaut/Go services
cp service-ci.yml myrepo/.forgejo/workflows/ci.yml

# For Docker image wrapper repos
cp docker-image-ci.yml myrepo/.forgejo/workflows/ci.yml
```

Then edit `ci.yml` and adjust the placeholders at the top. No further configuration needed — the templates are designed to work out of the box.

---

## Available templates

| Template | Target repo type | What it does |
| -------- | ------------- | --------------- |
| [`iac-ci.yml`](iac-ci.yml) | OpenTofu / IaC | Lint (betterlint, tflint), format validation, syntax checks. No deployment. |
| [`service-ci.yml`](service-ci.yml) | Kotlin/Micronaut/Go services | Lint (betterlint, OpenAPI, Gherkin), placeholder for build jobs. Image push/release job commented out — uncomment when ready. |
| [`docker-image-ci.yml`](docker-image-ci.yml) | Docker image wrapper repos (tcwlab internal use) | Lint, build, Trivy security scan with PR comment, multi-arch release to Docker Hub, auto-tag. Most elaborate template. |

All three follow a consistent job pattern: **Lint → Build/Test → Security → Release**. You can comment out or delete jobs that don't apply to your repo.

---

## Customization points

Each template has clear `# ── ADJUST ──` markers for values you must change:

**iac-ci.yml:**
- `OPENTOFU_VERSION` — from [`tcwlab/versions.yaml`](https://git.mon.k8b.co/tcwlab/)
- `BETTERLINT_VERSION` — from [`tcwlab/versions.yaml`](https://git.mon.k8b.co/tcwlab/)
- `permissions:`, `concurrency:`, `branches:` — to match your project's branching strategy

**service-ci.yml:**
- `BETTERLINT_VERSION` — from [`tcwlab/versions.yaml`](https://git.mon.k8b.co/tcwlab/)
- Build/test steps — uncomment and adapt the `# TODO: build-Job` section for your language (Gradle for Kotlin, `go build` for Go, etc.)

**docker-image-ci.yml:**
- `IMAGE: tcwlab/<toolname>` — replace `<toolname>` with your image name
- `BETTERLINT_VERSION`, `TRIVY_VERSION` — from [`tcwlab/versions.yaml`](https://git.mon.k8b.co/tcwlab/)
- Smoke-test command in the build step (currently `docker run ... --version`)
- Version extraction in the release step (currently `ARG <TOOL>_VERSION=...`)
- Repository secrets: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `FORGEJO_TOKEN`

---

## Version pinning

Always pin specific versions, never use `latest`:

- **Images**: `tcwlab/betterlint:2.12.0` — get exact versions from [`tcwlab/versions.yaml`](https://git.mon.k8b.co/tcwlab/)
- **Composite Actions**: `@v1` or `@v1.2.0` — major+minor is fine, don't use `@main`

Pinning ensures your pipeline doesn't break when tcwlab ships a new version. It's your explicit choice when to upgrade.

---

## How this repo fits into tcwlab

- **Templates** (this repo) → drop-in YAML files for consumer repos
- **Actions** (tcwlab/actions) → reusable Composite Actions for fine-grained logic
- **Images** (tcwlab/{betterlint,opentofu,buildx,trivy,semantic-release}) → Docker Hub

Templates are _copied_, not referenced via `uses:`. This means your repo owns the full CI workflow and can edit it freely.

---

## Source

- **Forgejo**: `git.mon.k8b.co/tcwlab/templates`
- **GitHub mirror**: `github.com/tcwlab/templates` (read-only, synced)

Report issues or suggest improvements on the GitHub mirror.

---

## License

Apache-2.0 — The Chameleon Way.
