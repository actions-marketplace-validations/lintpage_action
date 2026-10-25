# LintPage Action

Audit SEO, accessibility, and performance on preview deployments before they reach production.

This GitHub Action runs [LintPage](https://lintpage.com) audits on your pages and surfaces issues as GitHub annotations directly in your pull requests.

## Inputs

| Input | Description | Required | Default |
|---|---|---|---|
| `urls` | URLs to audit, separated by spaces or newlines. Each URL is audited separately | No | Auto-detected from deployment event |
| `config` | Path to LintPage config file | No | `.lintpagerc.json` (auto-detected) |
| `fail-on-warning` | Fail the check on warnings (not just errors) | No | `false` |
| `quiet` | Minimal output (counts only) | No | `false` |
| `version` | `lintpage` CLI version to install. Pinned so a CLI release cannot change your results unannounced; set `latest` to always use the newest | No | `0.5.1` |

The `api-key` input from v1.0.0 has been removed. The CLI no longer accepts the `--report` flag it enabled, so setting it made every run fail; it is now ignored with a warning.

## Outputs

| Output | Description |
|---|---|
| `audited` | Number of URLs audited |
| `exit-code` | Worst CLI exit code across all URLs: `0` pass, `1` findings, `2` error |

## Usage

### With Vercel preview deployments

```yaml
name: LintPage
on: deployment_status

jobs:
  lintpage:
    if: github.event.deployment_status.state == 'success'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: lintpage/action@v1
```

### With explicit URLs

```yaml
name: LintPage
on: pull_request

jobs:
  lintpage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: lintpage/action@v1
        with:
          urls: 'https://staging.myapp.com https://staging.myapp.com/pricing'
```

### With local build

```yaml
name: LintPage
on: pull_request

jobs:
  lintpage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - run: npm ci
      - run: npm run build
      - run: npm start &
      - run: npx wait-on http://localhost:3000
      - uses: lintpage/action@v1
        with:
          urls: 'http://localhost:3000'
```

### With custom config

```yaml
- uses: lintpage/action@v1
  with:
    urls: 'https://staging.myapp.com'
    config: './seo/lintpage.config.json'
    fail-on-warning: 'true'
```

## How it works

1. Installs the `lintpage` CLI at the version set by `version`
2. Detects target URLs from the action input or from `deployment_status` events (works automatically with Vercel, Netlify, etc.), and fails with an error if there are none
3. Audits each URL with `--format github` so issues appear as annotations on your PR. Every URL is audited even if an earlier one fails
4. Exits with a non-zero code if any URL has critical issues (or warnings with `fail-on-warning`)

## License

MIT
