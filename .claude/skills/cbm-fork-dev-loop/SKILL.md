---
name: cbm-fork-dev-loop
description: Use when building, testing, linting or committing in the codebase-memory-mcp fork (Tokarzewski/codebase-memory-mcp), merging upstream DeusData into it, or when a test, lint or hook run fails on the Windows checkout.
---

# Developing the codebase-memory-mcp fork

The global skill `codebase-memory-tooling` covers operating the installed tool (indexing, queries, the graph UI). This skill covers changing the C source.

## Where to work

- The Windows checkout is `C:\Github\Tokarzewski\codebase-memory-mcp`. Older notes name `C:\Github\codebase-memory-mcp`, which no longer exists. The workspace rule is to build Linux-only repos in WSL. The build needs gcc or clang, make and zlib.
- There is no WSL clone yet. Make one in the `Ubuntu` distro at `~/github/codebase-memory-mcp` (`git clone https://github.com/Tokarzewski/codebase-memory-mcp.git`, then `git remote add upstream https://github.com/DeusData/codebase-memory-mcp.git`). Run `scripts/setup.sh`, or `git config core.hooksPath scripts/hooks`, there. Run commands with `wsl -d Ubuntu --cd ~/github/codebase-memory-mcp -- bash -lc "<cmd>"`.
- `scripts/setup-windows.ps1` and `scripts/test-windows.ps1` exist for a native Windows (MSYS2) build. Use them only when the change is Windows-specific.

## Commands (each answers `--help`)

| Need | Command |
|---|---|
| Production build | `scripts/build.sh` (output `build/c/codebase-memory-mcp`; `--with-ui` adds the graph UI) |
| Iterate on a change | `scripts/test.sh --suites <a,b>` (incremental ASan+UBSan, seconds); get suite names from `build/c/test-runner --list-suites` |
| Static contracts only | `scripts/test.sh --contracts-only` (no compiler) |
| Merge gate | `scripts/test.sh` (clean build, every suite, contracts, prod-binary guards) |
| Concurrency change | `scripts/test.sh --tsan` |
| Lint | `scripts/lint.sh` (`--ci` skips clang-tidy) |
| Security layers | `make -f Makefile.cbm security` |

Exit codes: 0 is a pass, 2 a usage error, and 90 a guard (the run died without its summary, so it never counts as green).

## Rules

- Run heavy legs one at a time, in the background. The full `scripts/test.sh` is long. Check free RAM first (skill `heavy-job-resource-check`).
- The pre-commit hook (`scripts/hooks/pre-commit`) runs lint, the source security audit, a build and the full suite. It clears git's local env vars before the tests because the suites create their own repositories. Do the same in any script that runs git inside tests.
- Every commit needs `Signed-off-by` (`git commit -s`), enforced by `scripts/git-hooks/commit-msg` and the `dco.yml` workflow. Subjects are conventional commits (`feat(lang):`, `fix(discover):`, `test(extract):`).
- Upstream syncs are merges, not rebases: `git fetch upstream`, then `git merge upstream/main`. Push only to `origin`. Never open a PR against DeusData unless the owner asks.
- The fork's own work so far is the VB6 language series. To add a language, use skill `cbm-fork-add-language`.
- If a leg is red in CI but green locally, suspect the environment first (`scripts/README.md`, "Recommended workflows").
- Rebuilding does not change the installed binary (`codebase-memory-mcp --version`) until you reinstall it. Say which binary a result came from.
