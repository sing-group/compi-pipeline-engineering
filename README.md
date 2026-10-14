# Compi Pipeline Engineering Skill [![DOI](https://img.shields.io/badge/DOI-10.3389%2Ffbinf.2026.1824590-blue)](https://doi.org/10.3389/fbinf.2026.1824590)

An AI coding agent skill (compatible with [OpenCode](https://opencode.ai) and other agents supporting `SKILL.md`) for diagnosing, designing, refactoring, and verifying [Compi](https://www.sing-group.org/compi) bioinformatics pipelines based on the principles and recommendations presented in [*Software Engineering for Reproducible Pipeline Development in Bioinformatics*](https://doi.org/10.3389/fbinf.2026.1824590).

## Overview

This skill provides structured knowledge and workflows to help AI assistants engineer portable and reproducible Compi pipelines:

- **Compi Fundamentals**: Pipeline XML syntax (`<params>`, `<tasks>`, `<foreach>`, `<metadata>`), custom runners (`runner.xml`), parameter files, and CLI usage (`compi` and `compi-dk`).
- **SOLID Principles for Pipelines**:
  - **Single Responsibility (SRP)**: Separating compute tasks from glue tasks and splitting monolithic steps.
  - **Open/Closed (OCP)**: Extracting inline task code into external `src=` scripts and decoupling execution environments via runners.
  - **Liskov Substitution (LSP)**: Maintaining consistent output schemas across alternative tool implementations.
  - **Interface Segregation (ISP)**: Scoping parameters with `params=` instead of overusing `global="true"`.
  - **Dependency Inversion (DIP)**: Resolving format mismatches between tasks using dedicated adapter/glue tasks.
- **Robustness & Reproducibility**: Environment checks, input parameter validation (`IN`, `OR`, `NOT` constraints), runtime monitoring checks, container tag pinning, and reproducibility packs.
- **Project Layouts**: Support for both legacy (`pipeline.xml` at repository root) and new-style (`private/pipelines/main.xml`) project conventions.
- **Workflows**: Step-by-step **Diagnosis**, **Improvement**, and **Verification** workflows using `compi validate`, partial execution flags, and end-to-end tests.

## Installation

### Using `compi-dk` (Recommended)

Starting with Compi 1.7.0, you can install this skill directly using the `compi-dk get-compi-skills` command. It checks your current `compi-dk` version against [`versions.tsv`](versions.tsv) and downloads the compatible `SKILL.md` release into the specified directory (`-p` / `--path`).

#### Install for a specific project (OpenCode)

Run the following command from your pipeline repository root:

```bash
compi-dk get-compi-skills -p .opencode/skills/compi-pipeline-engineering
```

#### Install globally (OpenCode)

To make the skill available across all your projects:

```bash
compi-dk get-compi-skills -p ~/.config/opencode/skills/compi-pipeline-engineering
```

#### Command options

| Option | Short | Description |
|:-------|:------|:------------|
| `--path <path>` | `-p <path>` | Target directory where `SKILL.md` will be downloaded (created automatically if it does not exist). |

### Manual Installation

If you are using an older version of `compi-dk`, you can download `SKILL.md` directly into your agent's skill directory:

```bash
mkdir -p .opencode/skills/compi-pipeline-engineering
curl -fsSL https://raw.githubusercontent.com/sing-group/compi-pipeline-engineering/master/SKILL.md \
  -o .opencode/skills/compi-pipeline-engineering/SKILL.md
```

## Repository Structure

- [`SKILL.md`](SKILL.md): The complete skill definition, guidelines, and workflows.
- [`versions.tsv`](versions.tsv): Tab-separated mapping between Compi versions (`compi_version`) and skill release versions (`skills_version`), used by `compi-dk get-compi-skills` to resolve compatible releases.

## Related Links

- **Compi Homepage**: https://www.sing-group.org/compi
- **Compi Documentation**: https://www.sing-group.org/compi/docs
- **Compi Hub**: https://www.sing-group.org/compihub
- **Compi Source Code**: https://github.com/sing-group/compi

## Citing

The SOLID principles, robustness checks, and reproducibility recommendations implemented in this skill are presented in:

- D. Pérez-Rodríguez; A. Nogueira-Rodríguez; J. Vieira; C. Vieira; D. Glez-Peña; H. López-Fernández (2026) **Software Engineering for Reproducible Pipeline Development in Bioinformatics**. *Frontiers in Bioinformatics*. Volume 6 (2026), Article ID 1824590. ISSN: 2673-7647 [![DOI](https://img.shields.io/badge/DOI-10.3389%2Ffbinf.2026.1824590-blue)](https://doi.org/10.3389/fbinf.2026.1824590)

If you use Compi, please also cite:

- H. López-Fernández; O. Graña-Castro; A. Nogueira-Rodríguez; M. Reboiro-Jato; D. Glez-Peña (2021) **Compi: a Framework for Portable and Reproducible Pipelines**. *PeerJ Computer Science*. Volume 7: e593. ISSN: 2376-5992 [![DOI](https://img.shields.io/badge/DOI-10.7717%2Fpeerj--cs.593-blue)](https://doi.org/10.7717/peerj-cs.593)
