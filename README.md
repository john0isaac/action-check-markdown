# action-check-markdown GitHub Action

An action that runs [markdown-checker](https://github.com/john0isaac/markdown-checker) on PRs and stores the found issues in the GitHub environment.

## Workflow setup

**Required input:**

- `command`: function that runs `markdown-checker` on the module you want to analyze. Available options are `check_broken_paths`, `check_paths_tracking`, `check_urls_tracking`, and `check_urls_locale`.
- `directory`: directory to run the function on. for example, `./`.
- `guide-url`: contribution guide full URL. for example, `https://github.com/john0isaac/action-check-markdown/blob/main/CONTRIBUTING.md`.
- `github-token`: for example, `${{secrets.GITHUB_TOKEN}}`.

It is necessary to include the following permissions in your job. See the example of a workflow setup below.

```(yaml)
permissions:
  pull-requests: write
  contents: read
```

**Example:**

```(yaml)
jobs:
  check-broken-paths:
    [...]
    permissions:
      pull-requests: write
      contents: read
    steps:
    - uses: actions/checkout@v3
    [...]
    - uses: john0isaac/action-check-markdown@v1.0.6
      with: 
        github-token: ${{secrets.GITHUB_TOKEN}}
        command: check_broken_paths
        directory: ./
        guide-url: 'https://github.com/john0isaac/action-check-markdown/blob/main/CONTRIBUTING.md'
```
