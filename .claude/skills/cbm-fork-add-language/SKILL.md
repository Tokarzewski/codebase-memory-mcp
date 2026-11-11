---
name: cbm-fork-add-language
description: Use when adding or fixing a tree-sitter language in the codebase-memory-mcp fork - vendoring a grammar, wiring CBM_LANG_*, lang_specs, extension or content sniffing, class synthesis, or when the language-count contract fails after a grammar change.
---

# Adding a language to codebase-memory-mcp

The worked example is the VB6/VBA series on `main` (refs upstream issue #721): `6db22aad` (sniffers), `2082b2d2` (grammar), `684cf775` (class synthesis), `9a7445e9` (tests). Before each step, read `git show --stat <hash>` for the matching commit; it lists every surface that step touched. CONTRIBUTING.md, "Adding or Fixing Language Support", is the upstream rule. Infrastructure formats (Dockerfile, k8s, Kustomize) follow its infra-pass pattern instead and get no grammar.

Work in the WSL clone, not the Windows checkout (skill `cbm-fork-dev-loop`).

## Steps, one commit each

1. **Routing first.** If the new extension is already claimed (`.frm` by FORM, `.cls` by Apex/ObjectScript), add a content sniffer in `src/discover/language.c` (`cbm_disambiguate_*`) that recognises the new language's header and keeps the old owner as the default on any doubt. Until the grammar lands, route such files to `CBM_LANG_COUNT` (unsupported), not to a wrong grammar: a wrong grammar gives junk nodes and extract timeouts. Tests go in `tests/test_language.c`.
2. **Vendor the grammar.** `scripts/vendor-grammar.sh <repo_url> <name> [subdir]` copies `parser.c`, `scanner.c` and the headers into `internal/cbm/vendored/grammars/<name>/`. Record the pinned commit, licence and ABI. Add an entry to `scripts/new-languages.json`, then use `python scripts/generate-lang-code.py` (modes `wrappers`, `enum`, `specs`, `language`) for the boilerplate: `grammar_<name>.c`, the `CBM_LANG_<X>` enum in `internal/cbm/cbm.h`, the `lang_specs.c` row and the `language.c` extension rows.
3. **Extraction.** Fill the spec in `internal/cbm/lang_specs.c` (callable, class, field, variable, call and import node types, module root). Language-specific shapes go in `extract_defs.c`, `extract_calls.c` and `extract_imports.c`; shared helpers go in `helpers.c`. When the grammar has no class node but the file is one class (a VB6 `.cls`), synthesise the Class in `cbm_extract_definitions` and set the scope walker's baseline `enclosing_class_qn` in `extract_unified.c`, so calls are sourced by the Method and never by the Module.
4. **Release surfaces.** The published count is the number of grammar directories. `tests/test_language_count_contract.sh` fails unless all of these agree: `internal/cbm/vendored/grammars/MANIFEST.md` (the "- Grammars: **N**" line and a row), `README.md`, `docs/index.html`, `docs/llms.txt`, `pkg/npm/README.md`, `pkg/chocolatey/codebase-memory-mcp.nuspec`, `scripts/package-release.sh` and `server.json`. Also update `THIRD_PARTY.md` and `scripts/vendored-checksums.txt`, which the vendored-integrity contract checks.
5. **Tests.** Cover extension and sniff mapping (`test_language.c`), the grammar regression and label golden (`test_grammar_regression.c`, `test_grammar_labels.c`), the calls contract (`test_lang_contract.c`), an extraction block (`test_extraction.c`) and the repro registries in `tests/repro/` (`repro_language_registry.c`, `repro_call_node_manifest.c`, `repro_invariant_breadth.c`, `repro_call_argument_matrix_b.c`). A language missing from a registry fails in that registry, not in your suite.

## Verify

```bash
scripts/test.sh --suites language,grammar_regression,grammar_labels,lang_contract,extraction
bash tests/test_language_count_contract.sh
scripts/test.sh --contracts-only
```

Then index a real repository in that language with the built binary and check the counts with `search_graph`. CONTRIBUTING asks for a check against a real open-source repo. Run the full `scripts/test.sh` before pushing.

## Rules

- A big grammar's `parser.c` is huge (vba's is about a million lines). Never open it whole; grep it.
- The `.codebase-memory.json` rule `grammars` makes any change under `internal/cbm/vendored/**` run the whole suite in test-impact selection. Expect that.
- Commits need `Signed-off-by` (`git commit -s`), which the `scripts/git-hooks/commit-msg` hook enforces, and conventional subjects such as `feat(lang): ...`, `feat(extract): ...` and `fix(discover): ...`.
- Upstream, new dependencies, MCP tool changes and new pipeline passes need maintainer approval. The fork can carry them, but say so if the change is meant to go upstream.
