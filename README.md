# js-autofix-action

[GitHub Action](action.yaml) and [reusable workflow](.github/workflows/autofix-ci.yaml) to fix my JavaScript Action projects with [autofix.ci](https://autofix.ci/)

This is the JavaScript version of [go-autofix-action](https://github.com/suzuki-shunsuke/go-autofix-action).

1. Install prettier through [Takumi Guard](https://github.com/flatt-security/setup-takumi-guard-npm) with [download-npm-package-action](https://github.com/suzuki-shunsuke/download-npm-package-action) in a job with `id-token: write`
2. Fix the code in a job without `id-token: write`, and push the fix with autofix.ci
   - Update aqua-checksums.json (`aqua upc -prune`) in every directory where aqua-checksums.json exists
   - Format code with prettier
   - Fix newlines with [nllint](https://github.com/suzuki-shunsuke/nllint)
   - Update actions in Markdown files with [pinact](https://github.com/suzuki-shunsuke/pinact), except files under `.github`, which autofix.ci doesn't allow to change

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

## Use the action to generate files before fixing

If a project generates files (e.g. JSON Schema) from the code, call the action in your own workflow instead of the reusable workflow.
Check out the head commit of the pull request, generate files, and run the action with `checkout: "false"`.
The action formats the generated files and pushes them with the fix.

The workflow name must be `autofix.ci` and the workflow must be triggered by `pull_request`.
Separate jobs installing npm packages through Takumi Guard from the job running the action, which must not have `id-token: write`.

```yaml
jobs:
  npm-prettier:
    uses: suzuki-shunsuke/download-npm-package-action/.github/workflows/download.yaml@7b0c4cea0dbf5a58ec2b35c3c6432584369c1bfe # v0.0.2
    permissions:
      contents: read
      id-token: write
    with:
      ref: ${{github.event.pull_request.head.sha}}
      working_directory: .github/prettier
      artifact_name: node_modules-prettier
    secrets:
      TAKUMI_GUARD_BOT_ID: ${{secrets.TAKUMI_GUARD_BOT_ID}}
  npm:
    uses: suzuki-shunsuke/download-npm-package-action/.github/workflows/download.yaml@7b0c4cea0dbf5a58ec2b35c3c6432584369c1bfe # v0.0.2
    permissions:
      contents: read
      id-token: write
    with:
      ref: ${{github.event.pull_request.head.sha}}
      artifact_name: node_modules
    secrets:
      TAKUMI_GUARD_BOT_ID: ${{secrets.TAKUMI_GUARD_BOT_ID}}
  autofix:
    runs-on: ubuntu-26.04
    needs: [npm-prettier, npm]
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          ref: ${{github.event.pull_request.head.sha}}
          persist-credentials: false
      - uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0
        with:
          node-version-file: .node-version
      - uses: suzuki-shunsuke/download-npm-package-action/restore@7b0c4cea0dbf5a58ec2b35c3c6432584369c1bfe # v0.0.2
        with:
          artifact_name: node_modules
      - run: npm run schema
      - uses: suzuki-shunsuke/js-autofix-action@main
        with:
          aqua_version: v2.64.0
          checkout: "false"
          npm_artifact_name: node_modules-prettier
```

### Inputs of the action

- `aqua_version` (Required)
- `prettier_directory`: The directory where package.json and package-lock.json of prettier exist. The default is `.github/prettier`
- `npm_artifact_name`: The artifact name of prettier's node_modules. The default is `node_modules-autofix`
- `node-version-file`: The default is `.node-version`
- `checkout`: Whether the action checks out the head commit of the pull request. The default is `true`
