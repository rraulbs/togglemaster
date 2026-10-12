# togglemaster
Feature flag platform evolving from a monolith to a distributed cloud-native system on AWS. FIAP POSTECH DevOps &amp; Cloud Architecture Tech Challenge.

# ToggleMaster

Feature flag platform that evolves, phase by phase, from a simple monolithic
MVP into a distributed, resilient and highly automated cloud system on AWS.

This repository is part of the **Tech Challenge** of the postgraduate program
in **DevOps & Cloud Architecture** (FIAP POSTECH).

## About the project

ToggleMaster lets development teams release new features in a controlled way,
enabling or disabling them for specific users without a new deploy.

The base application is a Flask + PostgreSQL monolith, originally provided at
[dougls/toggle-master-monolith](https://github.com/dougls/toggle-master-monolith).

## Phases

| Phase | Theme | Status | Folder |
| ----- | ----- | ------ | ------ |
| 1 | Monolith on AWS (VPC, EC2, RDS) | In progress | [phase-1-monolith-aws](./phase-1-monolith-aws) |
| 2 | TBD | Not started | - |
| 3 | TBD | Not started | - |
| 4 | TBD | Not started | - |

## Repository structure
The repository keeps the original application as a baseline, and each phase is self-contained:
  - `toggle-master-monolith/`: original base application (Flask + PostgreSQL), kept unmodified as a reference.
  - `phase-N/`: documentation, infrastructure files and, when applicable, application code for each phase.

## Author

- Raul ([@rraulbs](https://github.com/rraulbs))