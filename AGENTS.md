# jojapicom/deploy-action

Instructions for coding agents and people working in this repository. This repository is public: everything written
here is published.

## What this is

"Deploy to JoJ API", a composite GitHub Action that runs the `@jojapi/cli` to deploy an API Worker: a preview for every
pull request, production on the default branch, with an optional public release note. A dependency-free Node script
(`src/run.mjs`), no build step. User documentation: [README.md](README.md); inputs and outputs: `action.yml`.

## Commands

```bash
node --test test/*.test.mjs
```

## Releasing

1. Commit to `main` and push.
2. Push an annotated tag `v1.0.x` (message = the release name).
3. `gh release create v1.0.x --verify-tag --latest --notes-file <file>` with one English paragraph, like the earlier
   notes.
4. The owner publishes it to the GitHub Marketplace: a release created through `gh` or the API never appears there, so
   the release's edit page needs "Publish this Action to the GitHub Marketplace" ticked and saved by hand.
5. The owner moves `v1` to the same commit:
   `git tag -f -a v1 -m "v1: <summary>" <sha> && git push --force origin refs/tags/v1`.
   Check with `git ls-remote --tags origin 'refs/tags/v1^{}'` and the version shown on the Marketplace page.

Steps 4 and 5 are two separate things the owner does; never report them as one. A release that pins a new CLI version
(`cli-version` in `action.yml`) is made only after that CLI version is live on npm.

## Rules

- `v1` must keep working for everyone who uses `@v1`: inputs keep their names and defaults, outputs keep their values.
- The token reaches only the CLI process as `JOJAPI_TOKEN` and is never printed.
- Workflows that use the action run on `pull_request`, never `pull_request_target`.
- Commit messages in English; nothing about customers, prices or people in code, docs or commits.

## Behaviour that is easy to get wrong

- **Private npm dependencies.** In a caller's workflow, `actions/checkout` writes the workflow token into the
  repository's git config by default (`persist-credentials: true`); `npm ci` then sends that header instead of the
  token in the `insteadOf` URL and GitHub answers "Repository not found", whether the secret is set or not. Such
  repositories need `persist-credentials: false` on checkout together with the token secret and the `insteadOf` rule.
- **A release note is recorded only by a production deploy that changes files.** A comment-only change produces no
  new bundle, hence no deploy and no note.
- The CLI uploads more than `index.mjs` from the working directory (see the CLI's AGENTS.md, "Upload set").
