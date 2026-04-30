# Monorepo Setup Guide: NPM Workspaces + Publishable Packages + Apps

This skill walks you through scaffolding a production-ready monorepo from scratch, following the same structure used in this project. It covers:

- Root workspace configuration with npm workspaces
- A publishable NPM package (TypeScript, standard-version, CHANGELOG)
- commitlint + Husky for enforced conventional commits
- App workspaces: a Next.js web app and an Express/Node API
- Release workflow (patch / minor / major)
- CI/CD integration tips

---

## Table of Contents

1. [Monorepo Structure](#1-monorepo-structure)
2. [Root Package Setup](#2-root-package-setup)
3. [Conventional Commits with commitlint & Husky](#3-conventional-commits-with-commitlint--husky)
4. [Publishable NPM Package](#4-publishable-npm-package)
5. [Web App Workspace (Next.js)](#5-web-app-workspace-nextjs)
6. [API App Workspace (Express)](#6-api-app-workspace-express)
7. [Cross-Workspace Dependency](#7-cross-workspace-dependency)
8. [Release Workflow](#8-release-workflow)
9. [CI/CD Integration](#9-cicd-integration)
10. [Quick-Reference Cheatsheet](#10-quick-reference-cheatsheet)

---

## 1. Monorepo Structure

```
my-monorepo/
├── apps/
│   ├── web/              # Next.js SaaS dashboard
│   └── api/              # Express/Node API server
├── packages/
│   └── my-package/       # Publishable NPM package
├── .husky/               # Git hooks
├── .gitignore
├── package.json          # Root workspace
└── package-lock.json
```

- `apps/*` — application workspaces (private, not published to npm)
- `packages/*` — library workspaces (publishable npm packages)

---

## 2. Root Package Setup

### Initialize the root

```bash
mkdir my-monorepo && cd my-monorepo
git init
npm init -y
```

### Edit `package.json` to configure workspaces

```json
{
  "name": "my-monorepo",
  "version": "1.0.0",
  "private": true,
  "description": "My monorepo",
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "dev": "npm run dev --workspace=apps/web",
    "build": "npm run build --workspaces --if-present",
    "start": "npm run start --workspace=apps/web",
    "lint": "npm run lint --workspaces --if-present",
    "test": "npm run test --workspaces --if-present"
  },
  "engines": {
    "node": ">=20.0.0"
  }
}
```

### Create the workspace directories

```bash
mkdir -p apps/web apps/api packages/my-package
```

### `.gitignore`

```
node_modules/
.next/
dist/
*.log
.env*.local
.DS_Store
```

---

## 3. Conventional Commits with commitlint & Husky

Enforcing the [Conventional Commits](https://www.conventionalcommits.org/) spec at the git-hook level ensures that `standard-version` can always generate accurate changelogs.

### Install tools at the root

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional husky
```

### Create `commitlint.config.js` at the root

```js
// commitlint.config.js
export default {
  extends: ["@commitlint/config-conventional"],
};
```

Or as `.commitlintrc.json`:

```json
{
  "extends": ["@commitlint/config-conventional"]
}
```

### Initialise Husky and add the commit-msg hook

```bash
npx husky init
echo "npx --no -- commitlint --edit \$1" > .husky/commit-msg
chmod +x .husky/commit-msg
```

Husky will now reject commits whose messages don't follow the conventional format.

### Commit message format

```
<type>(<optional-scope>): <subject>

[optional body]

[optional footer(s)]
```

| Type | SemVer impact | Description |
|------|--------------|-------------|
| `feat` | MINOR | New feature |
| `fix` | PATCH | Bug fix |
| `perf` | PATCH | Performance improvement |
| `docs` | — | Documentation only |
| `refactor` | — | Code refactoring |
| `test` | — | Adding / fixing tests |
| `chore` | — | Build process, tooling |
| `style` | — | Formatting, no logic change |
| `feat!` / `BREAKING CHANGE` | MAJOR | Breaking API change |

**Examples:**

```bash
git commit -m "feat(api): add user authentication endpoint"
git commit -m "fix(web): correct token expiration check"
git commit -m "feat!: rename CLI command syntax

BREAKING CHANGE: 'old-cmd' is now 'new-cmd'"
```

---

## 4. Publishable NPM Package

### Scaffold the package

```bash
cd packages/my-package
npm init -y
```

### `packages/my-package/package.json`

```json
{
  "name": "@your-org/my-package",
  "version": "0.1.0",
  "description": "My reusable package",
  "type": "module",
  "main": "./dist/index.js",
  "exports": {
    ".": "./dist/index.js"
  },
  "files": [
    "dist",
    "README.md"
  ],
  "scripts": {
    "build": "tsc",
    "dev": "tsx src/index.ts",
    "clean": "rm -rf dist",
    "prepublishOnly": "npm run build",
    "release": "standard-version",
    "release:patch": "standard-version --release-as patch",
    "release:minor": "standard-version --release-as minor",
    "release:major": "standard-version --release-as major"
  },
  "publishConfig": {
    "access": "public"
  },
  "license": "MIT",
  "devDependencies": {
    "standard-version": "^9.5.0",
    "typescript": "^5.0.0",
    "tsx": "^4.7.0",
    "@types/node": "^20.0.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

### Install package dependencies

```bash
# from the root of the monorepo
npm install --workspace=packages/my-package
```

### `packages/my-package/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ES2020",
    "lib": ["ES2020"],
    "moduleResolution": "node",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Source entry point

```bash
mkdir src
cat > src/index.ts << 'EOF'
export const hello = (name: string): string => `Hello, ${name}!`;
EOF
```

### Configure `.versionrc.json` for changelog generation

Create `packages/my-package/.versionrc.json`:

```json
{
  "types": [
    { "type": "feat",     "section": "Features",          "hidden": false },
    { "type": "fix",      "section": "Bug Fixes",          "hidden": false },
    { "type": "perf",     "section": "Performance",        "hidden": false },
    { "type": "docs",     "section": "Documentation",      "hidden": false },
    { "type": "refactor", "section": "Code Refactoring",   "hidden": false },
    { "type": "style",    "section": "Styling",            "hidden": true  },
    { "type": "test",     "section": "Tests",              "hidden": true  },
    { "type": "chore",    "section": "Chores",             "hidden": true  }
  ],
  "commitUrlFormat":  "https://github.com/YOUR_ORG/YOUR_REPO/commit/{{hash}}",
  "compareUrlFormat": "https://github.com/YOUR_ORG/YOUR_REPO/compare/{{previousTag}}...{{currentTag}}",
  "issueUrlFormat":   "https://github.com/YOUR_ORG/YOUR_REPO/issues/{{id}}",
  "userUrlFormat":    "https://github.com/{{user}}",
  "releaseCommitMessageFormat": "chore(release): {{currentTag}}",
  "issuePrefixes": ["#"],
  "skipBump":   false,
  "skipCommit": false,
  "skipTag":    false
}
```

Replace `YOUR_ORG/YOUR_REPO` with your GitHub repository path.

### Initialize the CHANGELOG

```bash
# In packages/my-package
cat > CHANGELOG.md << 'EOF'
# Changelog

All notable changes to this project will be documented in this file. See [standard-version](https://github.com/conventional-changelog/standard-version) for commit guidelines.
EOF
```

---

## 5. Web App Workspace (Next.js)

### Scaffold the app

```bash
cd apps/web
npx create-next-app@latest . --typescript --eslint --tailwind --app
```

The generated `package.json` will already have `name` set. Update it to use a scoped workspace name:

```json
{
  "name": "@my-monorepo/web",
  "version": "0.1.0",
  "private": true,
  ...
}
```

### Key scripts in `apps/web/package.json`

```json
{
  "scripts": {
    "dev":   "next dev",
    "build": "next build",
    "start": "next start",
    "lint":  "eslint"
  }
}
```

### Run from root

```bash
# Development
npm run dev --workspace=apps/web

# Build
npm run build --workspace=apps/web
```

---

## 6. API App Workspace (Express)

### Scaffold the app

```bash
mkdir -p apps/api/src
cd apps/api
npm init -y
```

### `apps/api/package.json`

```json
{
  "name": "@my-monorepo/api",
  "version": "0.1.0",
  "private": true,
  "description": "API server",
  "main": "dist/index.js",
  "type": "module",
  "scripts": {
    "dev":   "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "lint":  "eslint src"
  },
  "dependencies": {
    "express": "^4.18.2",
    "dotenv":  "^16.3.1"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/node":    "^20.0.0",
    "tsx":            "^4.7.0",
    "typescript":     "^5.0.0",
    "eslint":         "^9.0.0"
  },
  "engines": {
    "node": ">=20.0.0"
  }
}
```

### `apps/api/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ES2020",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Minimal entry point

```typescript
// apps/api/src/index.ts
import express from "express";
import dotenv from "dotenv";

dotenv.config();

const app = express();
const PORT = process.env.PORT ?? 3001;

app.use(express.json());

app.get("/health", (_req, res) => {
  res.json({ status: "ok" });
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### Run from root

```bash
npm run dev --workspace=apps/api
```

---

## 7. Cross-Workspace Dependency

To use your local package inside a workspace app:

```bash
# Install the local package into the web app
npm install @your-org/my-package --workspace=apps/web
```

npm workspaces will symlink the local package automatically — no need to publish first.

Import it normally:

```typescript
import { hello } from "@your-org/my-package";
```

---

## 8. Release Workflow

The release workflow lives **inside the package workspace** (`packages/my-package`), not at the root.

### Step 1 — Make changes and commit (using conventional commits)

```bash
git add .
git commit -m "feat(my-package): add new utility function"
```

commitlint will validate the message format before the commit is accepted.

### Step 2 — Create a release

```bash
cd packages/my-package

# Let standard-version auto-detect the bump from commits
npm run release

# Or specify the bump level explicitly
npm run release:patch   # 0.1.0 → 0.1.1
npm run release:minor   # 0.1.0 → 0.2.0
npm run release:major   # 0.1.0 → 1.0.0
```

`standard-version` will:
1. Analyze commits since the last git tag
2. Bump the version in `package.json`
3. Append a new section to `CHANGELOG.md`
4. Create a release commit: `chore(release): v0.2.0`
5. Create a git tag: `v0.2.0`

### Step 3 — Dry run (optional, recommended before first release)

```bash
npx standard-version --dry-run
```

### Step 4 — Push and publish

```bash
# Push commits and tags
git push origin main --follow-tags

# Publish to npm
npm publish --access public --workspace=packages/my-package
```

### Undo a release

```bash
# Delete the tag
git tag -d v0.2.0

# Undo the release commit
git reset --soft HEAD~1

# Fix and retry
npm run release:minor
```

---

## 9. CI/CD Integration

### GitHub Actions — Publish on tag push

Create `.github/workflows/publish.yml`:

```yaml
name: Publish Package

on:
  push:
    tags:
      - "v*"

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          registry-url: "https://registry.npmjs.org"

      - name: Install dependencies
        run: npm ci

      - name: Build package
        run: npm run build --workspace=packages/my-package

      - name: Publish to npm
        run: npm publish --access public --workspace=packages/my-package
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

> Store your npm token in the repository secret `NPM_TOKEN`.

### GitHub Actions — Lint & test on every push

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint --workspaces --if-present

      - name: Build
        run: npm run build --workspaces --if-present

      - name: Test
        run: npm run test --workspaces --if-present
```

---

## 10. Quick-Reference Cheatsheet

### Workspace commands

```bash
# Run a script in one workspace
npm run <script> --workspace=apps/web

# Run a script in all workspaces (skip missing)
npm run <script> --workspaces --if-present

# Install a dep in a specific workspace
npm install <pkg> --workspace=packages/my-package

# Install a local package across workspaces
npm install @your-org/my-package --workspace=apps/web
```

### Release commands (inside `packages/my-package`)

```bash
npm run release           # auto-detect bump
npm run release:patch     # bug fixes only
npm run release:minor     # new features
npm run release:major     # breaking changes

# Dry run (no file changes)
npx standard-version --dry-run

# Push after release
git push origin main --follow-tags
```

### Commit types

| Type | What changed | Version bump |
|------|-------------|-------------|
| `feat` | New feature | MINOR |
| `fix` | Bug fix | PATCH |
| `perf` | Performance | PATCH |
| `feat!` / `BREAKING CHANGE` | API break | MAJOR |
| `docs`, `refactor`, `test`, `chore` | Internal | none |

---

## References

- [npm Workspaces docs](https://docs.npmjs.com/cli/v10/using-npm/workspaces)
- [Conventional Commits spec](https://www.conventionalcommits.org/)
- [standard-version](https://github.com/conventional-changelog/standard-version)
- [commitlint](https://commitlint.js.org/)
- [Husky](https://typicode.github.io/husky/)
- [Semantic Versioning](https://semver.org/)
