# templates — Repo-Kontext

> **Onboarding-Handshake:** Lies in dieser Reihenfolge:
>
> 1. [`Projects/CLAUDE.md`](https://git.mon.k8b.co/) (globale Standards)
> 2. [`tcwlab/CLAUDE.md`](https://git.mon.k8b.co/tcwlab/) (Toolchain-Kontext, Konsumenten-API)
> 3. Diese Datei (templates-spezifisches)

---

## Was ist `templates`?

`templates` ist das Repo, in dem tcwlab-eigene **Forgejo-Workflow-Templates** als drop-in-fähige YAML-Files leben. Konsumenten-Repos kopieren ein passendes Template nach `.forgejo/workflows/ci.yml` (statt es per `uses:` zu referenzieren) und passen die `# ── ANPASSEN ──`-Stellen am Anfang der Datei an. Das Repo bildet zusammen mit [`actions`](https://git.mon.k8b.co/tcwlab/actions) die **Konsumenten-API** der Toolchain.

Die bewusste Entscheidung gegen `uses:`-Referenzen für ganze Workflows kommt aus zwei Gründen: (1) Forgejo unterstützt aktuell nicht reibungslos Re-Use von ganzen Workflow-Files (im Gegensatz zu GitHub `workflow_call`); (2) wir wollen, dass Konsumenten den ganzen CI-Pfad ihres Repos sehen — nicht hinter einer `uses:`-Tür verstecken. Composite Actions in `actions/` decken den feinkörnigen Wiederverwendungs-Fall ab.

### Konsumenten

Alle TCW-Repos beim ersten Bootstrap. Pattern: `cp <tcwlab>/templates/<typ>-ci.yml <repo>/.forgejo/workflows/ci.yml`, dann ANPASSEN-Stellen ändern, dann commit.

Aktuelle Konsumenten in der TCW-Welt sind die Service-Verticals der TCW-Plattform (Atrium, Spectrum, Tally und weitere), die K8Box-IaC-Repos und die `tcwlab`-Image-Repos selbst.

---

## Was ist drin?

Drei Templates, je nach Repo-Typ:

| Template | Ziel-Repo-Typ | Image-Footprint |
| -------- | ------------- | --------------- |
| [`iac-ci.yml`](https://git.mon.k8b.co/tcwlab/templates/src/branch/main/iac-ci.yml) | OpenTofu / IaC | `tcwlab/betterlint`, `tcwlab/opentofu` |
| [`service-ci.yml`](https://git.mon.k8b.co/tcwlab/templates/src/branch/main/service-ci.yml) | Kotlin/Micronaut/Go-Services | `tcwlab/betterlint` (Build-Jobs als Platzhalter) |
| [`docker-image-ci.yml`](https://git.mon.k8b.co/tcwlab/templates/src/branch/main/docker-image-ci.yml) | Docker-Image-Wrapper (tcwlab-intern) | `tcwlab/betterlint`, `tcwlab/buildx`, `tcwlab/trivy`, `tcwlab/semantic-release` |

Alle drei sind heute auf das gemeinsame Job-Skelett (Lint → Build/Test → Security → Release/Publish) ausgerichtet. Konsumenten löschen Jobs, die für ihren Pflegestand nicht relevant sind — z.B. Service-Repos, die noch keinen Container-Push haben, lassen den Build-Push-Job weg.

---

## Was bei einem neuen Template zu tun ist

1. **PR auf `claude/feat-<template-typ>-template`**: neue YAML-Datei nach Konvention `<typ>-ci.yml`.
2. **Header-Block** mit Erklärung „Wofür?", „Wann verwenden?", „Welche Inputs muss das Konsumenten-Repo setzen?".
3. **`# ── ANPASSEN ──`-Marker** an allen Stellen, die ein Konsument vor dem ersten Push ändern muss (Image-Name, Branches, Secrets-Refs).
4. **`env:`-Block** mit allen Image-Versionen prominent oben — analog zum Pattern in den existierenden Templates.
5. **Dokumentation** in dieser Datei plus in [`templates/README.md`](https://git.mon.k8b.co/tcwlab/templates/src/branch/main/README.md) erweitern.
6. **Smoke-Test**: in einem Sandbox-Konsumenten-Repo das Template einmal komplett durchlaufen lassen, bevor es gemerged wird.

---

## Versionsstrategie

Templates werden über **Git-Tags im `templates`-Repo** versioniert (`v1.0.0`, `v1.1.0`). Konsumenten kopieren physisch — sie referenzieren das Template nicht per `uses:`. Das bedeutet:

- Eine Bump im Template wirkt erst, wenn ein Konsument neu kopiert.
- Das ist gewollt — sonst würden Toolchain-Bumps die Konsumenten-Pipelines unangekündigt brechen.
- Doku-Pflicht: bei Major-Bump in einem Template ein Changelog-Eintrag, der erklärt, was Konsumenten beim Re-Apply anpassen müssen.

semantic-release im `release`-Job des Repos erzeugt die Tags automatisch aus Conventional-Commits.

---

## Release-Verfahren

Self-hostend wie `actions`: das Repo verwendet `tcwlab/betterlint` + `tcwlab/semantic-release` für Lint und Release. Output: Forgejo-Tags + Forgejo-Releases mit Changelog. Kein Docker-Image als Output.

---

## Was explizit NICHT in dieses Repo gehört

- **`workflow_call`-Patterns**. Forgejo unterstützt das nicht reibungslos genug — wir bleiben bei drop-in-Copy.
- **Repo-spezifische Logik**. Templates sind generisch. Wenn ein Konsument was Spezielles braucht, ändert er das im eigenen Repo.
- **Inline-Komplex-Logik**, die in eine Composite Action gehört. Wenn ein Block in mehreren Templates auftaucht, gehört er nach [`actions/`](https://git.mon.k8b.co/tcwlab/actions).
- **Helm-/Kustomize-/Tofu-spezifische Module**. Templates beschreiben CI-Workflows, nicht IaC- oder Deployment-Inhalte.
- **Konsumenten-spezifische Image-Names** (z.B. `harbor.k8b.io/atrium/idp`). Templates verwenden generische Platzhalter (`tcwlab/<toolname>`, `<image-name>`), die der Konsument ersetzt.

---

## Konsumenten-Snippets

### Bootstrap eines neuen IaC-Repos

```bash
mkdir -p myrepo/.forgejo/workflows
cp <tcwlab>/templates/iac-ci.yml myrepo/.forgejo/workflows/ci.yml

# Dann in ci.yml die ANPASSEN-Stellen ändern:
#   env.OPENTOFU_VERSION     → versions.yaml-aktueller Wert
#   env.BETTERLINT_VERSION   → versions.yaml-aktueller Wert
#   permissions / branches   → projektspezifisch
```

### Bootstrap eines neuen Service-Repos

```bash
cp <tcwlab>/templates/service-ci.yml myrepo/.forgejo/workflows/ci.yml
```

Plus typischerweise ein zweites File `.forgejo/workflows/release.yml`, das den Release-Pfad (semantic-release + Build/Push) abbildet — siehe `service-ci.yml`-Header für das aktuelle Pattern.

### Bootstrap eines neuen tcwlab-Image-Repos

```bash
cp <tcwlab>/templates/docker-image-ci.yml myrepo/.forgejo/workflows/ci.yml
```

Das Template enthält bereits den Auto-Tag-+-Publish-Pattern, den wir für `tcwlab/<image>:<version>`-Veröffentlichung verwenden.

---

## Bekannte Schmerzpunkte / offene Themen

- **Drift zwischen Template und tatsächlichem Konsumenten-Workflow**: nach Drop-in-Copy entwickelt sich der Konsument unabhängig weiter. Es gibt keine Drift-Detection. Idee: ein Komment-Marker im Konsumenten-Workflow, der die Template-Version zeigt — pflegerisch unattraktiv, aktuell verworfen.
- **Versions-Drift in `env:`-Block**: jeder Konsument hat seine eigenen `BETTERLINT_VERSION`-Werte. Bei Major-Bump in `betterlint` muss man manuell durch alle Konsumenten gehen. Renovate könnte hier später helfen, wenn `versions.yaml` als Single-Source eingebunden wird.
- **`docker-image-ci.yml` ist mit Abstand das umfangreichste Template** (~375 Zeilen). Spaltung in Sub-Templates wurde diskutiert, aber: Konsumenten wollen einen Workflow lesen, nicht drei. Wir behalten das große Template, dokumentieren aber die Job-Sektionen klarer im Header-Block.
- **Forgejo-API-Inkompatibilitäten**: Trivy-PR-Description-Update, Semantic-Release-mit-GitHub-Plugin — beides hängt an Forgejos GitHub-Kompatibilitäts-Layer. Bei Forgejo-Major-Bumps sind Templates die wahrscheinlichsten Bruch-Stellen.
- **`workflow_dispatch`-Support**: alle Templates haben den Trigger drin, weil manuelles Re-Run im Forgejo-UI praktisch ist. Konsumenten-Repos lassen ihn zumeist drin.
