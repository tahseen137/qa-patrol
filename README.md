# 🛡️ QA Patrol

**Automated QA testing for web apps — catches the bugs unit tests miss.**

[![ClawHub](https://img.shields.io/badge/ClawHub-qa--patrol-blue)](https://clawhub.ai/tahseen137/qa-patrol)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

## The Problem

We had **1,187 passing unit tests**. Our CEO tested the app for 10 minutes and found it _"still not fully usable."_

The bugs were in:
- **Cross-platform behavior** — `Alert.alert` callbacks don't fire on web, `Linking.openURL` fails inside modals
- **Auth state** — Supabase RLS policies blocked signed-in users (worked for anonymous!)
- **Data integrity** — Database values didn't match UI (12x inflation on rewards rates)
- **Payment integrations** — Stripe ES256 JWT rejection broke all checkouts
- **Cache masking** — Stale cache hid broken data for hours

**None of these are catchable by unit tests.** So we built QA Patrol.

## What It Does

QA Patrol is an [OpenClaw](https://openclaw.ai) skill that runs real browser-based E2E tests against your live web app. It uses declarative YAML test plans and produces structured QA reports.

### Test Levels

| Level | Template | What It Tests |
|-------|----------|---------------|
| **1** | `basic.yaml` | Homepage loads, no JS errors, navigation works |
| **2** | `auth-supabase.yaml` | Sign in/out, session persistence, RLS policies |
| **2** | `payments-stripe.yaml` | Stripe checkout, pricing accuracy, webhooks |
| **3** | `full-saas.yaml` | Everything: auth + payments + data integrity + static analysis + a11y |

### Bug Pattern Detection

QA Patrol knows about **5 categories** of bugs that unit tests structurally can't catch:

1. **Cross-Platform** — React Native Web `Alert.alert`, `Linking.openURL`, `Alert.prompt` issues
2. **Auth State** — RLS policies, session persistence, guest→user migration
3. **Data Integrity** — DB/UI mismatches, duplicate records, calculation errors
4. **Integration** — Stripe JWT auth, edge function failures, webhook reliability
5. **Cache Masking** — Stale data hiding real issues from signed-in users

## Install

### Via ClawHub (recommended)
```bash
npx clawhub@latest install qa-patrol
```

### Manual
Clone this repo into your OpenClaw workspace skills folder:
```bash
git clone https://github.com/tahseen137/qa-patrol.git ~/.openclaw/workspace/skills/qa-patrol
```

## Quick Start

Just tell your OpenClaw agent:
```
QA test https://your-app.com
```

Or use a specific template:
```
Run qa-patrol with the auth-supabase template on https://your-app.com
```

Or with a custom test plan:
```
Run qa-patrol with this plan: [paste YAML or point to file]
```

## v1.0.1 Updates (Feb 2026)

**Security & Privacy Hardening:**
- ✅ Full environment variable declarations (8 vars, all optional)
- ✅ Explicit permissions metadata (`browser`, `read`)
- ✅ Security & Privacy section added to SKILL.md
- ✅ Removed all hardcoded credentials from templates
- ✅ Added clarifying comments to bug-patterns.md (detection patterns, not exploits)

**Why these matter:** QA testing tools inherently reference secrets, database queries, and file operations — because they *test* these things. v1.0.1 makes the intent explicit and ensures no sensitive data is baked into the skill.

## Test Plan Example

```yaml
app:
  url: https://your-app.com
  name: My App
  stack: expo-web

auth:
  provider: supabase
  login_path: /auth
  accounts:
    free:
      email: test@example.com
      password: ${FREE_PASSWORD}

tests:
  smoke:
    - name: Homepage loads
      navigate: /
      assert:
        - element_exists: main
        - no_console_errors: true

  auth:
    - name: Sign in works
      steps:
        - navigate: /auth
        - type: { ref: email_input, text: "${auth.accounts.free.email}" }
        - type: { ref: password_input, text: "${auth.accounts.free.password}" }
        - click: { ref: sign_in_button }
        - wait: { url_contains: "/home", timeout: 10000 }
      assert:
        - url_contains: /home
        - element_exists: user_avatar
```

## File Structure

```
qa-patrol/
├── SKILL.md                          # Core workflow (329 lines)
├── references/
│   ├── bug-patterns.md               # 5 bug categories from real testing
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

## Built From Real Pain

This skill was born from 3 days of intensive QA testing on [Rewardly](https://rewardly-cyan.vercel.app), a React Native Web app with Supabase auth and Stripe payments. Every bug pattern, every test template, every detection rule comes from a real bug we found and fixed.

**9 critical bugs found. 0 caught by unit tests. All caught by QA Patrol.**

## Requirements

- [OpenClaw](https://openclaw.ai) with browser automation enabled
- A deployed web app to test

## License

MIT

## Links

- [ClawHub Page](https://clawhub.ai/tahseen137/qa-patrol)
- [OpenClaw](https://openclaw.ai)
- [OpenClaw Discord](https://discord.com/invite/clawd)
