# LintPage Action

Audit SEO, accessibility, and performance on preview deployments before they reach production.

This GitHub Action runs [LintPage](https://lintpage.com) audits on your pages and reports issues as GitHub annotations on your pull request's checks.

## Inputs

| Input | Description | Required | Default |
|---|---|---|---|
| `urls` | URLs to audit, separated by spaces or newlines. Each URL is audited separately | No | The `deployment_status` target URL |
| `config` | Path to a LintPage config file | No | `.lintpagerc.json`, `.lintpagerc` or `lintpage.config.json` in the working directory |
| `fail-on-warning` | Fail the check on warnings (not just critical issues) | No | `false` |
| `version` | `lintpage` CLI version to install: an npm version or dist-tag. Pinned so a CLI release cannot change your results unannounced; set `latest` to always use the newest | No | `0.5.1` |

The config file is read from the checked-out repository, so run `actions/checkout` before this action. A missing or invalid config file falls back to the default rules without an error.

## Outputs

| Output | Description |
|---|---|
| `audited` | Number of URLs audited |
| `exit-code` | Worst result across all URLs: `0` pass, `1` findings, `2` error |

## Upgrading from v1.0.0

`@v1` now points at v1.1.0. Three changes can affect an existing workflow:

- Every URL in `urls` is audited. v1.0.0 audited only the first URL and silently skipped the rest, so a job that passed before can now fail on a URL that was never checked.
- The CLI is pinned to `0.5.1` instead of `latest`. To use a newer CLI, set `version`, or wait for an action release that raises the default.
- The `api-key` input has been removed. The CLI no longer accepts the `--report` flag it enabled, so setting it made every run fail. GitHub now ignores it with an "Unexpected input" warning; delete it from your workflow.

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

This runs for every successful deployment, production included. To audit previews only, also check `github.event.deployment.environment` in the `if:`.

If your previews use Vercel Deployment Protection, LintPage cannot get past it, so a protected preview cannot be audited. Turn off protection for previews, or audit a public URL with the `urls` input.

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
      - run: npm start > server.log 2>&1 &
      - run: npx wait-on -t 60000 http://localhost:3000
      - uses: lintpage/action@v1
        with:
          urls: 'http://localhost:3000'
```

Against `localhost`, LintPage skips the checks that only make sense for a public site and lowers some others to info. A notice in the output lists both. Other private addresses, such as a service container at `http://web:3000`, are refused.

### With custom config and a pinned CLI

```yaml
- uses: lintpage/action@v1
  with:
    urls: 'https://staging.myapp.com'
    config: './seo/lintpage.config.json'
    fail-on-warning: 'true'
    version: '0.5.1'
```

## How it works

1. Installs the `lintpage` CLI at the version set by `version`
2. Takes URLs from the `urls` input or, if it is empty, from the `deployment_status` event's target URL. Fails with an error if there is neither
3. Audits each URL with `--format github`, in its own log group. Every URL is audited even if an earlier one fails
4. Fails the step if any URL has critical issues (or warnings with `fail-on-warning`), or if the CLI errors. To report without blocking, set `continue-on-error: true` on the step and read the `exit-code` output

Findings appear as annotations in the run summary and on the pull request's Checks tab, not inline on the diff. A finding's annotation does not name its URL; the "LintPage Verdict" notice after each URL does. GitHub shows a limited number of annotations per step, so with many URLs or findings some are left out of the summary. The full output is always in the log.

## License

MIT
