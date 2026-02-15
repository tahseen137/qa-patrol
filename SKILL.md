---
name: qa-patrol
version: 1.0.2
description: >
  Automated QA testing skill for web apps using local browser automation.
  Performs smoke tests, auth flow verification, payment integration checks, and cross-platform issue detection.
  All tests run locally via browser automation — no data leaves your machine. Static analysis and DB checks
  are OPTIONAL Level 3 features requiring explicit user configuration. Supports Supabase/Firebase auth,
  Stripe payments, React Native Web, Next.js, and SPAs.
tags: [qa, testing, automation, browser, supabase, stripe, react-native-web]
metadata:
  openclaw:
    emoji: "🔍"
    permissions:
      - browser  # Browser automation for testing
      - read     # Optional: local file access for static analysis (Level 3)
    requires:
      env:
        # All environment variables are OPTIONAL and only needed for Level 2+ auth testing
        # Secrets use ${env.VAR} interpolation — NEVER hardcoded in test plans
        - name: ADMIN_EMAIL
          required: false
          description: "Admin account email for auth testing (Level 2+)"
        - name: ADMIN_PASSWORD
          required: false
          description: "Admin account password for auth testing (Level 2+)"
        - name: FREE_EMAIL
          required: false
          description: "Free-tier account email for auth testing (Level 2+)"
        - name: FREE_PASSWORD
          required: false
          description: "Free-tier account password for auth testing (Level 2+)"
        - name: PRO_EMAIL
          required: false
          description: "Pro account email for subscription testing (Level 2+)"
        - name: PRO_PASSWORD
          required: false
          description: "Pro account password for subscription testing (Level 2+)"
        - name: DATABASE_URL
          required: false
          description: "Database connection string for data integrity checks (Level 3 only)"
        - name: APP_URL
          required: false
          description: "Target application URL (can also be passed via --url flag)"
---

# QA Patrol

Automated QA testing skill for web applications. Catches bugs that unit tests miss: cross-platform issues, auth state problems, data integrity failures, and integration breakages.

## Security & Privacy

This skill is designed with security and transparency in mind:

### Local Execution
- **All tests run locally on your machine. Nothing is sent to external servers. The browser automation uses OpenClaw's built-in browser control — no cloud services involved.**
- All browser automation runs entirely on your machine
- No data leaves your local machine during testing
- Test results and screenshots stay local unless you explicitly share them

### Secrets Handling
- **NEVER hardcode secrets** in test plans — always use environment variable interpolation: `${env.ADMIN_PASSWORD}`
- Credentials are read from your local environment at runtime
- Test plans in this skill's examples use only `${env.VAR}` placeholders
- The skill does not persist, log, or transmit credentials

### Optional Advanced Features (Level 3)
The following features are **completely optional** and require explicit user configuration:
- **Static Analysis**: Scans YOUR codebase (if you provide `repo_path`) for common bug patterns
- **Data Integrity Checks**: Queries YOUR database (if you provide `DATABASE_URL`) to verify UI accuracy
- These features help YOU find bugs in YOUR code — they don't exfiltrate anything

### Security Pattern Detection (Not Exploitation)
The `references/bug-patterns.md` file contains regex patterns for **detecting exposed secrets** in codebases (e.g., `sk_live_`, `api_key=`). These are **detection patterns** used to help developers find and fix security issues — they are NOT exploitation tools. This is standard practice in security linters like ESLint, Semgrep, and GitHub's secret scanning.

### User Control
- All test plans are **user-provided and user-reviewed**
- The skill only tests URLs and credentials that YOU explicitly configure
- No automatic scanning of external resources

## Quick Start

### Level 1: Zero-Config Smoke Test
```bash
# Just provide a URL
qa-patrol https://example.com
```

### Level 2: With Auth/Payments
```bash
# Use a test plan template
qa-patrol --plan auth-supabase.yaml --url https://example.com
```

### Level 3: Full Config
```bash
# Custom test plan with data integrity checks
qa-patrol --plan my-app.yaml
```

## Workflow

### 1. Load or Generate Test Plan

If a YAML test plan is provided, load it. Otherwise, generate a basic plan:

```yaml
app:
  url: <provided URL>
  name: <extracted from page title>

tests:
  smoke:
    - name: Homepage loads
      navigate: /
      assert:
        - element_exists: main
        - no_console_errors: true
```

See `assets/templates/` for test plan templates:
- `basic.yaml` - Zero-config smoke test
- `auth-supabase.yaml` - Supabase auth flows
- `payments-stripe.yaml` - Stripe checkout testing
- `full-saas.yaml` - Complete SaaS test plan

### 2. Execute Tests

Run tests in order: smoke → auth → payments → data_integrity → static_analysis.

For each test:
1. Navigate to the target URL
2. Execute steps (click, type, wait)
3. Capture snapshot and console logs
4. Evaluate assertions
5. Record PASS/FAIL/SKIP with evidence

#### Browser Automation Patterns

```python
# Navigate and snapshot
browser(action="navigate", targetUrl="https://example.com/page")
browser(action="snapshot")

# Form interaction
browser(action="act", request={"kind": "click", "ref": "email_input"})
browser(action="act", request={"kind": "type", "ref": "email_input", "text": "user@test.com"})
browser(action="act", request={"kind": "click", "ref": "submit_button"})

# Check console for errors
browser(action="console", level="error")
```

See `references/test-patterns.md` for complete automation patterns.

### 3. Check for Known Bug Patterns

Scan codebase (if accessible) for anti-patterns:

| Pattern | What to grep | Severity |
|---------|-------------|----------|
| Alert.alert on web | `Alert.alert` without `Platform.OS` guard | High |
| Linking in Modal | `Linking.openURL` inside Modal component | High |
| Missing RLS | Supabase queries without proper auth context | High |
| Hardcoded secrets | API keys in client code | Critical |

See `references/bug-patterns.md` for the full catalog.

### 4. Data Integrity Checks (Level 3)

When `data_integrity` tests are defined:

1. Execute the DB query (requires DB access)
2. Navigate to the UI path
3. Extract the displayed value
4. Compare against query result
5. Flag mismatches with severity based on % difference

### 5. Generate Report

Output a structured report:

```markdown
# QA Report: [App Name]
**Date**: YYYY-MM-DD HH:MM
**URL**: https://example.com
**Confidence**: 87%

## Summary
| Category | Pass | Fail | Skip |
|----------|------|------|------|
| Smoke    | 5    | 0    | 0    |
| Auth     | 3    | 1    | 0    |
| Payments | 0    | 0    | 2    |

## Failures

### [FAIL] Auth: Session persistence after refresh
**Steps**: Sign in → Refresh page → Check auth state
**Expected**: User remains signed in
**Actual**: Redirected to login page
**Evidence**: [screenshot]
**Severity**: High

## Recommendations
1. Fix session persistence (likely cookie/localStorage issue)
2. Add Platform.OS guards to Alert.alert calls
```

See `references/report-format.md` for the complete template.

## Test Plan Reference

### App Configuration

```yaml
app:
  url: https://example.com      # Required: base URL
  name: My App                  # Optional: display name
  stack: expo-web               # expo-web | nextjs | spa | static
```

### Auth Configuration

```yaml
auth:
  provider: supabase            # supabase | firebase | auth0 | custom
  login_path: /auth             # Path to login page
  accounts:
    admin:
      email: admin@test.com
      password: ${ADMIN_PASSWORD}  # Use env vars for secrets
    free:
      email: free@test.com
      password: ${FREE_PASSWORD}
    guest: true                 # Test anonymous/guest mode
```

### Test Types

#### Smoke Tests
```yaml
tests:
  smoke:
    - name: Homepage loads
      navigate: /
      assert:
        - element_exists: main
        - no_console_errors: true
        - no_network_errors: true
    
    - name: Navigation works
      navigate: /
      steps:
        - click: { ref: nav_link }
        - assert: { url_contains: "/target" }
```

#### Auth Tests
```yaml
tests:
  auth:
    - name: Sign in flow
      steps:
        - navigate: /auth
        - type: { ref: email_input, text: "${auth.accounts.free.email}" }
        - type: { ref: password_input, text: "${auth.accounts.free.password}" }
        - click: { ref: sign_in_button }
        - wait: { url_contains: "/home", timeout: 5000 }
        - assert: { element_exists: "user_avatar" }
    
    - name: Sign out flow
      requires: signed_in
      steps:
        - click: { ref: user_menu }
        - click: { ref: sign_out_button }
        - assert: { url_contains: "/auth" }
    
    - name: Session persistence
      requires: signed_in
      steps:
        - navigate: /home
        - refresh: true
        - assert: { element_exists: "user_avatar" }
```

#### Payment Tests
```yaml
tests:
  payments:
    provider: stripe
    tests:
      - name: Checkout creation
        steps:
          - navigate: /pricing
          - click: { ref: pro_plan_button }
          - wait: { url_contains: "checkout.stripe.com", timeout: 10000 }
          - assert: { element_exists: "cardNumber" }
```

#### Data Integrity Tests
```yaml
tests:
  data_integrity:
    - name: Card count matches
      query: "SELECT count(*) FROM cards WHERE country='CA'"
      ui_path: /settings
      ui_selector: "[data-testid='card-count']"
      tolerance: 0  # Exact match required
    
    - name: Points calculation
      query: "SELECT points_rate FROM tiers WHERE name='Gold'"
      ui_path: /calculator
      ui_selector: ".points-display"
      tolerance: 0.01  # 1% tolerance
```

#### Static Analysis
```yaml
tests:
  static_analysis:
    scan_path: ./src
    patterns:
      - name: Alert.alert without Platform guard
        grep: "Alert\\.alert"
        exclude_grep: "Platform\\.OS"
        severity: high
        fix_hint: "Wrap in Platform.OS check or use cross-platform alert"
      
      - name: Hardcoded API keys
        grep: "(sk_live_|pk_live_|api_key.*=.*['\"][a-zA-Z0-9]{20,})"
        severity: critical
```

### Assertions Reference

| Assertion | Description |
|-----------|-------------|
| `element_exists: "ref"` | Element with ref is in DOM |
| `element_visible: "ref"` | Element is visible |
| `text_contains: "string"` | Page contains text |
| `url_contains: "/path"` | URL includes path |
| `no_console_errors: true` | No console.error calls |
| `no_network_errors: true` | No failed network requests |
| `value_equals: { ref, value }` | Input value matches |
| `count_equals: { ref, count }` | Number of matching elements |

### Variable Interpolation

Use `${...}` for dynamic values:
- `${auth.accounts.free.email}` - From test plan
- `${env.API_KEY}` - From environment
- `${captured.user_id}` - From previous step capture

## Confidence Scoring

Calculate confidence based on test coverage and results:

```
base_confidence = 50
per_smoke_pass = +5 (max 20)
per_auth_pass = +8 (max 24)
per_payment_pass = +10 (max 20)
per_data_check_pass = +6 (max 18)
static_analysis_clean = +8
no_critical_failures = +10

final_confidence = min(base + bonuses - penalties, 100)
```

Penalties:
- Critical failure: -20
- High severity failure: -10
- Medium severity failure: -5
- Skipped critical test: -5

## Files

### References
- `references/test-patterns.md` - Browser automation patterns and examples
- `references/bug-patterns.md` - Known bug patterns to detect
- `references/report-format.md` - QA report template

### Templates
- `assets/templates/basic.yaml` - Zero-config smoke test
- `assets/templates/auth-supabase.yaml` - Supabase auth testing
- `assets/templates/payments-stripe.yaml` - Stripe payment testing
- `assets/templates/full-saas.yaml` - Complete SaaS test plan

### Examples
- `assets/examples/rewardly.yaml` - Real-world React Native Web app test plan

## Tips

1. **Start with smoke tests** - Verify basic functionality before auth/payments
2. **Use guest mode first** - Test without auth to establish baseline
3. **Check console early** - Console errors often reveal root causes
4. **Screenshot failures** - Always capture evidence for debugging
5. **Test cache states** - Sign out and clear cache to expose hidden issues
6. **Verify cross-platform** - If React Native Web, test alert/linking patterns
