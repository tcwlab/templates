# chameleon-ci/templates

Forgejo-Workflow-Templates als Konsumenten-API der [chameleon-ci](https://git.mon.k8b.co/chameleon-ci) Toolchain.

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

## Was ist das?

Drop-in-Workflow-Files. Konsumenten-Repos kopieren ein passendes Template nach `.forgejo/workflows/ci.yml` (statt es per `uses:` zu referenzieren) und passen die `# ── ANPASSEN ──`-Stellen am Anfang der Datei an. Drei Templates für die drei häufigen Repo-Typen in der TCW-Welt.

## Verfügbare Templates

| Template | Ziel-Repo-Typ | Verwendete Images |
| -------- | ------------- | ----------------- |
| [`iac-ci.yml`](iac-ci.yml) | OpenTofu / IaC | `tcwlab/betterlint`, `tcwlab/opentofu` |
| [`service-ci.yml`](service-ci.yml) | Kotlin/Micronaut/Go-Services | `tcwlab/betterlint` |
| [`docker-image-ci.yml`](docker-image-ci.yml) | Docker-Image-Wrapper (chameleon-ci-intern) | `tcwlab/betterlint`, `tcwlab/buildx`, `tcwlab/trivy`, `tcwlab/semantic-release` |

Alle drei folgen dem gemeinsamen Job-Skelett: **Lint → Build/Test → Security → Release/Publish**. Konsumenten löschen Jobs, die für ihren Pflegestand nicht relevant sind.

## Verwendung

### Bootstrap eines neuen IaC-Repos

```bash
mkdir -p myrepo/.forgejo/workflows
cp iac-ci.yml myrepo/.forgejo/workflows/ci.yml
```

Dann die `# ── ANPASSEN ──`-Stellen anpassen:

- `env.OPENTOFU_VERSION` und `env.BETTERLINT_VERSION` an den Stand aus [`chameleon-ci/versions.yaml`](https://git.mon.k8b.co/chameleon-ci/) angleichen.
- `permissions:`, `concurrency:` und `branches:` projekt-spezifisch.

### Bootstrap eines neuen Service-Repos

```bash
cp service-ci.yml myrepo/.forgejo/workflows/ci.yml
```

Plus typischerweise ein zweites File `release.yml` für den Release-Pfad (semantic-release + Build/Push) — siehe `service-ci.yml`-Header.

### Bootstrap eines neuen chameleon-ci-Image-Repos

```bash
cp docker-image-ci.yml myrepo/.forgejo/workflows/ci.yml
```

Das Template enthält bereits den Auto-Tag + Publish-Pattern, den wir für `tcwlab/<image>:<version>`-Veröffentlichungen verwenden, plus den Trivy-Scan-mit-PR-Markdown-Pattern.

## Versionsstrategie

Templates werden über **Git-Tags im `templates`-Repo** versioniert. Konsumenten kopieren physisch — sie referenzieren das Template nicht per `uses:`. Eine Bump im Template wirkt erst, wenn ein Konsument neu kopiert. Das ist bewusst — sonst würden Toolchain-Bumps die Konsumenten-Pipelines unangekündigt brechen.

Bei Major-Bumps in einem Template: Changelog-Eintrag im Forgejo-Release, der erklärt, was Konsumenten beim Re-Apply anpassen müssen.

## Was hier NICHT reingehört

- **`workflow_call`-Patterns** — Forgejo unterstützt das nicht reibungslos.
- **Repo-spezifische Logik** — Templates sind generisch.
- **Inline-Komplex-Logik**, die in eine Composite Action gehört — die wandert nach [`chameleon-ci/actions`](https://git.mon.k8b.co/chameleon-ci/actions).
- **Konsumenten-spezifische Image-Names** — nur generische `tcwlab/<toolname>`-Platzhalter.

## Lizenz

Apache-2.0 — The Chameleon Way.
