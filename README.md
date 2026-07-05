# action-check-markdown

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Check_Markdown-blue?logo=github)](https://github.com/marketplace/actions/check-markdown)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Powered by markdown-checker](https://img.shields.io/pypi/v/markdown-checker?label=markdown-checker)](https://pypi.org/project/markdown-checker/)

A GitHub Action that wraps [markdown-checker](https://github.com/john0isaac/markdown-checker) to validate `.md` and `.ipynb` files on your pull requests, then posts a Markdown report as a PR comment and job summary.

## Features

- Five checks: broken relative paths (`check_broken_paths`), broken web
  URLs (`check_broken_urls`), URLs missing a tracking ID
  (`check_urls_tracking`), paths missing a tracking ID
  (`check_paths_tracking`), and URLs containing a locale segment
  (`check_urls_locale`).
- Full control over URL-check behavior: timeouts, retries, 429/rate-limit
  handling, concurrency, and per-host request pacing.
- Skip specific files, domains, or URL substrings from any check.
- Choose the report format (`markdown`, `json`, `github-annotations`,
  `console`) and, optionally, load defaults from a `pyproject.toml`
  configuration file.
- On failure, posts the report as both a job summary and a pull request
  comment, and fails the workflow run.
- Keeps a single sticky comment per check up to date across runs instead of
  piling up new comments, and updates it to a success message once issues
  are resolved.

## Tutorial: get started

Add a workflow file such as `.github/workflows/check-markdown.yml`,
replacing `guide-url` with the URL of your own contribution guide:

```yaml
name: Check Markdown

on:
  pull_request:
    paths:
      - "**.md"
      - "**.ipynb"

permissions:
  contents: read

jobs:
  check-broken-paths:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: john0isaac/action-check-markdown@v1.3.0
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          command: check_broken_paths
          directory: ./
          guide-url: 'https://github.com/<owner>/<repo>/blob/main/CONTRIBUTING.md'
```

The `pull-requests: write` permission is required so the action can post
its findings as a comment on the pull request; `contents: read` is
required so `actions/checkout` can clone the repository. Open a pull
request that touches a checked file to see the report get posted.

## How-to guides

### Check for broken external URLs

Set `command: check_broken_urls` to flag web links that don't return a
successful response:

```yaml
- uses: john0isaac/action-check-markdown@v1.3.0
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    command: check_broken_urls
    directory: ./
    guide-url: 'https://github.com/<owner>/<repo>/blob/main/CONTRIBUTING.md'
```

### Require tracking IDs on links

Use `check_urls_tracking` to flag URLs on configured domains that are
missing a `wt.mc_id` query parameter, or `check_paths_tracking` to require
it on every relative path link, regardless of domain:

```yaml
- uses: john0isaac/action-check-markdown@v1.3.0
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    command: check_urls_tracking
    directory: ./
    tracking-domains: 'github.com,microsoft.com,learn.microsoft.com'
    guide-url: 'https://github.com/<owner>/<repo>/blob/main/CONTRIBUTING.md'
```

### Flag locale-specific URLs

Use `check_urls_locale` to flag URLs whose path contains a language-country
segment (e.g. `/en-us/`), which usually shouldn't be hardcoded in docs:

```yaml
- uses: john0isaac/action-check-markdown@v1.3.0
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    command: check_urls_locale
    directory: ./
    guide-url: 'https://github.com/<owner>/<repo>/blob/main/CONTRIBUTING.md'
```

### Run more than one check

Each run of the action performs a single check, selected with `command`.
Add one step (or job) per check to run several in the same workflow:

```yaml
steps:
  - uses: actions/checkout@v4
  - name: Check broken paths
    uses: john0isaac/action-check-markdown@v1.3.0
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}
      command: check_broken_paths
      directory: ./
      guide-url: 'https://github.com/<owner>/<repo>/blob/main/CONTRIBUTING.md'
  - name: Check broken URLs
    uses: john0isaac/action-check-markdown@v1.3.0
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}
      command: check_broken_urls
      directory: ./
      guide-url: 'https://github.com/<owner>/<repo>/blob/main/CONTRIBUTING.md'
```

### Limit which files are scanned

Use `directory` to point at a subfolder, and `extensions` to restrict which
file types are checked (default `.md,.ipynb`):

```yaml
- uses: john0isaac/action-check-markdown@v1.3.0
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    command: check_broken_paths
    directory: ./docs
    extensions: '.md'
    guide-url: 'https://github.com/<owner>/<repo>/blob/main/CONTRIBUTING.md'
```

### Skip specific files, domains, or URLs

- `skip-files` excludes file names from every check (default
  `CODE_OF_CONDUCT.md,SECURITY.md`).
- `skip-domains` and `skip-urls-containing` exclude hosts or URL
  substrings from URL-based checks (`check_broken_urls`,
  `check_urls_tracking`, `check_urls_locale`).

```yaml
- uses: john0isaac/action-check-markdown@v1.3.0
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    command: check_broken_urls
    directory: ./
    skip-files: 'CODE_OF_CONDUCT.md,SECURITY.md,CHANGELOG.md'
    skip-domains: 'localhost,example.com'
    skip-urls-containing: '/embed/,/preview/'
    guide-url: 'https://github.com/<owner>/<repo>/blob/main/CONTRIBUTING.md'
```

### Tune timeouts, retries, and rate limiting

`check_broken_urls` makes real HTTP requests, so large repositories may
need to tune concurrency and retry behavior to avoid tripping host rate
limits:

```yaml
- uses: john0isaac/action-check-markdown@v1.3.0
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    command: check_broken_urls
    directory: ./
    timeout: '20'
    retries: '3'
    retry-on-429: true
    fallback-retry-delay: '30'
    max-workers: '10'
    per-host-delay: '0.5'
    guide-url: 'https://github.com/<owner>/<repo>/blob/main/CONTRIBUTING.md'
```

### Use a `pyproject.toml` configuration file

Instead of repeating options on every step, set them once in a
`[tool.markdown-checker]` table and point `config` at that file (or omit
`config` to let it discover the nearest `pyproject.toml` automatically).
Use `isolated: true` to ignore configuration discovery entirely for a run:

```yaml
- uses: john0isaac/action-check-markdown@v1.3.0
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    command: check_broken_paths
    directory: ./
    config: ./pyproject.toml
    guide-url: 'https://github.com/<owner>/<repo>/blob/main/CONTRIBUTING.md'
```

### Choose a report format

`report-format` controls both the console output style and the file
written when errors are found (`markdown` when not set here or in
`pyproject.toml`):

```yaml
- uses: john0isaac/action-check-markdown@v1.3.0
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    command: check_broken_paths
    directory: ./
    report-format: json
    guide-url: 'https://github.com/<owner>/<repo>/blob/main/CONTRIBUTING.md'
```

## Reference

### Inputs

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `command` | Yes | `check_broken_paths` | Check to run: `check_broken_paths`, `check_broken_urls`, `check_paths_tracking`, `check_urls_tracking`, `check_urls_locale`. |
| `directory` | Yes | `./` | Directory to search recursively for matching files. |
| `guide-url` | No | *(unset)* | Full URL of your contributing guide, linked in the report. |
| `github-token` | Yes | `''` | GitHub token used to post the PR comment, e.g. `${{ secrets.GITHUB_TOKEN }}`. |
| `extensions` | No | *(unset:* `.md,.ipynb`*)* | Comma-separated file extensions to check. |
| `tracking-domains` | No | *(unset:* `github.com,microsoft.com,visualstudio.com,aka.ms,azure.com`*)* | Comma-separated hostnames that must carry a `wt.mc_id` parameter. Only affects `check_urls_tracking`. |
| `skip-files` | No | *(unset:* `CODE_OF_CONDUCT.md,SECURITY.md`*)* | Comma-separated file names to exclude from checking. |
| `skip-domains` | No | *(unset)* | Comma-separated domains to exclude from URL checks. |
| `skip-urls-containing` | No | *(unset)* | Comma-separated URL substrings to exclude from URL checks. |
| `timeout` | No | *(unset:* `20`*)* | Per-request timeout in seconds for URL checks (`0`-`50`). |
| `retries` | No | *(unset:* `3`*)* | Number of attempts before a URL is reported as broken (`0`-`10`). |
| `retry-on-429` | No | *(unset:* `true`*)* | Whether a 429 response is retried honouring `Retry-After` instead of reported immediately as rate-limited. |
| `fallback-retry-delay` | No | *(unset:* `30`*)* | Seconds reported as the retry delay when a 429 carries no `Retry-After` header (`0`-`300`). |
| `max-workers` | No | *(unset: available CPUs)* | Maximum number of concurrent URL-check worker threads. |
| `per-host-delay` | No | *(unset:* `0.5`*)* | Minimum delay in seconds enforced between two requests to the same host (`0.0`-`10.0`). |
| `report-format` | No | *(unset:* `markdown`*)* | Report format: `console`, `github-annotations`, `json`, `markdown`. |
| `config` | No | `''` | Path to a TOML file to read `[tool.markdown-checker]` configuration from, instead of discovering `pyproject.toml`. |
| `isolated` | No | `false` | Ignore `pyproject.toml` configuration discovery entirely for this run. |

Inputs marked *(unset)* are not passed to the CLI unless you set them.
For those, the effective value is resolved in this order: explicit
workflow input, then `[tool.markdown-checker]` in `pyproject.toml` (or the
file given by `config`), then the tool default shown after the colon. This
means your `pyproject.toml` settings are honored unless a workflow input
explicitly overrides them.

### Permissions

```yaml
permissions:
  pull-requests: write
  contents: read
```

`pull-requests: write` lets the action comment on the pull request;
`contents: read` lets `actions/checkout` clone the repository.

### Outputs

This action defines no outputs. Its result is communicated through the
job summary, a pull request comment, and the workflow run's exit status
(non-zero when errors are found).

## Explanation

### How it works

The action installs `markdown-checker` from PyPI, then invokes its CLI
with the inputs above, always writing the report under a randomly
generated file name so concurrent jobs don't collide. If the check finds
error-level issues, the report is appended to `$GITHUB_STEP_SUMMARY` and
the job is made to fail. Warning-level issues (e.g. rate-limited or
unverifiable URLs) are reported but never fail the run.

On pull requests, the action also syncs the result to a single sticky
comment via the GitHub API (`gh api`). Each comment is tagged with a
hidden marker keyed by `command` (e.g. `check_broken_urls`), so running
multiple checks in one workflow keeps one comment per check instead of
them overwriting each other, and comments from unrelated workflows are
never touched. When error-level issues are found, that comment is created
or updated with the report; once they're resolved, the same comment is
updated to a short success message. If no issues are found and no comment
exists yet, nothing is posted, keeping clean pull requests uncluttered.

On pull requests from forks, the default `GITHUB_TOKEN` is read-only, so
the action skips the comment sync step with a `::notice::` explaining why,
instead of failing on a confusing 403; the job summary and exit status
still reflect the check result. Use `pull_request_target` (carefully) if
you need comments on fork PRs, or `report-format: github-annotations` for
inline annotations that work with a read-only token.

### Why `report-format` changes the output file's extension

Each report format has a corresponding renderer with its own file
extension - `.md` for `markdown`, `.json` for `json`, and `.txt` for
`console`/`github-annotations`. The action detects which file the renderer
actually wrote (whether the format came from an input, `pyproject.toml`,
or the default) so the job summary and PR comment steps always read back
the right one.

### Relationship to the markdown-checker CLI

Every input mirrors a `markdown-checker` CLI option 1:1; see the
[markdown-checker documentation](https://markdown-checker.readthedocs.io/en/latest/)
for the full behavior of each check, and specifically
[How to Run markdown-checker in GitHub Actions](https://markdown-checker.readthedocs.io/en/latest/howto/github-actions/)
for how this action fits into the wider tool.

## Contributing

Contributions are welcome - see the [Contributing guide](CONTRIBUTING.md)
and [Code of Conduct](CODE_OF_CONDUCT.md).

## License

[MIT](LICENSE)
