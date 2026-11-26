# caelo-action

Run [Caelo](https://caelo.sh) in your CI pipeline. Caelo is a free, local security
scanner for AI-built apps. This action runs it on your repository and fails the job
when it finds a problem, so risky code does not reach your main branch unnoticed.

Caelo looks for:

- Exposed API keys (Stripe, OpenAI, AWS, GitHub)
- Browser-leaked secrets (NEXT_PUBLIC and VITE variables)
- Supabase service-role keys and row-level-security gaps
- Open Firebase rules
- Committed .env files

The scan runs locally on the runner. It reads files only and uploads nothing.

Made by [Irth Labs](https://github.com/arthi-arumugam-git/caelo). Caelo is open
source under the MIT license.

## Usage

Add a workflow at `.github/workflows/security.yml`:

```yaml
name: Security

on:
  push:
    branches: [main]
  pull_request:

jobs:
  caelo:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: arthi-arumugam-git/caelo-action@v1
```

That is the whole setup. On every push to `main` and every pull request, Caelo
scans the checked-out code and fails the job if it finds exposed secrets or open
databases.

## Inputs

| Input          | Default | Description                                                                 |
| -------------- | ------- | --------------------------------------------------------------------------- |
| `path`         | `.`     | Folder to scan. Defaults to the repository root.                            |
| `fail-on`      | `any`   | When to fail the job. `any` fails on any finding. `never` scans and reports but keeps the job green. |
| `node-version` | `20`    | Node.js version used to run Caelo via npx.                                  |

### Scan a subfolder

```yaml
      - uses: arthi-arumugam-git/caelo-action@v1
        with:
          path: ./apps/web
```

### Report without failing the build

Use this to try Caelo on an existing repository and read the report first, before
you let it block merges.

```yaml
      - uses: arthi-arumugam-git/caelo-action@v1
        with:
          fail-on: never
```

## How it works

The action is a composite action. It sets up Node.js and then runs:

```
npx -y caelo <path>
```

Caelo prints a graded report and exits with a non-zero code when it finds a problem.
The action uses that exit code to decide whether the job passes or fails. When
`fail-on` is `never`, the action still runs the scan and prints the report, but it
always exits zero.

## Requirements

- A Linux runner (`ubuntu-latest` is used in the example above).
- Network access on the runner so `npx` can fetch the `caelo` package.

## Summary and assumptions

This action was created as a ready-to-publish wrapper around the `caelo` npm package.
It contains `action.yml` (a composite action) and this README.

What it does:

- Runs `npx -y caelo <path>` on the checked-out repository.
- Fails the CI job when Caelo exits non-zero, which is how Caelo signals findings.
- Exposes `path`, `fail-on`, and `node-version` inputs.
- Carries the branding block (shield icon, green color).

Assumptions worth checking before you publish:

- The action relies on Caelo's documented behavior: `npx caelo <path>` scans a folder
  and exits non-zero when it finds exposed secrets or open databases. It does not pass
  any severity-threshold flags to Caelo, because those flags were not confirmed at the
  time of writing. If Caelo adds documented flags for severity thresholds later, the
  `fail-on` input can be extended to map onto them instead of only switching on Caelo's
  exit code.
- `fail-on` is handled by the action itself, not by a Caelo flag. `any` propagates
  Caelo's exit code, and `never` always exits zero.
- `node-version` defaults to `20`. Adjust it if Caelo documents a different minimum
  supported Node.js version.
- The example workflow pins `caelo-action@v1`. Tag a `v1` release (and move the `v1`
  tag forward on patch releases) so that reference resolves.
