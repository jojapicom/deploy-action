# Deploy to JoJ API

Deploy a [JoJ API](https://jojapi.com) Worker from GitHub Actions. Every pull request gets a
[preview deployment](https://docs.jojapi.com/studio/deployments) on its own URL, commented on the
pull request; every push to the default branch goes to production. Each deployment records its
commit, branch and pull request, so the Studio's **Deployments** tab links to the code it runs.

```yaml
- uses: jojapicom/deploy-action@v1
  with:
    token: ${{ secrets.JOJAPI_TOKEN }}
```

## Set up

1. **Put the Worker in the repository.** `npx @jojapi/cli pull <your-api>` writes the files and a
   `jojapi.json` that names the API. Commit both. Files under `src/` and npm dependencies are
   bundled by the CLI.
2. **Create a management token** in JoJ API's **Settings › Tokens** (Settings is in the user
   menu, top right). Use the **CI: production deploys** template (`code:read`, `code:write`,
   `variables:write`, `resources:write`, `deployments:promote`) and limit the token to the APIs
   this repository deploys. If you would rather promote to production yourself, use
   **CI: preview deploys** and run the workflow on pull requests only. A token created earlier
   with `code:read` and `code:write` keeps deploying to production as before.
3. **Add it to the repository** as the secret `JOJAPI_TOKEN` (**Settings → Secrets and variables
   → Actions**), by pasting the value. `gh secret set JOJAPI_TOKEN` works too when you paste at its
   prompt: piping an unset shell variable into it stores an empty secret without an error, and the
   action then fails with "The token input is empty".
4. **Add the workflow** as `.github/workflows/jojapi.yml`:

```yaml
name: Deploy to JoJ API

on:
  push:
    branches: [main]
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write # the preview URL comment
    concurrency: jojapi-${{ github.ref }}
    steps:
      - uses: actions/checkout@v7
      - uses: jojapicom/deploy-action@v1
        with:
          token: ${{ secrets.JOJAPI_TOKEN }}
```

## What happens

| Event | Result |
| --- | --- |
| Pull request opened or updated | A preview deployment with the pull request's head commit and title. One comment on the pull request shows its URL and is updated on every push |
| Push to the default branch (a merge) | A production deployment with the merge commit. Files the pull request already previewed are deployed again as they are |
| Nothing changed | No deployment; the step ends with a notice |
| Pull request from a fork or Dependabot | Skipped with a notice: such runs get no secrets |
| Empty token anywhere else | The step fails: the secret is not set or empty |
| Failed build or upload | The step fails with the error, the pull request comment shows it, and production stays as it was |

An API still in template mode switches to code mode with its first deploy: a pull request's
preview leaves production on the template's deployment, and **Back to template** in the Studio
restores the template. Roll back from the **Deployments** tab or with `npx @jojapi/cli rollback`.

Previews answer the API keys of the account that owns the API; make a deployment **Public** in
the Studio to share it.

## Release notes

The deployment's description (`message`) stays in your Studio. Consumers read a production deploy
under **Releases** on the API page only when it carries a release note; without one, only the page's
"Last updated" date moves. Pass the note from wherever your workflow keeps it, for example a step
that reads it from the merged pull request:

```yaml
      - uses: jojapicom/deploy-action@v1
        with:
          token: ${{ secrets.JOJAPI_TOKEN }}
          note: ${{ steps.release.outputs.note }}
```

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `token` | — | Management token from the **CI: production deploys** template, or **CI: preview deploys** for previews only (see [Set up](#set-up)). Pass it from a secret |
| `working-directory` | `.` | Folder with the API's `jojapi.json` and files |
| `production` | `auto` | `auto`: production on pushes to the default branch, a preview otherwise. `true` or `false` forces it |
| `message` | pull request title or commit subject | Description of the deployment, shown in your Studio |
| `note` | — | Public release note of a production deploy, listed under **Releases** on the API page. Ignored on previews |
| `comment` | `true` | Comment the preview URL on the pull request (needs `pull-requests: write`) |
| `install` | `auto` | `auto`: `npm ci` when `package-lock.json` exists and `node_modules` does not. `true` always installs, `false` never does. For pnpm or yarn, install in an earlier step |
| `github-token` | `${{ github.token }}` | Token for the pull request comment |
| `cli-version` | pinned per release | Version of [`@jojapi/cli`](https://www.npmjs.com/package/@jojapi/cli) to run |

## Outputs

| Output | Description |
| --- | --- |
| `status` | `deployed`, `unchanged`, `failed` or `skipped` |
| `url` | The new deployment's URL, `https://{api}--{id}.jojapi.dev` |
| `deployment-id` | The new deployment's id |
| `deployment-number` | The new deployment's number |
| `preview-url` | The URL that follows the latest deployment, `https://{api}--preview.jojapi.dev` |
| `promoted` | `true` when the deployment serves production |

```yaml
- uses: jojapicom/deploy-action@v1
  id: jojapi
  with:
    token: ${{ secrets.JOJAPI_TOKEN }}
- run: echo "Deployment ${{ steps.jojapi.outputs.deployment-id }} is at ${{ steps.jojapi.outputs.url }}"
  if: steps.jojapi.outputs.status == 'deployed'
```

## Several APIs in one repository

Give each API its own folder with its `jojapi.json` and one step (or job) per folder. Each API
keeps its own pull request comment.

```yaml
- uses: jojapicom/deploy-action@v1
  with:
    token: ${{ secrets.JOJAPI_TOKEN }}
    working-directory: apis/search
- uses: jojapicom/deploy-action@v1
  with:
    token: ${{ secrets.JOJAPI_TOKEN }}
    working-directory: apis/geo
```

## Security

The token reaches only the `jojapi` CLI process, as `JOJAPI_TOKEN`; the action never prints it and
the CLI never sends it anywhere but the JoJ API Management API. Use `pull_request`, not
`pull_request_target`: the latter would deploy code from forks with your token.

## Versions

`v1` follows the latest `v1.x.y` release; pin a full version (`@v1.0.0`) or a commit to freeze
it. Each release pins the CLI version it was tested with.

## Development

The action is a composite action with a dependency-free Node script (`src/run.mjs`), so there is
no build step; it runs on Linux and macOS runners and sets up Node 22 when the runner has no Node
20 or later. `node --test test/*.test.mjs` runs the tests.
