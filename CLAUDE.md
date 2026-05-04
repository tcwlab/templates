# templates — repo context

> **Onboarding handshake:** Read in this order:
>
> 1. [`Projects/CLAUDE.md`](https://git.mon.k8b.co/) (global standards)
> 2. [`tcwlab/CLAUDE.md`](https://git.mon.k8b.co/tcwlab/) (toolchain context, consumer API)
> 3. This file (templates-specific)

---

## What is `templates`?

`templates` is the repo where tcwlab publishes **Forgejo workflow templates** as drop-in-ready YAML files. Consumer repos copy the appropriate template to `.forgejo/workflows/ci.yml` (instead of referencing it via `uses:`) and adjust the `# ── ADJUST ──` placeholders at the top. Together with [`actions`](https://git.mon.k8b.co/tcwlab/actions), this repo forms the **consumer API** of the tcwlab toolchain.

We deliberately avoid `uses:` references for entire workflows for two reasons: (1) Forgejo does not yet support smooth reuse of whole workflow files (unlike GitHub's `workflow_call`); (2) we want consumers to see their full CI pipeline — not hidden behind a `uses:` door. Fine-grained reusability is handled by Composite Actions in `actions/`.

### Consumers

All TCW repos use templates at bootstrap. Pattern: `cp <tcwlab>/templates/<type>-ci.yml <repo>/.forgejo/workflows/ci.yml`, then adjust the ADJUST markers, then commit.

Current consumers in the TCW ecosystem are the service verticals (Atrium, Spectrum, Tally and others), the K8Box IaC repos, and the tcwlab image repos themselves.

---

## What's included?

Three templates, one for each repo type:

| Template | Target repo type | Image footprint |
| -------- | ------------- | --------------- |
| [`iac-ci.yml`](https://git.mon.k8b.co/tcwlab/templates/src/branch/main/iac-ci.yml) | OpenTofu / IaC | `tcwlab/betterlint`, `tcwlab/opentofu` |
| [`service-ci.yml`](https://git.mon.k8b.co/tcwlab/templates/src/branch/main/service-ci.yml) | Kotlin/Micronaut/Go services | `tcwlab/betterlint` (build jobs as placeholders) |
| [`docker-image-ci.yml`](https://git.mon.k8b.co/tcwlab/templates/src/branch/main/docker-image-ci.yml) | Docker image wrapper (tcwlab internal) | `tcwlab/betterlint`, `tcwlab/buildx`, `tcwlab/trivy`, `tcwlab/semantic-release` |

All three follow a shared job skeleton (Lint → Build/Test → Security → Release/Publish). Consumers delete jobs that don't apply to their situation — e.g. service repos not yet pushing containers omit the build-push job.

---

## Adding a new template

1. **PR to `claude/feat-<template-type>-template`**: create a new YAML file following the naming convention `<type>-ci.yml`.
2. **Header block** explaining "What is this for?", "When should I use it?", "What placeholders must the consumer repo set?".
3. **`# ── ADJUST ──` markers** at every location a consumer must change before their first push (image name, branches, secret refs).
4. **`env:` block** with all image versions at the top — follow the pattern in existing templates.
5. **Documentation** — update this file and [`templates/README.md`](https://git.mon.k8b.co/tcwlab/templates/src/branch/main/README.md).
6. **Smoke test**: run the new template end-to-end in a sandbox consumer repo before merging.

---

## Versioning strategy

Templates are versioned via **Git tags in the `templates` repo** (`v1.0.0`, `v1.1.0`). Consumers copy them physically — they do not reference them via `uses:`. This means:

- A template change only takes effect when a consumer copies it again.
- This is intentional — it prevents toolchain bumps from breaking consumer pipelines without a PR.
- Documentation requirement: for major template changes, add a changelog entry explaining what consumers need to adjust when re-applying.

semantic-release in the repo's `release` job generates tags automatically from Conventional Commits.

---

## Release process

Self-hosted like `actions`: the repo uses `tcwlab/betterlint` + `tcwlab/semantic-release` for linting and releases. Output: Forgejo tags + Forgejo releases with changelog. No Docker image output.

---

## What explicitly does NOT belong in this repo

- **`workflow_call` patterns**. Forgejo doesn't support them smoothly yet — we stick with drop-in copy.
- **Repo-specific logic**. Templates are generic. When a consumer needs something specific, they customize it in their own repo.
- **Inline complex logic that should be a Composite Action**. If a block appears in multiple templates, it belongs in [`actions/`](https://git.mon.k8b.co/tcwlab/actions).
- **Helm / Kustomize / OpenTofu specific modules**. Templates describe CI workflows, not IaC or deployment content.
- **Consumer-specific image names** (e.g., `harbor.k8b.io/atrium/idp`). Templates use generic placeholders (`tcwlab/<toolname>`, `<image-name>`) that the consumer fills in.

---

## Consumer snippets

### Bootstrap a new IaC repo

```bash
mkdir -p myrepo/.forgejo/workflows
cp <tcwlab>/templates/iac-ci.yml myrepo/.forgejo/workflows/ci.yml

# Then in ci.yml adjust the ADJUST markers:
#   env.OPENTOFU_VERSION     → current value from versions.yaml
#   env.BETTERLINT_VERSION   → current value from versions.yaml
#   permissions / branches   → project-specific
```

### Bootstrap a new service repo

```bash
cp <tcwlab>/templates/service-ci.yml myrepo/.forgejo/workflows/ci.yml
```

Typically also create a second file `.forgejo/workflows/release.yml` for the release path (semantic-release + build/push) — see the `service-ci.yml` header for the current pattern.

### Bootstrap a new tcwlab image repo

```bash
cp <tcwlab>/templates/docker-image-ci.yml myrepo/.forgejo/workflows/ci.yml
```

The template already includes the auto-tag + publish pattern we use for `tcwlab/<image>:<version>` releases.

---

## Known pain points / open issues

- **Drift between template and actual consumer workflow**: after copying, the consumer's workflow evolves independently. There is no drift detection. Idea: a comment marker in the consumer workflow showing the template version — unpopular to maintain, currently rejected.
- **Version drift in `env:` block**: each consumer has its own `BETTERLINT_VERSION` values. On major betterlint bumps, you must manually visit all consumers. Renovate could help later if we make `versions.yaml` the single source.
- **`docker-image-ci.yml` is the largest template** (~375 lines). Splitting into sub-templates was discussed, but: consumers want to read one workflow, not three. We keep the large template and document the job sections more clearly in the header.
- **Forgejo API incompatibilities**: Trivy PR description updates, semantic-release with GitHub plugin — both depend on Forgejo's GitHub compatibility layer. Forgejo major bumps are likely to break templates.
- **`workflow_dispatch` support**: all templates include the trigger because manual reruns in the Forgejo UI are practical. Consumer repos typically keep it.
