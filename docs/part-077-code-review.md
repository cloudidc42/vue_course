# Part 77: Code Review Best Practices

## Code Review คืออะไร?

Code Review คือกระบวนการที่ developer คนอื่นตรวจสอบ code ของเราก่อน merge เพื่อหา bugs, ปรับปรุง quality และแชร์ knowledge ในทีม

---

## 1. Code Review Culture

### วัฒนธรรมที่ดี

```
✅ Review = Learning opportunity
✅ Comments = Suggestions, not commands
✅ Disagree but commit
✅ Empathy first
✅ "We" not "You"

❌ Personal attacks
❌ Nitpicking trivial style issues
❌ Blocking for weeks
❌ Rubber stamping (approve without reading)
```

### Author Responsibilities

- เขียน description ที่ชัดเจนว่าทำอะไร ทำไม
- ทำ PR ให้เล็ก - อ่านได้ภายใน 30 นาที
- Self-review ก่อน assign reviewer
- Response to comments ภายใน 1 business day

### Reviewer Responsibilities

- Review ภายใน 1 business day
- ให้ feedback ที่ constructive และ actionable
- อ้างอิง standard/guideline เมื่อ reject
- Approve เมื่อ satisfied ไม่ต้องรอ perfect

---

## 2. สิ่งที่ต้องมองหาใน Code Review

### Correctness

```typescript
// ❌ Bug: ไม่ handle null/undefined
function getUserDisplayName(user: User) {
  return user.profile.firstName + ' ' + user.profile.lastName
  // Error ถ้า profile เป็น null
}

// ✅ Handle null properly
function getUserDisplayName(user: User) {
  if (!user.profile) return user.email
  const { firstName, lastName } = user.profile
  return [firstName, lastName].filter(Boolean).join(' ') || user.email
}
```

### Security

```typescript
// ❌ SQL Injection
const query = `SELECT * FROM users WHERE id = ${userId}`

// ✅ Parameterized query
const user = await prisma.user.findUnique({ where: { id: userId } })

// ❌ XSS vulnerability
element.innerHTML = userInput

// ✅ Safe
element.textContent = userInput

// ❌ Exposed sensitive data
return { user, password: user.passwordHash }

// ✅ Exclude sensitive fields
const { passwordHash, ...safeUser } = user
return { user: safeUser }
```

### Performance

```typescript
// ❌ Unnecessary re-renders
const Component = () => {
  const data = heavyComputation() // Run ทุก render
  return <div>{data}</div>
}

// ✅ Memoize
const data = computed(() => heavyComputation())

// ❌ Missing await
async function processUsers(users: User[]) {
  users.forEach(user => processUser(user)) // Missing await!
}

// ✅ Correct async handling
async function processUsers(users: User[]) {
  await Promise.all(users.map(user => processUser(user)))
}
```

### Maintainability

```typescript
// ❌ Magic numbers
if (score > 42) { // What is 42?
  grantAccess()
}

// ✅ Named constants
const MINIMUM_TRUST_SCORE = 42
if (score > MINIMUM_TRUST_SCORE) {
  grantAccess()
}

// ❌ Long function doing too much
async function handleUserRegistration(data) {
  // 200 lines of code...
}

// ✅ Small, focused functions
async function handleUserRegistration(data) {
  const validated = await validateRegistrationData(data)
  const user = await createUser(validated)
  await sendWelcomeEmail(user)
  await createDefaultWorkspace(user)
  return user
}
```

---

## 3. Giving Constructive Feedback

### Comment Types (Conventional Comments)

```
# Prefixes ที่ช่วยสื่อสาร intent
praise: โชว์ความประทับใจ
nitpick: เล็กน้อย ไม่บังคับแก้
suggestion: แนะนำ บางครั้งก็ดี
issue: ปัญหาที่ต้องแก้
question: ถามความเข้าใจ
thought: ความคิด ไม่จำเป็น
todo: ทำทีหลังได้
```

### ตัวอย่าง Comments ที่ดี

```
# ❌ ไม่ดี - คลุมเครือและ personal
"This code is terrible"
"Why would you do this?"

# ✅ ดี - ชัดเจนและ actionable
suggestion: Consider using `Promise.all()` here for parallel execution.
This would reduce the total time from ~300ms to ~100ms.

# Example:
const [users, projects] = await Promise.all([
  getUsers(),
  getProjects()
])

# ❌ ไม่ดี - ไม่บอกเหตุผล
issue: Change this

# ✅ ดี - อธิบายเหตุผล
issue: This will throw a TypeError if `user.profile` is null.
Users who haven't completed their profile will see a crash.

suggestion: Add a null check:
return user.profile?.firstName ?? user.email
```

---

## 4. Automated Code Review

### ESLint Configuration

```javascript
// eslint.config.js (ESLint 9)
import pluginVue from 'eslint-plugin-vue'
import pluginTs from '@typescript-eslint/eslint-plugin'
import parserTs from '@typescript-eslint/parser'
import pluginSecurity from 'eslint-plugin-security'

export default [
  {
    files: ['**/*.{ts,vue}'],
    plugins: {
      vue: pluginVue,
      '@typescript-eslint': pluginTs,
      security: pluginSecurity
    },
    languageOptions: {
      parser: parserTs,
      parserOptions: { ecmaVersion: 'latest' }
    },
    rules: {
      // TypeScript
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/no-unused-vars': 'error',
      '@typescript-eslint/prefer-nullish-coalescing': 'error',
      '@typescript-eslint/no-non-null-assertion': 'warn',
      
      // Vue
      'vue/component-name-in-template-casing': ['error', 'PascalCase'],
      'vue/no-unused-vars': 'error',
      'vue/prefer-import-from-vue': 'error',
      'vue/define-props-declaration': ['error', 'type-based'],
      
      // Security
      'security/detect-sql-injection': 'error',
      'security/detect-non-literal-regexp': 'warn',
      
      // General
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'no-debugger': 'error',
      'prefer-const': 'error'
    }
  }
]
```

### SonarQube Integration

```yaml
# sonar-project.properties
sonar.projectKey=my-nuxt-app
sonar.projectName=My Nuxt App
sonar.sources=.
sonar.exclusions=node_modules/**,dist/**,.nuxt/**
sonar.tests=tests
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.typescript.tsconfigPath=tsconfig.json

# Quality Gates
sonar.qualitygate.wait=true
```

```yaml
# .github/workflows/sonarqube.yml
name: SonarQube Analysis

on: [push, pull_request]

jobs:
  sonarqube:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

---

## 5. Review Checklist สำหรับ Vue/Nuxt

### Vue Component Checklist

```markdown
## Vue Component Review Checklist

### Reactivity
- [ ] ใช้ ref/reactive ถูกต้อง
- [ ] ไม่ destructure reactive object โดยตรง
- [ ] computed properties ใช้สำหรับ derived state
- [ ] watchers ใช้เฉพาะเมื่อจำเป็น

### Props & Events
- [ ] Props มี type definitions ครบ
- [ ] Props validation มี default values
- [ ] ไม่ mutate props โดยตรง
- [ ] emit events มี type definitions

### Composables
- [ ] Composables return reactive values
- [ ] Lifecycle hooks cleanup ใน onUnmounted
- [ ] ไม่มี memory leaks

### Performance
- [ ] v-for มี :key ที่ unique
- [ ] ไม่ใช้ v-if กับ v-for ใน element เดียวกัน
- [ ] ใช้ lazy loading สำหรับ heavy components

### Security
- [ ] ไม่ใช้ v-html กับ user input
- [ ] Sanitize data ก่อน render
- [ ] ไม่เปิดเผย sensitive data ใน template

### Accessibility
- [ ] Images มี alt text
- [ ] Forms มี label
- [ ] Interactive elements มี keyboard support
- [ ] Color contrast เพียงพอ
```

---

## 6. ตัวอย่าง: Review Checklist สมบูรณ์

```typescript
// review-checklist.ts - TypeScript checklist ที่ใช้ใน CI

interface ReviewCheck {
  id: string
  category: string
  description: string
  severity: 'error' | 'warning' | 'info'
  autoCheckable: boolean
}

export const REVIEW_CHECKLIST: ReviewCheck[] = [
  // Security
  {
    id: 'sec-001',
    category: 'Security',
    description: 'No hardcoded secrets or API keys',
    severity: 'error',
    autoCheckable: true // detect-secrets tool
  },
  {
    id: 'sec-002',
    category: 'Security',
    description: 'User input is sanitized before storage',
    severity: 'error',
    autoCheckable: false
  },
  
  // Performance
  {
    id: 'perf-001',
    category: 'Performance',
    description: 'No N+1 query patterns',
    severity: 'warning',
    autoCheckable: false
  },
  
  // Tests
  {
    id: 'test-001',
    category: 'Testing',
    description: 'New features have corresponding tests',
    severity: 'warning',
    autoCheckable: true // Coverage threshold check
  }
]
```

### PR Review GitHub Action

```yaml
# .github/workflows/auto-review.yml
name: Automated Review Checks

on: pull_request

jobs:
  check-sensitive-files:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check for secrets
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}

  check-pr-size:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check PR size
        run: |
          CHANGED_LINES=$(git diff --stat HEAD~1 | tail -1 | awk '{print $4+$6}')
          echo "Changed lines: $CHANGED_LINES"
          if [ "$CHANGED_LINES" -gt 500 ]; then
            echo "::warning::PR is large ($CHANGED_LINES lines). Consider breaking it up."
          fi

  coverage-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run test -- --coverage
      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "::error::Coverage $COVERAGE% is below 80% threshold"
            exit 1
          fi
```

---

## สรุป

Code Review ที่ดี:
1. **Culture first** - สร้าง culture ของการเรียนรู้ ไม่ใช่การตัดสิน
2. **Small PRs** - ทำ PR เล็กๆ review ได้ง่ายกว่า
3. **Clear communication** - อธิบาย "ทำไม" ไม่ใช่แค่ "อะไร"
4. **Automate what you can** - ใช้ ESLint, SonarQube, CI checks
5. **Checklist** - มี checklist ที่ทีม agree กัน
6. **Timely reviews** - ตอบสนองภายใน 1 business day
