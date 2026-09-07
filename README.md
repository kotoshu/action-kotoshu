# action-kotoshu

GitHub Action that runs [Kotoshu](https://github.com/kotoshu/kotoshu) on
your repository and uploads the SARIF report to GitHub's Security tab.

v2 delegates the file walk to the gem's directory mode (plan 88), so
`.gitignore`, hidden files, and the `.git`/`node_modules`/`vendor`/`target`
trees are honored by `kotoshu check` itself. v2 also wires baselines
(`kotoshu baseline init` / `kotoshu check --baseline`), the `--include`
and `--exclude` directory filters, and inline directive suppression.

## Usage

```yaml
- uses: kotoshu/action-kotoshu@v2
  with:
    files: .
    language: en
    format: sarif
```

## Inputs

| Name | Default | Purpose |
|---|---|---|
| `files` | `.` | Space-separated files, directories, or glob patterns. Directories are walked by the gem (gitignore-aware). |
| `language` | `auto` | Language code or `auto` for detection. |
| `format` | `sarif` | `text`, `json`, or `sarif`. |
| `baseline` | _(none)_ | Baseline JSON file. Errors it covers pass; only new errors fail. |
| `include` | _(none)_ | Directory mode only: check just files matching these globs (replaces the known-extension default). |
| `exclude` | _(none)_ | Directory mode only: skip files matching these globs (always wins over `include`). |
| `show_suppressed` | `false` | Also list entries suppressed by inline directives or the baseline (SARIF notes). |
| `fail_on_error` | `true` | Fail the step on failing spelling errors (with a baseline: only errors the baseline does not cover). |
| `category` | `kotoshu` | SARIF upload category, also used to scope the report artifact name. |
| `version` | _(latest)_ | Pin kotoshu gem version. Must be `>= 0.8.0` (directory mode + baselines). |
| `prewarm_languages` | `en` | Languages to cache before the check. |
| `offline` | `true` | Use only cached dictionaries. |
| `output_path` | `kotoshu-results.sarif` | Where to write the report. |

## Outputs

| Name | Description |
|---|---|
| `error-count` | Failing spelling errors (new errors only when a baseline is set). |
| `sarif-path` | Path to the SARIF/JSON report (empty when `format` is `text`). |

## Baselines

Baselines freeze existing misspellings so the check fails only on new
errors. They are wired through `kotoshu baseline init` and
`kotoshu check --baseline`.

```yaml
- uses: kotoshu/action-kotoshu@v2
  with:
    files: .
    language: en
    baseline: .kotoshu-baseline.json
```

`kotoshu baseline init` takes file targets (it reads each file once to
extract its line numbers) — directory targets are not supported. Run it
from the repository root with paths relative to the root, the same way
the action passes them to `kotoshu check`:

```bash
# Generate or refresh the baseline
kotoshu baseline init \
  README.md docs/guide.md docs/old-notes.md \
  --language en \
  --output .kotoshu-baseline.json

# Commit .kotoshu-baseline.json alongside the content it covers
git add .kotoshu-baseline.json README.md docs/
```

The default output filename is `.kotoshu-baseline.json` (see
`Kotoshu::Baseline::Store::DEFAULT_FILENAME`). Baseline entries are keyed
by `(file, word, count)` and a `count` larger than 1 absorbs repeats on
the same line.

In SARIF, entries the baseline covers are emitted as level `note` with a
`suppressions[].justification: "baseline"`. Entries the baseline does
not cover are emitted as level `warning` and fail the step.

## Ignoring words inline

Each format recognizes inline directives (see the gem README
`Ignoring Words Inline`):

```
<!-- kotoshu:disable-next-line -->
// kotoshu:disable-next-line
kotoshu:disable-line
```

Use these to suppress individual words or whole lines without touching
the baseline.

## How it works

1. Sets up Ruby 3.4 via `ruby/setup-ruby`.
2. Caches `~/.cache/kotoshu/` keyed on pre-warmed languages + OS.
3. Installs the `kotoshu` gem (defaults to latest; fails if `< 0.8.0`).
4. Pre-warms each language in `prewarm_languages`.
5. Expands glob-bearing tokens in `files`, passes the rest to
   `kotoshu check` directly so the gem walks directories.
6. Forwards `--include`, `--exclude`, `--baseline`, `--show-suppressed`
   when set; otherwise lets `kotoshu check` use its defaults.
7. Reads the failing error count from the report
   (`errorCount` for JSON; warnings-only count for SARIF; last-line
   summary for text).
8. Uploads SARIF to the GitHub Security tab via
   `codeql-action/upload-sarif` and uploads the report as an artifact.

Exit codes from `kotoshu check`:

| Code | Meaning | Step behavior |
|---|---|---|
| 0 | No errors | Success |
| 1 | Errors found | Fails only when `fail_on_error: true` |
| 2 | Usage error (bad flags, missing target, missing baseline) | Always fails |
| 3 | Language not set up (cache missing) | Always fails |

## Example: full CI workflow

```yaml
name: spell-check
on:
  push:
    branches: [main]
  pull_request:

jobs:
  kotoshu:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write   # required for SARIF upload
    steps:
      - uses: actions/checkout@v4
      - uses: kotoshu/action-kotoshu@v2
        with:
          files: .
          language: en
          baseline: .kotoshu-baseline.json
```

## Migrating from v1

| v1 | v2 | Why |
|---|---|---|
| `files: .` warned `No files matched` and ran nothing | `files: .` walks the whole tree (gitignore-aware) | v1 expanded a literal `.` to `[ -f . ]` and dropped it; the gem now does the walk |
| `kotoshu check` ran per file with JSON/SARIF concatenation in the wrapper | One `kotoshu check` invocation emits a combined document | Matches the gem directory mode (one run per file in SARIF, one `files[]` in JSON) |
| Hard errors (usage, missing language) were silently swallowed per file | Gem exit codes `2` and `3` fail the step | Surfaces real failures instead of zero-count success |
| `text` format produced nothing in the log | `text` is `tee`d into the report and shown in the step log | v1 captured per-file stdout and discarded it for text |
| Output paths for text were always empty | All formats write the report to `output_path` | Easier to attach the report to a workflow run |
| One `kotoshu-report` artifact per run | Artifact name `kotoshu-report-<category>` | Lets the same job call the action multiple times with distinct categories |

`fail_on_error` keeps the same meaning (the step fails when failing
spelling errors are found); under v2, with a baseline, "failing" means
errors the baseline does not cover. Personal-dictionary inputs are not
exposed yet — the gem `kotoshu check` does not currently consult the
personal dictionary in 0.9.x (it only feeds the `kotoshu personal`
subcommand). Use inline directives or the baseline for word-ignore lists
in the meantime.

## License

BSD-2-Clause, same as Kotoshu.
