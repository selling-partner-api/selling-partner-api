# CI/CD Architecture and Workflow Documentation

## Overview

The repository uses a **fully automated CI/CD pipeline** with 11 production workflows:

**Core Workflows:**
- ✅ Automated testing and linting on all PRs
- ✅ Documentation deployment (only when docs change)
- ✅ Independent package releases (models and SDK)
- ✅ Auto-merge for release PRs

**Automation Workflows:**
- 🤖 Daily vendor sync from Amazon's upstream models
- 🤖 Auto-update SDK when models publishes
- 🤖 Breaking change detection for API removals
- 🤖 PR labeling and auto-assignment

**Publishing:**
- 📦 Dual publishing to npm + GitHub Packages
- 📦 Supports both manual and automatic triggers
- 📦 Provenance attestation for supply chain security

## Complete Automation Flow

```
Daily 2 AM UTC → Vendor Sync → Models Build → PR Created → Tests Pass → Auto-Merge
                                                                              ↓
                                                                    Release-Please PR
                                                                              ↓
                                                                    Auto-Merge → Publish Models
                                                                                        ↓
                                                                              SDK Auto-Update → Tests
                                                                                        ↓
                                                                              Release-Please PR
                                                                                        ↓
                                                                              Auto-Merge → Publish SDK
                                                                                        ↓
                                                                                    ✅ Complete!
```

## Workflows

### 1. Continuous Integration (`ci.yaml`)

**Triggers:**
- Push to `main` branch (excluding docs and markdown files)
- Pull requests (excluding docs and markdown files)

**Skips:**
- Release-please PRs (labeled with `release: pending`)

**Actions:**
1. Install dependencies with Bun
2. Run linting (`bun run lint`)
3. Build all packages (`bun run build`)
4. Run tests with coverage (`bunx vitest run --coverage`)
5. On main branch: Update and commit coverage badge

**Purpose:** Ensures all code changes pass quality checks before merging.

---

### 2. Documentation Deployment (`deploy-docs.yaml`)

**Triggers:**
- Push to `main` branch that affects:
  - `docs/**` files
  - `.github/workflows/deploy-docs.yaml`
- Manual workflow dispatch

**Actions:**
1. Build VitePress documentation
2. Deploy to GitHub Pages

**Purpose:** Keeps documentation up-to-date automatically, only rebuilding when docs actually change.

---

### 3. Release Please (`release-please.yaml`)

**Triggers:**
- Push to `main` branch that affects:
  - `packages/models/**`
  - `packages/sdk/**`
  - Release configuration files
- Manual workflow dispatch

**Actions:**
1. Creates/updates separate release PRs for each package
2. Manages version bumps based on conventional commits
3. Generates CHANGELOGs
4. Tags releases when PRs are merged

**Configuration:**
- Separate PRs for models and SDK (`separate-pull-requests: true`)
- Component-specific tags (e.g., `@selling-partner-api/models@1.0.0`)
- PRs labeled with `release: pending`

**Purpose:** Automates semantic versioning and release management for both packages independently.

---

### 4. Release Please Auto-Merge (`release-please-auto-merge.yaml`)

**Triggers:**
- Pull request events (opened, labeled, reopened, etc.)

**Conditions:**
- PR author is `github-actions[bot]`
- PR has `release: pending` label

**Actions:**
- Enables auto-merge with squash method

**Purpose:** Automatically merges release PRs when ready, streamlining the release process.

---

### 5. Publish Models (`publish-models.yaml`)

**Triggers:**
- Release published event for tags starting with `@selling-partner-api/models@`

**Actions:**
1. Checkout code at the release tag
2. Install dependencies with Bun
3. Build models package
4. Validate version matches tag
5. Publish to npm registry
6. Publish to GitHub Packages

**Secrets Required:**
- `NPM_SECRET`: npm authentication token

**Purpose:** Publishes the models package to both npm and GitHub Packages when a release is created.

---

### 6. Publish SDK (`publish-sdk.yaml`)

**Triggers:**
- Release published event for tags starting with `@selling-partner-api/sdk@`

**Actions:**
1. Checkout code at the release tag
2. Install dependencies with Bun
3. Run full CI checks (lint, build, test)
4. Validate version matches tag
5. **Wait for models dependency to be available on npm**
6. Publish to npm registry
7. Publish to GitHub Packages

**Secrets Required:**
- `NPM_SECRET`: npm authentication token

**Purpose:** Publishes the SDK package after ensuring its models dependency is available, preventing version mismatch issues.

---

### 7. Vendor Sync (`vendor-sync.yaml`) 🆕

**Triggers:**
- Scheduled: Daily at 2:00 AM UTC
- Manual workflow dispatch

**Actions:**
1. Update vendor submodule from Amazon's upstream repository
2. Check if `models/` or `schemas/` directories actually changed
3. Rebuild models package (`merged.json` and `paths.ts`)
4. Detect if build produced changes
5. Create PR with conventional commit: `chore(models): sync vendor models`
6. Enable auto-merge

**Smart Detection:**
- Only creates PR if vendor submodule changed
- Only builds if `models/` or `schemas/` affected
- Only creates PR if build produces changes

**Purpose:** Daily automatic synchronization with Amazon's Selling Partner API models, creating a PR only when there are actual changes to review.

---

### 8. Update SDK on Models Release (`update-sdk-on-models-release.yaml`) 🆕

**Triggers:**
- Release published for models (tags: `models-v*` or `@selling-partner-api/models@*`)
- Manual workflow dispatch with models version input

**Actions:**
1. Extract models version from release tag
2. Check if SDK update is needed
3. **Detect breaking changes:**
   - Installs old and new models versions from npm
   - Compares exported paths
   - Detects removed API endpoints
4. Update SDK's models dependency in `package.json`
5. Determine commit type based on changes:
   - **Breaking changes** → `feat(sdk)!:` (major version bump)
   - **Minor update** → `feat(sdk):` (minor version bump)
   - **Patch update** → `fix(sdk):` (patch version bump)
6. Create PR with appropriate conventional commit
7. Enable auto-merge

**Breaking Change Detection:**
```typescript
// Compares old vs new paths from models package
const removedPaths = oldPaths.filter(p => !newPaths.includes(p));
if (removedPaths.length > 0) {
  // Breaking change detected!
  return { breaking: true, removedCount: removedPaths.length };
}
```

**Purpose:** Automatically updates SDK when models publishes, with intelligent semantic versioning based on breaking change detection.

---

### 9. Dependabot Auto-Merge (`dependabot-auto-merge.yaml`)

**Triggers:**
- Pull request events from Dependabot

**Actions:**
- Auto-approves and merges security updates and patch version updates

**Purpose:** Keeps dependencies up-to-date with minimal manual intervention.

---

## Release Flow

### Models Package Release

1. Developer makes changes to `packages/models/**`
2. CI runs on PR
3. PR merged to main
4. Release-please creates/updates release PR
5. Release PR auto-merges (or manually merged)
6. GitHub release created with tag `@selling-partner-api/models@X.Y.Z`
7. `publish-models.yaml` publishes to npm and GitHub Packages
8. `update-sdk-models-dependency.yaml` creates PR to update SDK dependency
9. SDK dependency PR follows standard flow...

### SDK Package Release

1. Developer makes changes to `packages/sdk/**` (or dependency update PR created)
2. CI runs on PR
3. PR merged to main
4. Release-please creates/updates release PR
5. Release PR auto-merges (or manually merged)
6. GitHub release created with tag `@selling-partner-api/sdk@X.Y.Z`
7. `publish-sdk.yaml`:
   - Runs full test suite
   - Waits for models dependency availability
   - Publishes to npm and GitHub Packages

### Preventing Conflicts

The workflows prevent simultaneous releases through:
- **Separate release PRs**: Each package gets its own PR
- **Dependency updates via PR**: Models releases trigger SDK dependency updates through a new PR, not direct modification
- **Sequential flow**: SDK waits for models availability before publishing

---

## Testing Strategy

### PR Checks
- All PRs (except release-please) run full CI
- Tests, linting, and build must pass

### Pre-Publish Checks
- SDK package runs full test suite before publishing
- Models package only builds (no tests currently)

### Coverage Reporting
- Coverage badge automatically updated on main branch pushes
- Located at `docs/assets/coverage-badge.svg`

---

## Conventional Commits

Release-please relies on conventional commit messages to determine version bumps:

- `fix:` → Patch version (0.0.X)
- `feat:` → Minor version (0.X.0)
- `BREAKING CHANGE:` or `feat!:` or `fix!:` → Major version (X.0.0)
- `chore:`, `docs:`, `style:`, etc. → No release

Special markers:
- `[skip ci]` - Skip CI workflows
- `[skip release]` - Skip release-please processing

---

## Secrets Configuration

Required repository secrets:

| Secret | Description | Used By |
|--------|-------------|---------|
| `NPM_SECRET` | npm authentication token for publishing | `publish-models.yaml`, `publish-sdk.yaml` |
| `GITHUB_TOKEN` | Automatically provided by GitHub Actions | All workflows (GitHub Packages, PR creation) |

---

## Manual Operations

### Manually Trigger Release
```bash
# Via GitHub UI: Actions → release-please → Run workflow
```

### Manually Sync Models
```bash
# Via GitHub UI: Actions → sync-models → Run workflow
```

### Skip CI/Release
```bash
git commit -m "chore: update config [skip ci][skip release]"
```

---

## Best Practices

1. **Use conventional commits** for automatic versioning
2. **Let automation handle releases** - merge release PRs when ready
3. **Review dependency update PRs** from models releases before merging
4. **Monitor workflow runs** for any publishing failures
5. **Test locally** before pushing: `bun run lint && bun run build && bun run test`

---

## Troubleshooting

### Release PR not created
- Check if commits use conventional commit format
- Verify paths affected are in release-please config

### Publish fails waiting for models
- Check if models package published successfully
- Verify npm registry availability
- Check if version in SDK's package.json matches published models version

### Tests fail in publish-sdk
- Fix must be merged to main first
- Release PR will be updated automatically
- Re-merge release PR after fix

### Dual package releases
- Should not happen with current setup
- If it does, merge one release PR at a time
- Dependency update PR will sync versions


## Workflow Trigger Map

```
┌─────────────────────────────────────────────────────────────┐
│                    Event-Driven Workflows                    │
└─────────────────────────────────────────────────────────────┘

Push to main (code changes)
  ├─→ ci.yaml (if not docs-only)
  │   └─→ Run lint, build, test, update coverage badge
  │
  ├─→ deploy-docs.yaml (if docs changed)
  │   └─→ Build & deploy VitePress to GitHub Pages
  │
  └─→ release-please.yaml (if packages changed)
      └─→ Create/update release PRs

Pull Request opened/updated
  ├─→ ci.yaml (if not release PR)
  │   └─→ Run lint, build, test
  │
  ├─→ release-please-auto-merge.yaml (if release PR)
  │   └─→ Enable auto-merge
  │
  └─→ dependabot-auto-merge.yaml (if Dependabot PR)
      └─→ Auto-approve & merge security/patch updates

Release Published
  ├─→ publish-models.yaml (if @selling-partner-api/models@*)
  │   ├─→ Build & publish to npm + GitHub Packages
  │   └─→ Triggers: update-sdk-models-dependency.yaml
  │       └─→ Create PR to update SDK dependency
  │
  └─→ publish-sdk.yaml (if @selling-partner-api/sdk@*)
      ├─→ Wait for models dependency on npm
      ├─→ Run full test suite
      └─→ Build & publish to npm + GitHub Packages

Scheduled (Daily 5:00 AM UTC)
  └─→ sync-models.yaml
      ├─→ Update vendor/selling-partner-api-models
      ├─→ Rebuild models & SDK
      └─→ Create PR with changes
```

## Release Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                     Models Release Flow                       │
└──────────────────────────────────────────────────────────────┘

Developer commits to packages/models/
  ↓
PR opened → ci.yaml runs
  ↓
PR merged to main
  ↓
release-please.yaml creates release PR
  ↓
Release PR merged (auto or manual)
  ↓
GitHub release created (@selling-partner-api/models@X.Y.Z)
  ↓
publish-models.yaml publishes package
  ↓
update-sdk-models-dependency.yaml creates PR
  ↓
[SDK Release Flow starts...]


┌──────────────────────────────────────────────────────────────┐
│                      SDK Release Flow                         │
└──────────────────────────────────────────────────────────────┘

Developer commits to packages/sdk/ (or dependency update PR)
  ↓
PR opened → ci.yaml runs
  ↓
PR merged to main
  ↓
release-please.yaml creates release PR
  ↓
Release PR merged (auto or manual)
  ↓
GitHub release created (@selling-partner-api/sdk@X.Y.Z)
  ↓
publish-sdk.yaml:
  ├─→ Waits for @selling-partner-api/models availability
  ├─→ Runs tests
  └─→ Publishes package
```

## Conflict Prevention Strategy

```
┌──────────────────────────────────────────────────────────────┐
│         How We Prevent Simultaneous Package Releases         │
└──────────────────────────────────────────────────────────────┘

✓ Separate Release PRs
  - release-please config: separate-pull-requests: true
  - Each package gets its own PR
  - PRs can be merged independently

✓ Dependency Updates via PR
  - Models release → Creates new PR for SDK
  - Not a direct commit to main
  - Goes through normal review cycle

✓ Sequential Publishing
  - SDK waits for models on npm before publishing
  - Max wait: 5 minutes (30 attempts × 10s)

✓ No Automatic Merge of Both
  - Even with auto-merge enabled
  - PRs are separate
  - One merges, triggers release
  - Second PR stays open until ready
```

## When Workflows Run

| Workflow | On Push | On PR | On Release | Scheduled | Manual |
|----------|---------|-------|------------|-----------|--------|
| ci.yaml | ✓ | ✓ | | | |
| deploy-docs.yaml | ✓ (docs) | | | | ✓ |
| release-please.yaml | ✓ (pkg) | | | | ✓ |
| release-please-auto-merge.yaml | | ✓ (bot) | | | |
| publish-models.yaml | | | ✓ (models) | | |
| publish-sdk.yaml | | | ✓ (sdk) | | |
| update-sdk-models-dependency.yaml | | | ✓ (models) | | |
| sync-models.yaml | | | | ✓ (daily) | ✓ |
| dependabot-auto-merge.yaml | | ✓ (bot) | | | |

## Key Features

### ✓ Full Automation
- Releases managed by conventional commits
- Auto-merge for release PRs
- Auto-updates for dependencies

### ✓ Quality Gates
- Lint & test before merge
- Full test suite before SDK publish
- Version validation on publish

### ✓ Smart Triggers
- Docs only deploy when docs change
- CI skips release PRs (they'll be tested on publish)
- Path-based filtering reduces unnecessary runs

### ✓ Dual Publishing
- Both packages → npm + GitHub Packages
- Provenance attestation enabled
- Public access configured

### ✓ Dependency Safety
- SDK waits for models availability
- Exact version matching validated
- Timeout protection (won't wait forever)

## Common Commands

```bash
# Trigger a release (commit to main with conventional commit)
git commit -m "feat: add new feature"

# Skip CI for docs-only changes
git commit -m "docs: update readme [skip ci]"

# Skip both CI and release
git commit -m "chore: update config [skip ci][skip release]"

# Manual release trigger
# GitHub UI → Actions → release-please → Run workflow

# Manual model sync
# GitHub UI → Actions → sync-models → Run workflow
```

## Monitoring

Watch these areas:
- GitHub Actions tab for workflow runs
- Release PRs for version bumps
- npm registry for published packages
- Coverage badge for test coverage trends

## Emergency Procedures

### Stop a bad release
1. Don't merge the release PR
2. Fix the issue in a new PR
3. Release PR will update automatically

### Revert a published package
```bash
# For npm (within 72 hours)
npm unpublish @selling-partner-api/sdk@X.Y.Z

# Better approach: Publish a new patch version with fix
```

### Manual publish if automation fails
```bash
# Not recommended, but possible:
# 1. Checkout the release tag
# 2. Build the package
# 3. npm publish manually
# See individual publish-*.yaml for exact steps
```


## Complete End-to-End Flow

```
╔════════════════════════════════════════════════════════════════════════╗
║                         VENDOR SYNC WORKFLOW                           ║
║                      (Daily at 2 AM UTC)                               ║
╚════════════════════════════════════════════════════════════════════════╝
                                  │
                                  ▼
                    ┌─────────────────────────────┐
                    │   Update Vendor Submodule   │
                    │ vendor/selling-partner-api- │
                    │         models              │
                    └─────────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────┐
                    │  Check for Changes in:      │
                    │  - models/                  │
                    │  - schemas/                 │
                    └─────────────────────────────┘
                                  │
                        ┌─────────┴─────────┐
                        │                   │
                  No Changes          Changes Found
                        │                   │
                        ▼                   ▼
                    ┌───────┐    ┌─────────────────────────┐
                    │ Skip  │    │ Rebuild Models Package  │
                    └───────┘    │  - merged.json          │
                                 │  - paths.ts             │
                                 └─────────────────────────┘
                                             │
                                             ▼
                                 ┌─────────────────────────┐
                                 │ Create PR:              │
                                 │ "chore(models): sync    │
                                 │   vendor models"        │
                                 └─────────────────────────┘
                                             │
                                             ▼
                                 ┌─────────────────────────┐
                                 │ CI Tests Run            │
                                 └─────────────────────────┘
                                             │
                                    ✅ Tests Pass
                                             │
                                             ▼
                                 ┌─────────────────────────┐
                                 │ Auto-Merge PR           │
                                 └─────────────────────────┘
                                             │
╔════════════════════════════════════════════════════════════════════════╗
║                      RELEASE-PLEASE WORKFLOW                           ║
╚════════════════════════════════════════════════════════════════════════╝
                                             │
                                             ▼
                                 ┌─────────────────────────┐
                                 │ Analyze Commits         │
                                 │ - chore → patch         │
                                 │ - fix → patch           │
                                 │ - feat → minor          │
                                 │ - feat! → major         │
                                 └─────────────────────────┘
                                             │
                                             ▼
                                 ┌─────────────────────────┐
                                 │ Create Release PR:      │
                                 │ "chore(models): release │
                                 │    X.Y.Z"               │
                                 └─────────────────────────┘
                                             │
                                             ▼
                                 ┌─────────────────────────┐
                                 │ Auto-Merge Release PR   │
                                 └─────────────────────────┘
                                             │
╔════════════════════════════════════════════════════════════════════════╗
║                       PUBLISH MODELS WORKFLOW                          ║
╚════════════════════════════════════════════════════════════════════════╝
                                             │
                                             ▼
                                 ┌─────────────────────────┐
                                 │ Build & Test            │
                                 └─────────────────────────┘
                                             │
                                             ▼
                                 ┌─────────────────────────┐
                                 │ Publish to npm:         │
                                 │ @selling-partner-api/   │
                                 │   models@X.Y.Z          │
                                 └─────────────────────────┘
                                             │
                                             ▼
                                 ┌─────────────────────────┐
                                 │ Publish to GitHub       │
                                 │ Packages                │
                                 └─────────────────────────┘
                                             │
╔════════════════════════════════════════════════════════════════════════╗
║                    SDK AUTO-UPDATE WORKFLOW                            ║
╚════════════════════════════════════════════════════════════════════════╝
                                             │
                                             ▼
                                 ┌─────────────────────────┐
                                 │ Detect Models Release   │
                                 │ (via tag)               │
                                 └─────────────────────────┘
                                             │
                                             ▼
                                 ┌─────────────────────────┐
                                 │ Install Old Version     │
                                 │ (from npm)              │
                                 └─────────────────────────┘
                                             │
                                             ▼
                                 ┌─────────────────────────┐
                                 │ Install New Version     │
                                 │ (from npm)              │
                                 └─────────────────────────┘
                                             │
                                             ▼
                                 ┌─────────────────────────┐
                                 │ Compare paths:          │
                                 │ - Check for removals    │
                                 │ - Detect breaking       │
                                 │   changes               │
                                 └─────────────────────────┘
                                             │
                        ┌────────────────────┴────────────────────┐
                        │                                         │
                  Breaking Found                          No Breaking
                        │                                         │
                        ▼                                         ▼
            ┌───────────────────────┐             ┌─────────────────────────┐
            │ Commit: feat(sdk)!:   │             │ Check Version Type:     │
            │ (Major bump)          │             │ - Major/Minor → feat:   │
            └───────────────────────┘             │ - Patch → fix:          │
                        │                         └─────────────────────────┘
                        │                                         │
                        └────────────────┬────────────────────────┘
                                         │
                                         ▼
                             ┌─────────────────────────┐
                             │ Update package.json     │
                             │ dependency              │
                             └─────────────────────────┘
                                         │
                                         ▼
                             ┌─────────────────────────┐
                             │ Create PR               │
                             └─────────────────────────┘
                                         │
                                         ▼
                             ┌─────────────────────────┐
                             │ Run SDK Tests           │
                             └─────────────────────────┘
                                         │
                                ✅ Tests Pass
                                         │
                                         ▼
                             ┌─────────────────────────┐
                             │ Auto-Merge PR           │
                             └─────────────────────────┘
                                         │
╔════════════════════════════════════════════════════════════════════════╗
║                      RELEASE-PLEASE WORKFLOW                           ║
║                          (SDK Release)                                 ║
╚════════════════════════════════════════════════════════════════════════╝
                                         │
                                         ▼
                             ┌─────────────────────────┐
                             │ Create SDK Release PR   │
                             └─────────────────────────┘
                                         │
                                         ▼
                             ┌─────────────────────────┐
                             │ Auto-Merge Release PR   │
                             └─────────────────────────┘
                                         │
╔════════════════════════════════════════════════════════════════════════╗
║                        PUBLISH SDK WORKFLOW                            ║
╚════════════════════════════════════════════════════════════════════════╝
                                         │
                                         ▼
                             ┌─────────────────────────┐
                             │ Build & Test            │
                             └─────────────────────────┘
                                         │
                                         ▼
                             ┌─────────────────────────┐
                             │ Publish to npm:         │
                             │ @selling-partner-api/   │
                             │   sdk@X.Y.Z             │
                             └─────────────────────────┘
                                         │
                                         ▼
                             ┌─────────────────────────┐
                             │ Publish to GitHub       │
                             │ Packages                │
                             └─────────────────────────┘
                                         │
                                         ▼
                             ╔═════════════════════════╗
                             ║    🎉 COMPLETE! 🎉      ║
                             ║                         ║
                             ║ Both packages updated   ║
                             ║ and published to npm!   ║
                             ╚═════════════════════════╝
```

## Timeline Example

**Day 1 - 2:00 AM UTC:**
```
02:00 - vendor-sync.yaml runs
02:01 - Vendor submodule updated (commit abc123)
02:02 - Models rebuilt (merged.json, paths.ts changed)
02:03 - PR created: "chore(models): sync vendor models"
02:04 - CI starts (lint, build, test)
02:06 - CI passes ✅
02:06 - PR auto-merges
```

**Day 1 - 2:10 AM UTC:**
```
02:10 - release-please.yaml triggers
02:11 - Analyzes commits: 1 chore commit found
02:12 - Creates PR: "chore(models): release 0.4.1"
02:13 - Auto-merge enabled
02:13 - PR auto-merges (no CI on release PRs)
```

**Day 1 - 2:15 AM UTC:**
```
02:15 - publish-models.yaml triggers (tag: models-v0.4.1)
02:16 - Builds models package
02:17 - Tests pass ✅
02:18 - Publishes to npm: @selling-partner-api/models@0.4.1
02:19 - Publishes to GitHub Packages
```

**Day 1 - 2:20 AM UTC:**
```
02:20 - update-sdk-on-models-release.yaml triggers
02:21 - Installs models@0.4.0 (current)
02:22 - Installs models@0.4.1 (new)
02:23 - Compares paths: no removals found ✅
02:24 - Determines commit type: fix (patch update)
02:25 - Updates SDK package.json: models@^0.4.1
02:26 - Creates PR: "fix(sdk): update models to 0.4.1"
02:27 - SDK tests run
02:29 - Tests pass ✅
02:29 - PR auto-merges
```

**Day 1 - 2:35 AM UTC:**
```
02:35 - release-please.yaml triggers
02:36 - Analyzes commits: 1 fix commit found
02:37 - Creates PR: "chore(sdk): release 1.0.1"
02:38 - Auto-merge enabled
02:38 - PR auto-merges
```

**Day 1 - 2:40 AM UTC:**
```
02:40 - publish-sdk.yaml triggers (tag: sdk-v1.0.1)
02:41 - Builds SDK package
02:43 - Tests pass ✅
02:44 - Publishes to npm: @selling-partner-api/sdk@1.0.1
02:45 - Publishes to GitHub Packages
02:46 - ✅ COMPLETE! Both packages updated and published!
```

**Total Time:** ~46 minutes from vendor sync to final publish
**Manual Intervention:** 0 steps
**Human Involvement:** Review commits the next morning ☕

## Workflow Dependencies

```
vendor-sync.yaml
    │
    ├─> Creates commit → Triggers release-please.yaml
    │
    └─> release-please.yaml
            │
            └─> Creates tag → Triggers publish-models.yaml
                    │
                    └─> Publishes to npm → Triggers update-sdk-on-models-release.yaml
                            │
                            └─> Creates commit → Triggers release-please.yaml
                                    │
                                    └─> Creates tag → Triggers publish-sdk.yaml
                                            │
                                            └─> Publishes to npm
                                                    │
                                                    └─> ✅ Done!
```

## Breaking Change Flow

**Scenario:** Amazon removes an API endpoint

```
Day 1, 2:00 AM - Vendor Sync
    │
    ├─> API endpoint removed in vendor/selling-partner-api-models
    │
    └─> Models rebuilt (paths.ts no longer includes removed endpoint)
            │
            └─> PR created → Auto-merged
                    │
                    └─> Models published: 0.4.1 → 0.5.0 (minor bump, no breaking)
                            │
                            └─> SDK auto-update runs
                                    │
                                    ├─> Breaking change detected! ⚠️
                                    │   (path '/orders/v0/orders' removed)
                                    │
                                    └─> Creates PR: "feat(sdk)!: update models to 0.5.0"
                                            │
                                            └─> BREAKING CHANGE footer added
                                                    │
                                                    └─> Auto-merged
                                                            │
                                                            └─> SDK released: 1.0.1 → 2.0.0 (major bump!)
                                                                    │
                                                                    └─> Users see clear breaking change in CHANGELOG
```

## Manual Override Points

Every workflow has manual trigger capability:

```
Manual Triggers Available:
├─ vendor-sync.yaml ─────────→ gh workflow run vendor-sync.yaml
├─ publish-models.yaml ──────→ gh workflow run publish-models.yaml -f tag=models-v0.5.0
├─ publish-sdk.yaml ─────────→ gh workflow run publish-sdk.yaml -f tag=sdk-v2.0.0
└─ update-sdk-on-models-release.yaml → gh workflow run update-sdk-on-models-release.yaml -f models_version=0.5.0
```

## Monitoring Dashboard (CLI)

```bash
# Today's automation status
gh run list --created "$(date +%Y-%m-%d)" --limit 20

# Vendor sync history
gh run list --workflow=vendor-sync.yaml --limit 5

# Recent releases
gh release list --limit 10

# Current package versions
npm view @selling-partner-api/models version
npm view @selling-partner-api/sdk version

# Open PRs (should be mostly empty if auto-merge works!)
gh pr list
```
