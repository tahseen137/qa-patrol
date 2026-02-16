# 🛡️ QA Patrol

**Automated QA testing for web apps — catches the bugs unit tests miss.**

[![ClawHub](https://img.shields.io/badge/ClawHub-qa--patrol-blue)](https://clawhub.ai/tahseen137/qa-patrol)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.3-blue)](https://github.com/tahseen137/qa-patrol)

---

## 🎯 The Problem

We had **1,187 passing unit tests**. Our CEO tested the app for 10 minutes and found it _"still not fully usable."_

The bugs were in:
- **Cross-platform behavior** — `Alert.alert` callbacks don't fire on web, `Linking.openURL` fails inside modals
- **Auth state** — Supabase RLS policies blocked signed-in users (worked for anonymous!)
- **Data integrity** — Database values didn't match UI (12x inflation on rewards rates)
- **Payment integrations** — Stripe ES256 JWT rejection broke all checkouts
- **Cache masking** — Stale cache hid broken data for hours

**None of these are catchable by unit tests.** So we built QA Patrol.

---

## 🚀 Why QA Patrol?

| Feature | QA Patrol | Playwright | Cypress | BrowserStack | QA Wolf |
|---------|-----------|------------|---------|--------------|---------|
| **Setup** | Zero-install ✨ | npm install | npm install | Cloud account | Managed service |
| **Format** | YAML | TypeScript/Python | JavaScript | Code | Managed |
| **Privacy** | 100% local 🔒 | Local | Local | ☁️ Cloud | ☁️ Cloud |
| **React Native Web** | ✅ Specialized | Generic | Generic | Generic | Generic |
| **Price** | **Free** | Free | $69+/mo | $39+/mo | **$279+/mo** |
| **AI Debugging** | ❌ | ❌ | Limited | ✅ | ✅ |
| **Learning Curve** | Low (YAML) | High (coding) | Medium | Medium | Zero (managed) |

### 🎖️ When to Choose QA Patrol

- ✅ You use **OpenClaw** and already have browser automation
- ✅ You need **privacy-first testing** (data never leaves your machine)
- ✅ You're testing a **React Native Web** app with platform-specific bugs
- ✅ You prefer **declarative YAML** over writing test code
- ✅ You want **zero-cost** testing for side projects or MVPs

### 🔄 When to Use Others

- **Playwright** → Multi-browser testing, TypeScript/Python tests, CI/CD at scale
- **Cypress** → Visual debugging, component testing, existing JavaScript codebase
- **BrowserStack** → Need 3500+ browser/OS combos, mobile device testing
- **QA Wolf** → Enterprise budget, need human QA engineers on-call

---

## ⚡️ Quick Start

### Zero-Config Smoke Test
Just give OpenClaw a URL:
```
QA test https://your-app.com
```

### With Authentication
```
Run qa-patrol with the auth-supabase template on https://your-app.com
```

### Full Test Suite
```
Run qa-patrol with the full-saas template on https://your-app.com
```

---

## 📋 What QA Patrol Tests

### Test Levels

| Level | Template | What It Tests | Time |
|-------|----------|---------------|------|
| **1 - Smoke** | `basic.yaml` | Homepage loads, no JS errors, navigation works | ~30s |
| **2 - Auth** | `auth-supabase.yaml` | Sign in/out, session persistence, RLS policies | ~2min |
| **2 - Payments** | `payments-stripe.yaml` | Stripe checkout, pricing accuracy, webhooks | ~3min |
| **3 - Full** | `full-saas.yaml` | Auth + Payments + Data + Static Analysis + A11y | ~5min |

### Bug Pattern Detection

QA Patrol knows about **5 categories** of bugs that unit tests structurally can't catch:

| Category | Examples | Severity |
|----------|----------|----------|
| **Cross-Platform** | `Alert.alert` callbacks on web, `Linking.openURL` in modals | High |
| **Auth State** | RLS policies, session persistence, guest→user migration | Critical |
| **Data Integrity** | DB/UI mismatches, duplicate records, calculation errors | High |
| **Integration** | Stripe JWT auth, edge function failures, webhook reliability | Critical |
| **Cache Masking** | Stale data hiding real issues from signed-in users | Medium |

---

## 📦 Installation

### Via ClawHub (Recommended)
```bash
npx clawhub@latest install qa-patrol
```

### Manual
Clone this repo into your OpenClaw workspace skills folder:
```bash
git clone https://github.com/tahseen137/qa-patrol.git ~/.openclaw/workspace/skills/qa-patrol
```

---

## 📖 Usage Examples

### Example 1: Basic Smoke Test

**Test Plan** (`basic.yaml`):
```yaml
app:
  url: https://your-app.com
  name: My App

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
        - click: { ref: pricing_link }
      assert:
        - url_contains: /pricing
```

**Output**:
```markdown
# QA Report: My App
**Confidence**: 75%

## Summary
| Category | Pass | Fail |
|----------|------|------|
| Smoke    | 2    | 0    |

✅ All smoke tests passed!
```

---

### Example 2: Auth Testing (Supabase)

**Test Plan** (`auth-supabase.yaml` excerpt):
```yaml
app:
  url: https://your-app.com
  stack: expo-web

auth:
  provider: supabase
  login_path: /auth
  accounts:
    free:
      email: test@example.com
      password: ${FREE_PASSWORD}  # From environment

tests:
  auth:
    - name: Sign in flow
      steps:
        - navigate: /auth
        - type: { ref: email_input, text: "${auth.accounts.free.email}" }
        - type: { ref: password_input, text: "${auth.accounts.free.password}" }
        - click: { ref: sign_in_button }
        - wait: { url_contains: "/home", timeout: 5000 }
      assert:
        - element_exists: user_avatar
    
    - name: Session persists after refresh
      requires: signed_in
      steps:
        - navigate: /home
        - refresh: true
      assert:
        - element_exists: user_avatar
```

**Output**:
```markdown
## Auth Tests
| Test | Status | Duration |
|------|--------|----------|
| Sign in flow | ✅ PASS | 3.2s |
| Session persistence | ❌ FAIL | 2.1s |

### [FAIL] Session persistence
**Expected**: User remains signed in after refresh
**Actual**: Redirected to /auth (signed out)
**Console errors**: `Failed to restore session: Invalid refresh token`

**Recommendation**: Check token storage and refresh logic.
```

---

### Example 3: Payment Testing (Stripe)

**Test Plan** (`payments-stripe.yaml` excerpt):
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
        assert:
          - element_exists: "cardNumber"
      
      - name: Pricing accuracy
        steps:
          - navigate: /pricing
          - extract: { ref: pro_price, capture_as: ui_price }
        assert:
          - value_matches: { captured: ui_price, expected: "$49.00" }
```

**Output**:
```markdown
## Payment Tests
| Test | Status | Duration |
|------|--------|----------|
| Checkout creation | ✅ PASS | 4.2s |
| Pricing accuracy | ✅ PASS | 0.8s |
```

---

### Example 4: Data Integrity Testing

**Test Plan** (`full-saas.yaml` excerpt):
```yaml
tests:
  data_integrity:
    - name: Card count matches DB
      query: "SELECT count(*) FROM cards WHERE country='CA'"
      ui_path: /settings
      ui_selector: "[data-testid='card-count']"
      tolerance: 0  # Exact match required
    
    - name: Rewards multiplier accuracy
      query: "SELECT multiplier FROM cards WHERE name='Air Miles'"
      ui_path: /calculator/airmiles
      ui_selector: ".multiplier-display"
      tolerance: 0.01  # 1% tolerance
```

**Output**:
```markdown
## Data Integrity
| Test | Status | DB Value | UI Value | Diff |
|------|--------|----------|----------|------|
| Card count | ⚠️ FAIL | 220 | 225 | +5 (2.3%) |
| Rewards multiplier | ✅ PASS | 1.00 | 1.00 | 0% |

### [FAIL] Card count mismatch
**Severity**: Medium
**Possible causes**:
- Stale count cached in UI
- Different filtering logic between DB query and UI
- Recently deleted cards not reflected in UI
```

---

## 🔍 Static Analysis

QA Patrol can scan your codebase for known bug patterns:

**Patterns Detected**:
```bash
# High severity - Cross-platform issues
✅ Alert.alert without Platform guard: 3 files
✅ Linking.openURL in Modal: 1 file
❌ Exposed API keys: 0 files

# Medium severity
✅ Console.log in production: 12 files
✅ Hardcoded localhost: 2 files
```

**Example Output**:
```markdown
## Static Analysis Results

### [FOUND] Alert.alert without Platform guard (High)
**Files**:
- `src/components/DeleteConfirm.tsx:45`
- `src/screens/Settings.tsx:123`
- `src/screens/Profile.tsx:67`

**Issue**: Alert callbacks won't fire on web platform

**Fix**:
\`\`\`typescript
if (Platform.OS === 'web') {
  if (window.confirm(message)) onConfirm();
} else {
  Alert.alert(title, message, [
    { text: 'Cancel' },
    { text: 'OK', onPress: onConfirm }
  ]);
}
\`\`\`
```

---

## 📊 Sample Report

Full QA report generated by QA Patrol:

```markdown
# QA Report: Rewardly
**Date**: 2026-02-14  
**URL**: https://rewardly-cyan.vercel.app  
**Confidence**: 87%

## Summary
| Category | Pass | Fail | Skip | Pass Rate |
|----------|------|------|------|-----------|
| Smoke    | 5    | 0    | 0    | 100% |
| Auth     | 3    | 1    | 0    | 75% |
| Payments | 1    | 0    | 1    | 100% |
| Data     | 2    | 1    | 0    | 67% |
| Static   | 4    | 2    | 0    | 67% |
| **Total**| **15** | **4** | **1** | **79%** |

## Critical Findings
1. ❌ Session persistence broken (Auth)
2. ⚠️ Card count mismatch: UI shows 225, DB has 220 (Data)
3. ⚠️ 3 instances of Alert.alert without Platform guard (Static)

## Recommendations
1. Fix session token storage and refresh logic
2. Investigate card count discrepancy (cache issue?)
3. Add Platform.OS guards to Alert.alert calls
```

---

## 🏗️ Architecture

### File Structure

```
qa-patrol/
├── SKILL.md                          # Core workflow (329 lines)
├── README.md                         # This file
├── _meta.json                        # ClawHub metadata
├── references/
│   ├── bug-patterns.md               # 20+ known patterns from real testing
│   ├── test-patterns.md              # Browser automation playbook
│   └── report-format.md              # Structured QA report template
└── assets/
    ├── templates/
    │   ├── basic.yaml                # Level 1: Zero-config smoke test
    │   ├── auth-supabase.yaml        # Level 2: Supabase auth flows
    │   ├── payments-stripe.yaml      # Level 2: Stripe payment tests
    │   └── full-saas.yaml            # Level 3: Complete SaaS test plan
    └── examples/
        └── rewardly.yaml             # Real-world example from our app
```

### How It Works

1. **Agent receives command**: `QA test https://app.com`
2. **Loads test plan**: YAML template or auto-generates basic plan
3. **Executes tests**: Uses OpenClaw browser automation
4. **Captures evidence**: Screenshots, console logs, network errors
5. **Evaluates assertions**: Checks element existence, console errors, data integrity
6. **Detects patterns**: Scans for known bugs (optional, Level 3)
7. **Generates report**: Structured markdown with confidence score

---

## 🔒 Security & Privacy

### All Tests Run Locally
- ✅ **No cloud services** — Uses OpenClaw's built-in browser automation
- ✅ **No data exfiltration** — Your app data never leaves your machine
- ✅ **No third-party accounts** — Unlike BrowserStack or QA Wolf
- ✅ **Open source** — Audit the skill yourself (it's just markdown!)

### Environment Variables (All Optional)

| Variable | Purpose | Required |
|----------|---------|----------|
| `APP_URL` | Target app URL | No (can use `--url` flag) |
| `ADMIN_EMAIL` | Admin test account | Only for auth tests |
| `ADMIN_PASSWORD` | Admin password | Only for auth tests |
| `FREE_EMAIL` | Free tier test account | Only for auth tests |
| `FREE_PASSWORD` | Free tier password | Only for auth tests |
| `DATABASE_URL` | DB connection for data checks | Only for Level 3 |

**⚠️ Use test credentials only — never supply production passwords.**

### Secrets Handling
- **Never hardcode secrets** in test plans — always use `${env.VAR}` interpolation
- Credentials read from local environment at runtime
- Skill does not persist, log, or transmit credentials

---

## 🎓 Learning Resources

### Documentation
- [SKILL.md](SKILL.md) - Core workflow and test plan reference
- [test-patterns.md](references/test-patterns.md) - Browser automation patterns
- [bug-patterns.md](references/bug-patterns.md) - Known bug patterns catalog
- [report-format.md](references/report-format.md) - QA report structure

### Templates
- [basic.yaml](assets/templates/basic.yaml) - Smoke test template
- [auth-supabase.yaml](assets/templates/auth-supabase.yaml) - Supabase auth testing
- [payments-stripe.yaml](assets/templates/payments-stripe.yaml) - Stripe payment testing
- [full-saas.yaml](assets/templates/full-saas.yaml) - Complete SaaS QA plan

### Examples
- [rewardly.yaml](assets/examples/rewardly.yaml) - Real-world React Native Web app

---

## 🏆 Battle-Tested

This skill was born from **3 days of intensive QA testing** on [Rewardly](https://rewardly-cyan.vercel.app), a React Native Web app with Supabase auth and Stripe payments.

**Results**:
- 🐛 **9 critical bugs found**
- ✅ **0 caught by 1,187 unit tests**
- 🎯 **100% caught by QA Patrol**

Every bug pattern, test template, and detection rule comes from real production bugs we found and fixed.

---

## 🤝 Contributing

We welcome contributions! Here's how to help:

### Report Bugs
Found a bug pattern QA Patrol missed? [Open an issue](https://github.com/tahseen137/qa-patrol/issues) with:
- App URL (if public)
- Bug description
- Why unit tests missed it
- How to reproduce

### Add Templates
Have a test plan for Firebase auth? NextAuth? PayPal? Submit a PR!

### Improve Documentation
Typos, clarity improvements, more examples — all welcome.

---

## 📜 License

MIT — See [LICENSE](LICENSE) for details.

---

## 🔗 Links

- **ClawHub**: [clawhub.ai/tahseen137/qa-patrol](https://clawhub.ai/tahseen137/qa-patrol)
- **OpenClaw**: [openclaw.ai](https://openclaw.ai)
- **OpenClaw Discord**: [discord.com/invite/clawd](https://discord.com/invite/clawd)
- **GitHub**: [github.com/tahseen137/qa-patrol](https://github.com/tahseen137/qa-patrol)

---

## 📞 Support

- **Questions?** Ask in [OpenClaw Discord](https://discord.com/invite/clawd)
- **Bugs?** [Open an issue](https://github.com/tahseen137/qa-patrol/issues)
- **Feature requests?** [Start a discussion](https://github.com/tahseen137/qa-patrol/discussions)

---

**Built with ❤️ by the OpenClaw community**
