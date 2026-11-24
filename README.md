# dev-platform

Design docs for a small GitHub-native development platform: one place to define CI, review, and planning conventions that every new project inherits.

| Repo | Role |
|------|------|
| [.github](https://github.com/akhiljayaraj/.github) | Reusable CI and CodeQL workflows, community-health defaults, labels |
| [template-app](https://github.com/akhiljayaraj/template-app) | Template with thin workflow callers and a bootstrap script |

Projects pin workflows at `@v1`, so a pipeline change rolls out by moving one tag.

- [Design spec](docs/superpowers/specs/2026-08-18-dev-platform-design.md)
- [Implementation plan](docs/superpowers/plans/2026-08-18-dev-platform.md)
