# SP-API

[![npm](https://img.shields.io/npm/v/%40selling-partner-api%2Fsdk?label=@selling-partner-api%2Fsdk&color=cb3837)](https://www.npmjs.com/package/@selling-partner-api/sdk)
[![npm](https://img.shields.io/npm/v/%40selling-partner-api%2Fmodels?label=@selling-partner-api%2Fmodels&color=cb3837)](https://www.npmjs.com/package/@selling-partner-api/models)
[![CI](https://github.com/selling-partner-api/selling-partner-api/actions/workflows/ci.yaml/badge.svg?branch=main)](https://github.com/selling-partner-api/selling-partner-api/actions/workflows/ci.yaml)
[![Publish](https://github.com/selling-partner-api/selling-partner-api/actions/workflows/publish-sdk.yaml/badge.svg?branch=main)](https://github.com/selling-partner-api/selling-partner-api/actions/workflows/publish-sdk.yaml)
[![Vendor Models Sync](https://github.com/selling-partner-api/selling-partner-api/actions/workflows/vendor-sync.yaml/badge.svg)](https://github.com/selling-partner-api/selling-partner-api/actions/workflows/vendor-sync.yaml)
![Coverage](https://raw.githubusercontent.com/selling-partner-api/selling-partner-api/main/docs/assets/coverage-badge.svg)

_**Unofficial** knowledge and resource base for the **Amazon Selling Partner API (SP-API)**._

_Real-world tips, gotchas, links and patterns._

_A developer survival guide._

> **Disclaimer**: Not affiliated with Amazon.com, Inc. or it's affiliates. This is not an official SP-API resource. It is a community project with the goal of helping developers navigate the complexities of the SP-API._

> **Note**: This is a **work in progress**. Contributions are welcome! See [Contributing](./CONTRIBUTING.md).

---

This project is focused on providing information and resources not provided by the official documentation and repositories as well as sharing real-world experiences and solutions to common problems encountered while working with the SP-API.

Official documentation can be found here: [developer-docs.amazon.com](https://developer-docs.amazon.com/sp-api/docs)

## Why this exists

The SP-API is **huge, confusing, and under-documented**.
This repo collects **developer-verified notes**, quirks, and examples that go beyond the official docs — **no leaks, no secrets, just battle-tested experience.**

---

## Contents

-   **[Guide](./docs/guide)** → core topics (auth, reports, rate limits)
-   **[Troubleshooting](./docs/troubleshooting)** → common errors, real causes
-   **[Patterns](./docs/patterns)** → integration & workflow design tips

---

## Using this repo

-   Browse on GitHub → files are just Markdown, no build required.
-   Or run locally as a [VitePress](https://vitepress.dev) site for nicer navigation.

```bash
bun install
bun run docs:dev
```

### Workspace layout

-   `packages/sdk` – the TypeScript runtime SDK published to npm and GitHub Packages as `@selling-partner-api/sdk`.
-   `packages/models` – publishes the consolidated OpenAPI schema and type surface as `@selling-partner-api/models`.
-   `vendor/selling-partner-api-models` – git submodule pointing to [amzn/selling-partner-api-models](https://github.com/amzn/selling-partner-api-models).
-   `.github/workflows` – automation for syncing models, versioning via release-please, and publishing packages to npm/GitHub Packages.

### Automation highlights

-   **Nightly model sync** (`sync-models.yml`) updates the submodule, rebuilds the merged OpenAPI artefacts, regenerates rate-limits, and dispatches a release sweep when new payloads land.
-   **Release management** is handled by [`release-please`](https://github.com/google-github-actions/release-please). Separate PRs track `@selling-partner-api/models` and `@selling-partner-api/sdk`, auto-bumping versions, changelogs, and the SDK dependency on the latest models build.
-   **CI** runs on every push/PR using Bun 1.2.x to ensure builds and tests stay green.
-   **Publish workflows** respond to GitHub releases. Once release-please cuts `@selling-partner-api/models@*` / `@selling-partner-api/sdk@*`, dedicated jobs publish to npm and GitHub Packages with provenance (requires the `NPM_SECRET` secret).

### Preparing npm publish automation

1. Create an npm automation token with the `publish` scope: <https://www.npmjs.com/settings/{your-organization}/tokens>.
2. In this repository, add a GitHub Actions secret called `NPM_SECRET` with that token value.
3. (Optional) If you need to publish under an organization scope, ensure the automation token has the correct team permissions and that the package `@selling-partner-api/sdk` is owned by that account.

Once configured, land your feature work on `main` and let release-please open the release PRs. Review/merge the PRs (auto-merge is enabled when checks pass) and the ensuing GitHub releases will trigger the publish workflows automatically.

## Contributing

Pull requests not only welcome but endorsed! See [Contributing](./CONTRIBUTING.md).
This repository follows a [Code of Conduct](./CODE_OF_CONDUCT.md).

## License

This project is licensed under the [MIT License](./LICENSE).

## Security

See [Security Policy](./SECURITY.md) for how to report sensitive issues.

## Honorable Mentions

This is not the only community SP-API resource out there! Check out these other great projects:
- [vikingcodes/awesome-spapi](https://github.com/vikingcodes/awesome-spapi)
