# js-autofix-action

[GitHub Action](action.yaml) and [reusable workflow](.github/workflows/autofix-ci.yaml) to fix my JavaScript Action projects with [autofix.ci](https://autofix.ci/)

This is the JavaScript version of [go-autofix-action](https://github.com/suzuki-shunsuke/go-autofix-action).

1. Install prettier through [Takumi Guard](https://github.com/flatt-security/setup-takumi-guard-npm) with [download-npm-package-action](https://github.com/suzuki-shunsuke/download-npm-package-action) in a job with `id-token: write`
2. Fix the code in a job without `id-token: write`, and push the fix with autofix.ci
   - Update aqua-checksums.json (`aqua upc -prune`) in every directory where aqua-checksums.json exists
   - Format code with prettier
   - Fix newlines with [nllint](https://github.com/suzuki-shunsuke/nllint)
   - Update actions in Markdown files with [pinact](https://github.com/suzuki-shunsuke/pinact) (skipped for pull requests from forks)

## How to use

1. Put package.json and package-lock.json that depend on only prettier in `.github/prettier`, so the workflow doesn't install all the dependencies of the project

```json
{
  "private": true,
  "devDependencies": {
    "prettier": "3.9.10"
  }
}
```

2. Call the reusable workflow. The workflow name must be `autofix.ci`, which autofix.ci requires

```yaml
---
name: autofix.ci
on: pull_request
permissions: {}
jobs:
  fix:
    uses: suzuki-shunsuke/js-autofix-action/.github/workflows/autofix-ci.yaml@main
    permissions:
      contents: read
      id-token: write # For Takumi Guard
    with:
      aqua_version: v2.64.0
    secrets:
      TAKUMI_GUARD_BOT_ID: ${{secrets.TAKUMI_GUARD_BOT_ID}}
  autofix:
    # Register this job as a required status check
    runs-on: ubuntu-26.04
    if: always() && (contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled'))
    timeout-minutes: 10
    permissions: {}
    needs:
      - fix
    steps:
      - run: exit 1
```

### Inputs

- `aqua_version` (Required)
- `prettier_directory`: The directory where package.json and package-lock.json of prettier exist. The default is `.github/prettier`
- `node-version-file`: The default is `.node-version`
