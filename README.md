# UltraCoS — token-cost reduction for Claude Code

UltraCoS is a Claude Code plugin that lowers token cost across a session without
changing what the agent can see. It compacts tool-result output, removes repeated
content, and steers compaction toward a dense form — all **lossless by meaning**,
**fail-open** (any error passes the original through untouched), and fast (a
prebuilt native binary on the hot path, Python as the portable fallback).

It is free for noncommercial use (PolyForm Noncommercial 1.0.0); commercial use
needs a paid license — see [COMMERCIAL.md](COMMERCIAL.md).

## Install

```sh
claude plugin marketplace add MikkoParkkola/ultracos
claude plugin install ultracos
```

Hooks fire automatically on the next session. Run `ultracos-stats` to see savings,
`ultracos-set-level` to change aggressiveness.

## What it does (the wired features)

UltraCoS registers six hook points; every one fails open.

| Hook | What it does |
|---|---|
| **PostToolUse — codec** | Compacts each tool result: ANSI strip, JSON minify, blank-collapse, shape-aware compaction (JSON / YAML / TOML / code / filesystem path-lists), oversize truncation, schema-tag. Runs as the native binary, Python fallback. |
| **PostToolUse — session dedup** | A repeated `Read`/`Grep`/`Glob`/`Monitor` result is replaced with a short reference to its earlier occurrence in the session. |
| **PreToolUse — history dedup** | Collapses duplicate context already carried in earlier turns before a tool runs. |
| **PreCompact — summary-form mandate** | When Claude Code compacts, UltraCoS injects an instruction to summarize in a dense, structured form. |
| **UserPromptSubmit — mode detector + stats** | Detects the active aggressiveness level and serves the `ultracos-stats` view. |
| **SessionStart — skill loader** | Loads the UltraCoS mode skill so the agent understands the dense conventions. |

### Safety: it can reduce tokens but never corrupt context

Two guards make this safe:

- **Break-even gating** — a transform is applied only when it saves enough tokens
  to be worth its schema tag. Below that, the original passes through verbatim.
- **Anchor-survival guard** — a compaction that would drop the load-bearing
  `file:line`, error code, identifier, that made the output useful is automatically
  reverted. Truncation is the only lossy step, and it is anchor-guarded.

## The ULTRACOS-L1 dialect language

The codec binary also provides a **lossless prose↔dense transcoder** — the
ULTRACOS-L1 dialect — exposed as `ultracos-core compress` / `ultracos-core expand`.
It rewrites verbose, repetitive instruction-style prose into a denser dialect the
same model decodes natively, and expands it back exactly:

```
expand(compress(x)) == x      # byte-identical for dialect content;
                              # unrecognized text passes through untouched
```

It is round-trip lossless (verifiable: `compress | expand` reproduces the input
byte-for-byte) and the codec source documents a **44.6% token reduction on dialect
content** (`opus-dialect-validate-2026-05-31`). This is the language layer behind
the PreCompact dense-form mandate.

## Architecture — and why there are Python files

UltraCoS is **Rust-first, Python-fallback**:

- The hot-path codec ships as **prebuilt native binaries** under
  [`bin/<triple>/`](bin/) (macOS and Linux, arm64 and x86_64). The PostToolUse hook
  runs the binary by default — roughly `5 ms` per call versus `~170 ms` to launch a
  Python interpreter, with identical output.
- The **Python codec is the portable fallback** — used on an unsupported platform,
  a missing binary, an `exec` denied by policy, or `ULTRACOS_RUST=0`. So
  `hooks/PostToolUse/ultracos_codec.py` and the modules it imports (cache, dedup,
  anchor-guard, tokenizer, paths) exist so the plugin still works where the binary
  cannot run. Every path is fail-open.
- The **lightweight glue hooks** (skill loader, mode detector, stats handler,
  history dedup, PreCompact mandate) are Python because they are trivial and not on
  the per-tool-result hot path; a native port would buy nothing.

Binaries are reproducible from the in-repo source via [`bin/build.sh`](bin/build.sh)
and verified by [`bin/SHA256SUMS`](bin/SHA256SUMS). The codec source is fully open —
[`ultracos-core/`](ultracos-core/) — read every line.

## Configuration

| Env var | Default | Effect |
|---|---|---|
| `ULTRACOS_RUST` | on | Set `0` to force the Python codec. |
| `ULTRACOS_ANCHOR_GUARD` | on | Set `0`/`off` to disable the anchor-survival revert (not recommended). |
| `ULTRACOS_CACHE_AWARE` | off | When on, skips compacting content that is already a hot prompt-cache prefix, to keep the cache key stable. |
| `ULTRACOS_DATA_DIR` | `~/.ultracos` | Where the audit log and state live. |

## Calibration — a published snapshot, kept current as a service

The codec's keep-vs-compact boundary uses a token estimate. UltraCoS ships a
**calibration snapshot** ([`calibration/`](calibration/)): per-model
`tokens-per-char` values fitted from real, model-billed token counts, so the
estimate matches a model's actual tokenizer rather than a fixed assumption. The
fallback, when no snapshot value applies, is the classic 4-characters-per-token
estimate.

**Public vs private.** The codec source, the published snapshot (numbers, schema,
version), this methodology, and the fallback are all here and inspectable. The data,
the fitting method, and the pipeline that *produce* the snapshot are not — that is
what makes a snapshot a result you can use but not regenerate. See
[METHODOLOGY.md](METHODOLOGY.md).

**It is a service.** A model's tokenizer can change with a model update, with no
changelog. The snapshot is therefore refreshed as model tokenizers change. A frozen
copy keeps working under the license; a refreshed one tracks the change.

Every published value is fitted from measured counts. The project does not publish
performance figures it has not measured.

## Observability

UltraCoS writes an append-only audit row per compaction event (savings per tool,
shape, version) so its effect is measurable, not asserted. `ultracos-stats` reads it.

## License

**PolyForm Noncommercial License 1.0.0** — free for any noncommercial use.
Commercial use requires a paid license: see [COMMERCIAL.md](COMMERCIAL.md) or
contact **mikko.parkkola@iki.fi**. Full text in [LICENSE](LICENSE).
