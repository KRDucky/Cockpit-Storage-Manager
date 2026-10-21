# Repository Instructions for Claude Code
Read docs/PRODUCT_SPEC.md, docs/ARCHITECTURE.md, docs/SAFETY_AND_TESTING.md, and docs/REFERENCES.md before work, then docs/DECISIONS.md and the accepted ADRs in docs/adr/. Follow the project's Claude Project Instructions. If a file is missing, report it and do not invent requirements.

Source of truth (ADR-0009): this repository's docs/ directory is authoritative. Claude Project Knowledge is contextual reference only; when it conflicts with the repository, follow the current repository documentation and accepted ADRs. Product name: cockpit-storage-manager; internal prefix csm-.

Never run filesystem-destructive commands on a real host. Only a disposable VM or clearly identified loopback fixture may receive formatting/partitioning/recovery tests. Ask for explicit milestone authorization before writing code beyond the assigned milestone. Keep docs/STATUS.md, docs/DECISIONS.md, docs/SAFETY_CASES.md up to date. Do not publish releases or modify unrelated services.

Before code, state plan, risk and tests. After code, report files changed, real test results, gaps and next milestone. Typed API, validated plan engine, strict helper privileges and idempotent import required. No shell string construction or unvalidated root commands. No automatic SnapRAID sync in failure or recovery workflows.
