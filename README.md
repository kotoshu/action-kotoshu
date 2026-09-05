# action-kotoshu

GitHub Action that runs [Kotoshu](https://github.com/kotoshu/kotoshu) on
your repository and uploads the SARIF report to GitHub's Security tab.

## Usage

```yaml
- uses: kotoshu/action-kotoshu@v1
  with:
    files: "*.md docs/**/*.adoc"
    language: en
    format: sarif
```

## Inputs

| Name | Default | Purpose |
|---|---|---|
| `files` | `.` | Space-separated files or globs to check |
| `language` | `auto` | Language code or `auto` for detection |
| `format` | `sarif` | `text`, `json`, or `sarif` |
| `fail_on_error` | `true` | Fail the step if any errors are found |
| `prewarm_languages` | `en` | Languages to cache before the check |
| `offline` | `true` | Use only cached dictionaries |
| `version` | _(latest)_ | Pin kotoshu gem version |
| `output_path` | `kotoshu-results.sarif` | Where to write the report |

## Outputs

| Name | Description |
|---|---|
| `error-count` | Total spelling errors found |
| `sarif-path` | Path to the SARIF report (empty unless `format == sarif`) |

## How it works

1. Sets up Ruby 3.4 via `ruby/setup-ruby`.
2. Caches `~/.cache/kotoshu/` keyed on pre-warmed languages + OS.
3. Installs the `kotoshu` gem.
4. Pre-warms each language in `prewarm_languages` (idempotent — uses cache).
5. Runs `kotoshu check` on every file matched by `files`.
6. Combines per-file SARIF into one report.
7. Uploads SARIF to the GitHub Security tab via `codeql-action/upload-sarif`.
8. Uploads the report as an artifact.

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
      - uses: kotoshu/action-kotoshu@v1
        with:
          files: "*.md docs/**/*.adoc"
          language: en
```

## License

BSD-2-Clause, same as Kotoshu.


## Coming with the next gem release

The current published gem (0.7.0) supports the six original full-feature
languages. The next release (tracking `main`) brings:

- **19 full-feature languages** — `prewarm_languages: "it nl pl tr el uk ..."`
  will work once the setup gate widens.
- **Directory mode** — `files: "."` will walk the whole tree through the
  gem itself (respecting `.gitignore`, skipping `node_modules`/`vendor`),
  replacing the action's bash glob expansion.
- **Baselines** — freeze existing errors and fail only on new ones:

```yaml
- uses: kotoshu/action-kotoshu@v1
  with:
    files: "."
    baseline: .kotoshu-baseline.json
```

Generate the baseline with `kotoshu baseline init . > .kotoshu-baseline.json`
and commit it.
