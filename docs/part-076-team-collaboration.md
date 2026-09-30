# Part 76: Team Collaboration

## ความสำคัญของ Team Collaboration

การทำงานร่วมกันเป็น team ที่ดีช่วยให้ code quality สูงขึ้น delivery เร็วขึ้น และลด bugs ในระยะยาว

---

## 1. Git Workflow Strategies

### GitFlow

```
main ──────────────────────────────────── v1.0 ── v2.0
         ↑                                   ↑
hotfix/fix-payment                    release/1.0
         ↑                             ↑
develop ─────────────────────────────────────────────
         ↑           ↑
feature/auth    feature/dashboard
```

```bash
# GitFlow commands
git flow init

# Feature branch
git flow feature start user-authentication
git flow feature finish user-authentication

# Release branch
git flow release start 1.0.0
git flow release finish 1.0.0

# Hotfix
git flow hotfix start fix-payment-bug
git flow hotfix finish fix-payment-bug
```

### Trunk-based Development (แนะนำสำหรับ CI/CD)

```bash
# ทุกคน commit ลง main โดยตรง หรือ short-lived feature branches
# Branch อายุไม่เกิน 1-2 วัน

# Feature flags ใช้แทน long-running branches
git checkout -b feat/new-checkout-flow
# ทำงาน...
git push origin feat/new-checkout-flow
# PR และ merge ภายใน 1-2 วัน
```

---

## 2. Commit Convention

### Conventional Commits

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

### ประเภท Commits

```bash
# feat: feature ใหม่
git commit -m "feat(auth): add OAuth2 with Google"

# fix: แก้บัก
git commit -m "fix(billing): correct Stripe webhook signature validation"

# docs: แก้ documentation
git commit -m "docs(api): update endpoint documentation for v2"

# style: code style (ไม่เปลี่ยน logic)
git commit -m "style(components): format with prettier"

# refactor: refactoring โดยไม่เพิ่ม feature
git commit -m "refactor(auth): extract token validation to middleware"

# perf: performance improvement
git commit -m "perf(queries): add index for user search"

# test: เพิ่ม tests
git commit -m "test(auth): add unit tests for login flow"

# chore: maintenance tasks
git commit -m "chore(deps): upgrade nuxt to 3.9.0"

# ci: CI configuration
git commit -m "ci: add automated testing to PR workflow"

# BREAKING CHANGE
git commit -m "feat(api)!: change user endpoint response format

BREAKING CHANGE: User response now uses camelCase fields
instead of snake_case. Update client code accordingly."
```

### Commitlint Setup

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'perf', 'test', 'chore', 'ci', 'revert'
    ]],
    'scope-case': [2, 'always', 'kebab-case'],
    'subject-max-length': [2, 'always', 100],
    'body-max-line-length': [2, 'always', 120]
  }
}
```

---

## 3. Branch Strategy

### Branch Naming Convention

```bash
# Features
feat/TICKET-123-user-authentication
feat/add-dark-mode

# Bug fixes  
fix/TICKET-456-payment-error
fix/correct-email-validation

# Hotfixes (urgent)
hotfix/critical-security-patch

# Releases
release/v2.3.0
release/2024-q1

# Experiments
experiment/new-onboarding-flow

# Chores
chore/update-dependencies
chore/cleanup-unused-components
```

### Branch Protection Rules (GitHub)

```yaml
# .github/branch-protection.yml (ใช้กับ GitHub API)
protection_rules:
  main:
    required_status_checks:
      strict: true
      contexts:
        - "ci/tests"
        - "ci/lint"
        - "ci/type-check"
        - "ci/build"
    enforce_admins: true
    required_pull_request_reviews:
      required_approving_review_count: 2
      dismiss_stale_reviews: true
      require_code_owner_reviews: true
    restrictions:
      users: []
      teams: ["senior-engineers"]
    allow_force_pushes: false
    allow_deletions: false
```

---

## 4. Pull Request Templates

```markdown
<!-- .github/pull_request_template.md -->
## Description
<!-- What does this PR do? -->

## Type of Change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to change)
- [ ] Documentation update
- [ ] Performance improvement
- [ ] Refactoring

## Related Issues
Closes #<!-- issue number -->

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manually tested in dev environment
- [ ] Tested edge cases

## Screenshots (if UI changes)
<!-- Add screenshots here -->

## Checklist
- [ ] Code follows team style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated
- [ ] No sensitive data exposed
- [ ] Performance impact considered

## Deployment Notes
<!-- Any special deployment steps? Database migrations? -->
```

---

## 5. Semantic Versioning

### Version Format

```
MAJOR.MINOR.PATCH

1.0.0
│ │ └── PATCH: Bug fixes, backward compatible
│ └──── MINOR: New features, backward compatible
└────── MAJOR: Breaking changes
```

### package.json Version Management

```json
{
  "version": "2.1.3",
  "scripts": {
    "version:patch": "npm version patch",
    "version:minor": "npm version minor",
    "version:major": "npm version major",
    "release": "standard-version",
    "release:minor": "standard-version --release-as minor",
    "release:major": "standard-version --release-as major"
  }
}
```

---

## 6. Changelog Management

### CHANGELOG.md Format

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- AI-powered code suggestions

## [2.1.0] - 2024-01-15

### Added
- New dashboard with real-time analytics
- Dark mode support
- Bulk user management

### Changed
- Improved performance of user search (3x faster)
- Updated API response format for consistency

### Fixed
- Fixed payment webhook validation (#234)
- Corrected email notification for password reset

### Security
- Updated dependencies to fix CVE-2024-1234

## [2.0.0] - 2023-12-01

### Breaking Changes
- API v1 endpoints removed. Please migrate to v2
- User response format changed to camelCase

### Added
- Multi-tenant support
- Stripe billing integration
```

### Automated Changelog กับ standard-version

```javascript
// .versionrc.js
module.exports = {
  types: [
    { type: 'feat', section: '✨ Features' },
    { type: 'fix', section: '🐛 Bug Fixes' },
    { type: 'perf', section: '⚡ Performance' },
    { type: 'refactor', section: '♻️ Refactoring' },
    { type: 'docs', section: '📚 Documentation' },
    { type: 'chore', hidden: true },
    { type: 'style', hidden: true },
    { type: 'test', hidden: true }
  ],
  commitUrlFormat: 'https://github.com/myorg/myapp/commit/{{hash}}',
  compareUrlFormat: 'https://github.com/myorg/myapp/compare/{{previousTag}}...{{currentTag}}'
}
```

---

## 7. ตัวอย่าง: Team Git Workflow Setup

### GitHub Actions สำหรับ PR Validation

```yaml
# .github/workflows/pr-validation.yml
name: PR Validation

on:
  pull_request:
    branches: [main, develop]

jobs:
  lint-commit-messages:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - name: Lint commits
        run: npx commitlint --from ${{ github.event.pull_request.base.sha }} --to ${{ github.event.pull_request.head.sha }}

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run test -- --coverage
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build
          path: .output/

  size-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci && npm run build
      - name: Check bundle size
        run: npx bundlesize
```

### CODEOWNERS

```
# .github/CODEOWNERS
# Global owners
* @team-lead @senior-dev

# Frontend
/components/ @frontend-team
/pages/ @frontend-team
/stores/ @frontend-team

# Backend
/server/ @backend-team
/prisma/ @backend-team @dba-team

# Infrastructure
/docker/ @devops-team
/.github/ @devops-team @team-lead

# Security-sensitive
/server/middleware/auth.ts @security-team @team-lead
/server/api/billing/ @backend-team @team-lead
```

### Git Hooks กับ Husky

```json
// package.json
{
  "scripts": {
    "prepare": "husky install"
  },
  "lint-staged": {
    "*.{ts,vue}": ["eslint --fix", "prettier --write"],
    "*.{json,md,yml}": ["prettier --write"]
  }
}
```

```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# Run lint-staged
npx lint-staged

# Run type check
npm run type-check

# .husky/commit-msg
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# Validate commit message
npx commitlint --edit "$1"
```

---

## สรุป

Team collaboration ที่ดีต้องการ:
1. **Clear Git workflow** - ทุกคนเข้าใจและปฏิบัติตาม
2. **Commit conventions** - ทำให้ history อ่านง่าย
3. **PR templates** - รีวิวได้ง่ายขึ้น
4. **Automated checks** - CI/CD ช่วย enforce standards
5. **Semantic versioning** - version ที่สื่อความหมาย
6. **Documentation** - เพื่อ onboard members ใหม่ได้เร็ว
