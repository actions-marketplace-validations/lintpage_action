# LintPage Action

Audit SEO, accessibility, and performance on preview deployments before they reach production.

This GitHub Action runs [LintPage](https://lintpage.com) audits on your pages and reports issues as GitHub annotations on your pull request's checks.

## Inputs

| Input | Description | Required | Default |
|---|---|---|---|
| `urls` | URLs to audit, separated by spaces or newlines. Each URL is audited separately | No | The `deployment_status` target URL |
| `config` | Path to a LintPage config file | No | `.lintpagerc.json`, `.lintpagerc` or `lintpage.config.json` in the working directory |
| `fail-on-warning` | Fail the check on warnings (not just critical issues) | No | `false` |
| `quiet` | Minimal output: the verdict and notices plus critical annotations (and warnings with `fail-on-warning`) | No | `false` |
| `vercel-bypass-secret` | Vercel Protection Bypass for Automation secret, passed from a GitHub secret. Sent only to the `*.vercel.app` URLs this action audits. See below | No | |
| `version` | `lintpage` CLI version to install: an npm version or dist-tag. Pinned so a CLI release cannot change your results unannounced; set `latest` to always use the newest | No | `0.5.5` |

The config file is read from the checked-out repository, so run `actions/checkout` before this action. A missing or invalid config file falls back to the default rules without an error.

## Outputs

| Output | Description |
|---|---|
| `audited` | Number of URLs audited. Skipped entries are not counted |
| `exit-code` | Worst result across all URLs: `0` pass, `1` findings, `2` error or a skipped entry |

## Upgrading

`@v1` points at the latest v1 release.

### v1.2.3

- The CLI is pinned to `0.5.5`. To keep the 0.5.4 results, set `version: '0.5.4'`.
- `cors-policy` reports `Access-Control-Allow-Origin: *` without credentials as info instead of a warning. Vercel (and Cloudflare Pages on `*.pages.dev`) send it on static pages by default, and without credentials it only lets other sites read what anyone can already fetch. With `fail-on-warning`, a page that failed only on this can now pass. A wildcard with credentials stays critical, and a server that reflects any origin stays a warning. Setting a severity for `cors-policy` in your config is not a way to keep the old warning: it applies to every `cors-policy` finding, so it would also turn the critical cases into warnings.
- The CLI package now carries the license notices for the code it bundles (undici and llhttp).

### v1.2.2

- The CLI is pinned to `0.5.4`. Two rules change, as below. To keep the 0.5.3 results, set `version: '0.5.3'`.
- `meta-render-blocking` counts only scripts a browser runs as classic JavaScript. Scripts held back by a consent manager (`type="text/plain"`) or moved to a worker by Partytown (`type="text/partytown"`) are no longer counted as blocking, nor are scripts browsers never run: a `type` with parameters such as `text/javascript; charset=utf-8`, a non-JavaScript `language` attribute, or an empty `src`. With `fail-on-warning`, a page that failed only on this rule can now pass.
- `tracking` reads only the Content-Security-Policy directives that let a page load or connect, such as `script-src`, `connect-src` and `default-src`. An analytics host named only in `report-uri`, `frame-ancestors` or `form-action` no longer counts, so a page whose only sign of analytics was such a host now gets the info `tracking` finding. If you raised `tracking` to a warning as described under v1.2.1, that page can now fail with `fail-on-warning`; set its severity back to `INFO` to avoid it.

### v1.2.1

- The CLI is pinned to `0.5.3`. Two rules report less, so with `fail-on-warning` a page that failed only on them can now pass. To keep the 0.5.2 results, set `version: '0.5.2'`.
- `tracking`: a page with no analytics visible in its HTML gets an info finding instead of a warning, since analytics loaded from a JavaScript bundle cannot be seen. An analytics host in a `preconnect`, `dns-prefetch`, `preload` or `modulepreload` link, or in a Content-Security-Policy header or meta tag, now counts as found. To keep the warning, set `"tracking": { "enabled": true, "severity": "WARNING" }` under `rules` in your config file.
- `meta-render-blocking` no longer counts a `<script nomodule>`, such as the Next.js polyfills, as blocking.

### v1.2.0

- The CLI is pinned to `0.5.2`. A page that redirects to another host, including a Vercel deployment that redirects to the Vercel login page, is now reported by the CLI as a critical `redirected-off-site` finding (exit code `1`), rather than refused by the action (exit code `2`). Such a URL now counts in `audited`.
- New `vercel-bypass-secret` input to audit deployments behind Vercel Deployment Protection. See [With Vercel preview deployments](#with-vercel-preview-deployments).
- On a `*.vercel.app` URL, the `X-Robots-Tag: noindex` Vercel adds to deployment URLs is reported as info, not as a critical issue.
- The `quiet` input is back and works.
- Annotations name the page they are about, show `:` and `,` as written, and each audited URL adds a section to the job summary.
- If you set `version` below `0.5.2`, a protected Vercel deployment can pass again by auditing the Vercel login page; v1.1.1 refused it in the action.

### v1.1.1

A `*.vercel.app` deployment behind Vercel Deployment Protection used to pass, because the audit checked the Vercel login page instead of your site. It now fails with an error that names the cause, and is not counted in `audited`. Nothing else changes.

### From v1.0.0 to v1.1.0

Three changes can affect an existing workflow:

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

#### Deployments behind Vercel Deployment Protection

If your project uses Deployment Protection, the deployment URLs Vercel reports redirect to the Vercel login page, for production deployments as well as previews. Without access, the audit fails with a critical `redirected-off-site` finding that names the cause. To audit the deployments:

1. In Vercel, open the project's **Settings**, then **Deployment Protection**, and create a **Protection Bypass for Automation** secret.
2. In GitHub, open the repository's **Settings**, then **Secrets and variables**, then **Actions**, and add it as a repository secret named `VERCEL_AUTOMATION_BYPASS_SECRET`.
3. Pass it to the action:

```yaml
      - uses: lintpage/action@v1
        with:
          vercel-bypass-secret: ${{ secrets.VERCEL_AUTOMATION_BYPASS_SECRET }}
```

The secret reaches the CLI through its environment, never on a command line, and neither the action nor the CLI prints it. Pass it from `secrets`, as above: GitHub masks secrets in logs, but a plain string or a `vars` value given as an input is shown in the step's log before the step runs. It is sent as the `x-vercel-protection-bypass` header only to the https `*.vercel.app` URLs this action audits (from `urls` or the deployment event), and only to that exact origin: not to other sites the audit contacts, and not along a redirect to another host. With the secret set, list only your own project's deployments in `urls`, since every audited `*.vercel.app` URL receives it.

Use the `vercel-bypass-secret` input rather than setting `VERCEL_AUTOMATION_BYPASS_SECRET` in the step's `env`: the CLI also needs the list of URLs it may send the secret to, which only the input sets up, so the environment variable alone sends nothing.

Vercel also sends `X-Robots-Tag: noindex` on its deployment URLs, so the CLI reports that noindex as info on a `*.vercel.app` URL, and says so. Audit your production domain to check its indexing. Previews may also get a `canonical-self-referencing` warning when their canonical tag points at the production domain, which matters if you set `fail-on-warning`.

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
    version: '0.5.5'
```

## How it works

1. Installs the `lintpage` CLI at the version set by `version`
2. Takes URLs from the `urls` input or, if it is empty, from the `deployment_status` event's target URL. Fails with an error if there is neither
3. Audits each URL with `--format github`, in its own log group. Every URL is audited even if an earlier one fails
4. Fails the step if any URL has critical issues (or warnings with `fail-on-warning`), or makes the CLI error. A redirect to another host, such as a Vercel login page, is a critical issue. To report without blocking, set `continue-on-error: true` on the step and read the `exit-code` output

Findings appear as annotations in the run summary and on the pull request's Checks tab, not inline on the diff. Each finding's annotation names the page it is about, and each URL's verdict comes first. GitHub shows a limited number of annotations per step, so with many URLs or findings some can be left out; each URL also adds a section to the job summary with its verdict, score, counts and top findings, and the full output is always in the log.

## License

MIT
