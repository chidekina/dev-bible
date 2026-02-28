# Dependency Auditing

## Overview

Modern Node.js applications depend on hundreds of packages. Each one is a potential attack vector — a compromised or vulnerable dependency can expose your entire application. Dependency auditing is the practice of systematically checking your dependency tree for known vulnerabilities, keeping packages up to date, and preventing malicious packages from entering your supply chain. This chapter covers tools, workflows, and the operational practices that make dependency security sustainable.

---

## Prerequisites

- Node.js package management (npm, pnpm, or Bun)
- Basic CI/CD pipeline understanding (GitHub Actions)
- Familiarity with `package.json` and lockfiles

---

## Core Concepts

### The dependency risk surface

```
Your code → direct dependencies → transitive dependencies → ... → OS packages
```

A typical Next.js application has ~1,000 packages in `node_modules`. You directly control ~50; the rest are pulled in transitively. A vulnerability in any of them can affect you.

### Vulnerability severity levels

| Level | CVSS Score | Example | Action |
|-------|-----------|---------|--------|
| Critical | 9.0–10.0 | Remote code execution | Fix immediately |
| High | 7.0–8.9 | Auth bypass, data exposure | Fix within 24 hours |
| Medium | 4.0–6.9 | Information disclosure | Fix within a sprint |
| Low | 0.1–3.9 | Minor issues | Track, fix when possible |

---

## Hands-On Examples

### npm audit

```bash
# Run a basic audit
npm audit

# Only report high and critical
npm audit --audit-level=high

# Fix automatically fixable issues
npm audit fix

# Fix including breaking changes (major version bumps)
npm audit fix --force

# Output JSON for CI parsing
npm audit --json
```

Sample output:

```
found 3 vulnerabilities (1 moderate, 2 high)
  run `npm audit fix` to fix 1 of them.
  2 vulnerabilities require manual review.
  See the full report for details.
```

### CI integration with GitHub Actions

```yaml
# .github/workflows/security.yml
name: Security Audit

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 8 * * 1' # every Monday at 8am UTC

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Install dependencies
        run: npm ci

      - name: Run security audit
        run: npm audit --audit-level=high
        # Fails the build on high or critical vulnerabilities
```

### Snyk — advanced vulnerability scanning

Snyk goes beyond `npm audit` — it checks for vulnerabilities that are not yet in the npm advisory database and provides remediation advice.

```bash
# Install Snyk CLI
npm install -g snyk

# Authenticate
snyk auth

# Test for vulnerabilities
snyk test

# Test and fail on high severity
snyk test --severity-threshold=high

# Monitor your project (reports to Snyk dashboard)
snyk monitor
```

```yaml
# GitHub Actions with Snyk
- name: Run Snyk vulnerability scan
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

### Dependabot — automated dependency updates

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "08:00"
      timezone: "America/Sao_Paulo"
    groups:
      # Group patch and minor updates together to reduce PR noise
      patch-and-minor:
        update-types:
          - "minor"
          - "patch"
    ignore:
      # Major version bumps require manual review
      - dependency-name: "*"
        update-types: ["version-update:semver-major"]
    reviewers:
      - "your-github-username"
    labels:
      - "dependencies"
```

Dependabot opens PRs automatically when updates are available. Group updates by patch/minor to keep PR volume manageable.

### Checking for lockfile integrity

```bash
# npm ci uses package-lock.json exactly — fails if it's inconsistent
npm ci

# Never use npm install in CI — it can update the lockfile
```

Always commit your lockfile (`package-lock.json`, `pnpm-lock.yaml`, `bun.lockb`) to version control. Without a lockfile, `npm install` can install different versions on different machines.

### Auditing licenses

Not all dependencies are safe to use in commercial products. Check licenses:

```bash
# license-checker lists all dependency licenses
npx license-checker --onlyAllow 'MIT;ISC;BSD-2-Clause;BSD-3-Clause;Apache-2.0;CC0-1.0;0BSD'
```

```typescript
// package.json script
{
  "scripts": {
    "check:licenses": "license-checker --onlyAllow 'MIT;ISC;BSD-2-Clause;BSD-3-Clause;Apache-2.0'"
  }
}
```

GPL-licensed dependencies in a commercial closed-source application can create legal obligations.

### Supply chain attack prevention

Supply chain attacks (like the `event-stream` incident and the `node-ipc` sabotage) introduce malicious code into packages you depend on.

Mitigations:

```bash
# Pin exact versions in package.json (no ^ or ~ ranges)
{
  "dependencies": {
    "fastify": "4.28.1",  // exact — not "^4.28.1"
    "zod": "3.23.8"
  }
}
```

```bash
# Verify package integrity after install
npm ci --ignore-scripts  # skip postinstall scripts
```

```bash
# Use npm pack to inspect what a package contains before installing
npm pack fastify --dry-run
```

```bash
# Check for typosquatting before installing a new package
# Verify package name, author, and weekly downloads on npmjs.com
```

### Finding outdated packages

```bash
# List outdated packages
npm outdated

# Update to the latest compatible versions
npx npm-check-updates -u --target minor
npm install
```

---

## Common Patterns & Best Practices

- **Run `npm audit` in CI** — fail on high severity, so vulnerabilities do not reach production
- **Enable Dependabot** — automate the discovery of dependency updates
- **Use `npm ci` in CI** — deterministic installs from lockfile; fails if lockfile is inconsistent
- **Review Dependabot PRs with your normal review process** — do not auto-merge without checking
- **Keep a monthly "dependency health" task** — update things Dependabot misses (major versions)
- **Check licenses** for commercial projects — GPL in a closed-source product is a legal risk
- **Audit new dependencies before adding them** — check npm downloads, GitHub stars, maintenance status

---

## Anti-Patterns to Avoid

- Not committing the lockfile — allows different versions to install on different machines
- Using `--legacy-peer-deps` to silence peer dependency warnings — hides real compatibility problems
- `npm audit fix --force` without reading what it changes — force can introduce breaking changes
- Ignoring Dependabot PRs indefinitely — they pile up and become overwhelming
- Installing packages with very few downloads or recent creation dates without verifying them
- Running `postinstall` scripts from third-party packages without review

---

## Debugging & Troubleshooting

**"npm audit reports vulnerabilities but fix doesn't resolve them"**
Some vulnerabilities are in transitive dependencies that cannot be updated without breaking the direct dependency. Options:
1. Check if the direct dependency has released a fix — update it
2. Use `overrides` in `package.json` to pin a safe version of the transitive dep:

```json
{
  "overrides": {
    "lodash": "^4.17.21"
  }
}
```

3. File an issue with the direct dependency maintainer

**"Dependabot opens 50 PRs at once"**
Configure grouping in `dependabot.yml` to batch patch/minor updates, and set a `daily` or `weekly` schedule instead of `daily` open.

**"License checker fails on a package we need"**
Check the package's actual license terms — some are permissive despite the label. If needed, contact the package author. As a last resort, use a fork with a license change.

---

## Real-World Scenarios

**Scenario: Responding to a critical CVE**

1. `npm audit --json` — identify affected packages and CVE
2. Check if a fix is available: `npm audit fix`
3. If not: find the advisory, check if there is a workaround
4. Check your code: does it use the vulnerable code path?
5. If vulnerable and no fix: isolate the package behind a service boundary, notify stakeholders
6. Monitor the package's GitHub for a fix, subscribe to release notifications

**Scenario: Zero-dependency policy for security-critical code**

For cryptographic operations, authentication handlers, and payment flows, minimize dependencies:

```typescript
// Use Node.js built-ins for crypto — no third-party dependencies
import { createHash, randomBytes, timingSafeEqual } from 'crypto';

// Only use well-established, widely audited packages for password hashing
import bcrypt from 'bcrypt'; // 10M+ weekly downloads, actively maintained, security audits
```

---

## Further Reading

- [npm Audit documentation](https://docs.npmjs.com/cli/v10/commands/npm-audit)
- [Snyk Node.js security guide](https://snyk.io/learn/nodejs-security/)
- [GitHub Dependabot documentation](https://docs.github.com/en/code-security/dependabot)
- [OWASP Software Composition Analysis](https://owasp.org/www-community/Component_Analysis)
- [Supply chain security — SLSA framework](https://slsa.dev/)

---

## Summary

Dependency auditing is a continuous process, not a one-time task. The baseline is `npm audit` in CI with `--audit-level=high` — any high or critical vulnerability fails the build. Dependabot automates the discovery of updates and opens PRs on a schedule, keeping your dependencies from drifting dangerously out of date. For commercial projects, license auditing is equally important. The deeper practice — pinning exact versions, auditing new packages before adding them, and reviewing Dependabot PRs carefully — transforms dependency management from a reactive scramble into a controlled, low-risk workflow.
