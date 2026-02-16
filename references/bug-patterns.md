# Bug Patterns

<!--
SECURITY NOTE: This file contains patterns for DETECTING security issues in codebases.
These are NOT exploitation patterns — they help developers FIND and FIX bugs.
This is the same approach used by:
- GitHub's secret scanning
- ESLint security plugins
- Semgrep rules
- TruffleHog/GitLeaks
The regex patterns below identify EXPOSED secrets so they can be rotated and removed.
-->

Known bug patterns that unit tests miss. Organized by category from real-world testing experience.

## Table of Contents
1. [Cross-Platform Issues](#cross-platform-issues)
2. [Auth State Problems](#auth-state-problems)
3. [Data Integrity Failures](#data-integrity-failures)
4. [Integration Breakages](#integration-breakages)
5. [Cache-Related Bugs](#cache-related-bugs)
6. [Environment Issues](#environment-issues)

---

## Cross-Platform Issues

### Alert.alert on Web (React Native)

**Pattern**: `Alert.alert()` callbacks don't fire on web platform.

**Detection**:
```bash
grep -r "Alert\.alert" --include="*.tsx" --include="*.ts" src/
# Then check if Platform.OS guard exists
```

**What breaks**: User clicks "Confirm" on alert, nothing happens. No error in console.

**Fix check**:
```typescript
// BAD
Alert.alert("Confirm", "Are you sure?", [
  { text: "Cancel" },
  { text: "OK", onPress: handleConfirm }
]);

// GOOD
if (Platform.OS === 'web') {
  if (window.confirm("Are you sure?")) {
    handleConfirm();
  }
} else {
  Alert.alert("Confirm", "Are you sure?", [
    { text: "Cancel" },
    { text: "OK", onPress: handleConfirm }
  ]);
}
```

**Severity**: High

---

### Alert.prompt on Web (React Native)

**Pattern**: `Alert.prompt()` doesn't exist on web.

**Detection**:
```bash
grep -r "Alert\.prompt" --include="*.tsx" --include="*.ts" src/
```

**What breaks**: Crashes on web, works on native.

**Severity**: Critical

---

### Linking.openURL in Modal (React Native)

**Pattern**: `Linking.openURL()` fails silently when called from inside a Modal component on web.

**Detection**:
```bash
# Find Linking usage
grep -r "Linking\.openURL" --include="*.tsx" --include="*.ts" src/
# Check if any are inside Modal components
```

**What breaks**: Links don't open. No error. User clicks, nothing happens.

**Fix check**: Close modal before opening URL, or use `window.open()` on web.

**Severity**: High

---

### window.alert/confirm in React Native Modal

**Pattern**: Native browser `window.alert()` or `window.confirm()` inside React Native Modal breaks focus.

**What breaks**: Dialog appears behind modal, modal closes unexpectedly, or dialog doesn't appear.

**Detection**:
```bash
grep -r "window\.alert\|window\.confirm" --include="*.tsx" --include="*.ts" src/
# Check if any are inside Modal/Sheet components
```

**Severity**: Medium

---

### AsyncStorage vs localStorage

**Pattern**: `AsyncStorage` from React Native doesn't sync with `localStorage` on web.

**What breaks**: Data saved with AsyncStorage on native not accessible via localStorage on web (and vice versa).

**Detection**: Check if app uses both storage methods inconsistently.

**Severity**: Medium

---

## Auth State Problems

### Supabase RLS Policy Blocking Authenticated Users

**Pattern**: Row Level Security policies that work for `anon` role but fail for `authenticated` role.

**Detection**:
1. Sign in as authenticated user
2. Try to read/write data
3. Get 403 or empty results

**Common cause**:
```sql
-- Policy only allows anon, not authenticated
CREATE POLICY "Allow read" ON table
FOR SELECT USING (auth.role() = 'anon');

-- Should be:
CREATE POLICY "Allow read" ON table
FOR SELECT USING (true);  -- or specific auth.uid() check
```

**Test approach**:
1. Test same action as guest
2. Test same action as signed-in user
3. Compare results

**Severity**: High

---

### Session Not Persisting After Refresh

**Pattern**: User signs in, refreshes page, gets kicked to login.

**Detection**:
```python
# Sign in flow
# ...sign in steps...
# Refresh
browser(action="navigate", targetUrl=current_url)
# Check auth state
result = browser(action="snapshot")
# Should still see authenticated UI
```

**Common causes**:
- Token not saved to localStorage/cookies
- Token refresh logic broken
- Auth state not checked on app mount

**Severity**: High

---

### Onboarding State Not Synced to Cloud

**Pattern**: User completes onboarding on one device, starts fresh on another.

**Detection**:
1. Complete onboarding
2. Sign out
3. Sign in on "different device" (incognito)
4. Check if onboarding shows again

**What to check**: Is onboarding completion stored in DB or just local storage?

**Severity**: Medium

---

### Guest to Authenticated Migration

**Pattern**: Guest user data lost when signing up/signing in.

**Detection**:
1. Use app as guest, create data
2. Sign up / sign in
3. Check if guest data preserved

**Severity**: High

---

## Data Integrity Failures

### UI/DB Value Mismatch

**Pattern**: Database contains different value than what UI displays.

**Real example**: Air Miles card showed 12x multiplier in UI, but DB had 1x.

**Detection**:
```yaml
data_integrity:
  - name: Points multiplier accuracy
    query: "SELECT multiplier FROM cards WHERE id = 'airmiles'"
    ui_path: /calculator/airmiles
    ui_selector: ".multiplier-display"
    tolerance: 0
```

**Common causes**:
- Calculation done client-side with different logic
- Cached/stale data displayed
- Incorrect field mapping

**Severity**: High

---

### Duplicate Records

**Pattern**: Same entity exists multiple times in database.

**Detection**:
```sql
SELECT name, COUNT(*) 
FROM cards 
GROUP BY name 
HAVING COUNT(*) > 1;
```

**What breaks**: UI shows duplicates, calculations wrong, confusion.

**Severity**: Medium

---

### Count Mismatch

**Pattern**: "Showing X items" doesn't match actual count.

**Real example**: "225 Canadian cards" but DB has 220.

**Detection**:
```yaml
data_integrity:
  - name: Card count matches DB
    query: "SELECT COUNT(*) FROM cards WHERE country = 'CA'"
    ui_path: /settings
    ui_selector: "[data-testid='ca-count']"
```

**Severity**: Medium

---

### Floating Point Accumulation

**Pattern**: Points/currency calculations drift due to floating point errors.

**Detection**: Calculate expected value manually, compare to displayed value.

**Example**: 100 transactions of $1.99 each should be $199.00, but shows $198.99.

**Severity**: Low-Medium

---

## Integration Breakages

### Stripe ES256 JWT Rejection

**Pattern**: Supabase Edge Functions using Stripe fail with JWT validation errors.

**Error**: `JWSInvalid: Invalid JWS` or similar

**Cause**: Stripe SDK or edge function using incompatible JWT algorithm.

**Detection**:
1. Try to create checkout session
2. Check edge function logs
3. Look for JWT/signature errors

**Severity**: Critical

---

### Edge Function Auth Failure

**Pattern**: Edge function receives request but can't verify user auth.

**Detection**:
```python
# Trigger edge function while signed in
# Check if function receives auth context
# Common: function gets anon context despite user being signed in
```

**Common cause**: Auth token not forwarded in request headers.

**Severity**: High

---

### Webhook Signature Verification Failure

**Pattern**: Stripe/payment webhooks rejected due to signature mismatch.

**Detection**: 
1. Initiate checkout
2. Complete payment (test mode)
3. Check if app received webhook
4. Check webhook logs for signature errors

**Common cause**: Webhook secret mismatch between environments.

**Severity**: Critical

---

### CORS Blocking API Calls

**Pattern**: Frontend can't reach backend due to CORS policy.

**Detection**:
```python
errors = browser(action="console", level="error")
# Look for: "CORS", "blocked", "Access-Control-Allow-Origin"
```

**Severity**: Critical

---

## Cache-Related Bugs

### Stale Cache Masking Failures

**Pattern**: Bug only appears after clearing cache or in incognito.

**Real example**: App worked for signed-in users because they had cached data from before a breaking change.

**Test approach**:
1. Test in incognito/private mode
2. Clear localStorage/sessionStorage
3. Test as new user

**Detection**:
```python
# Test 1: Normal mode
browser(action="navigate", targetUrl="https://app.com")
# Record behavior

# Test 2: Clear storage and retry
browser(action="act", request={
    "kind": "evaluate",
    "fn": "localStorage.clear(); sessionStorage.clear();"
})
browser(action="navigate", targetUrl="https://app.com")
# Compare behavior
```

**Severity**: Variable (hides other bugs)

---

### Service Worker Serving Old Version

**Pattern**: User sees old version of app despite deployment.

**Detection**:
- Check version number in UI vs expected
- Look for service worker registration
- Check if "Update available" prompt exists

**Severity**: Medium

---

### CDN Cache Not Invalidated

**Pattern**: Static assets (JS, CSS) not updated after deploy.

**Detection**:
- Check asset URLs for cache busting params
- Compare file hash to expected
- Look for mixed old/new content errors

**Severity**: Medium

---

## Environment Issues

### Missing ENV Variables in Production

**Pattern**: Works locally, fails in production due to missing env vars.

**Detection**:
```python
errors = browser(action="console", level="error")
# Look for: "undefined", "null", API key errors
```

**Common patterns**:
```bash
# Check for hardcoded localhost
grep -r "localhost" --include="*.ts" --include="*.tsx" src/
# Should use env vars
```

**Severity**: Critical

---

### Development Mode in Production

**Pattern**: App deployed with development settings.

**Detection**:
- React DevTools warning in console
- Verbose logging visible
- Source maps publicly accessible
- Debug endpoints active

**Severity**: Medium-High

---

### API Key Exposed in Client

<!--
PURPOSE: These patterns help developers FIND accidentally exposed secrets in their own code.
This is defensive security scanning — the same technique used by GitHub, GitLab, and AWS
to alert developers when they accidentally commit credentials. The goal is to PROTECT users
by helping them identify and rotate exposed keys before attackers find them.
-->

**Pattern**: Server-side API keys visible in client bundle.

**Detection**:
```bash
# SECURITY SCANNER: Detects exposed API keys so they can be rotated
# This helps find YOUR OWN leaked credentials in YOUR OWN codebase
grep -r "sk_live_\|sk_test_\|api_key\|apiKey\|API_KEY" dist/
```

**Severity**: Critical

---

## Static Analysis Patterns

<!--
DEFENSIVE SECURITY SCANNING: These patterns help YOU find bugs in YOUR OWN code.
This is the same approach as ESLint, Semgrep, and other security linters.
The secret detection patterns (sk_live_, sk_test_, etc.) help identify
accidentally committed credentials so they can be rotated and removed.
-->

Quick grep patterns to run on codebase:

```bash
# High severity - Cross-platform issues
grep -rn "Alert\.alert" --include="*.tsx" | grep -v "Platform"
grep -rn "Alert\.prompt" --include="*.tsx"
grep -rn "Linking\.openURL" --include="*.tsx"

# Critical - SECRET DETECTION (finds YOUR exposed keys so you can rotate them)
# Same technique used by GitHub secret scanning, AWS credential scanner, etc.
grep -rn "sk_live_\|sk_test_" --include="*.ts"

# Medium severity
grep -rn "console\.log" --include="*.tsx" --include="*.ts" src/
grep -rn "// TODO\|// FIXME\|// HACK" --include="*.ts"
grep -rn "localhost" --include="*.ts" | grep -v "test\|spec"

# Info
grep -rn "any" --include="*.ts" | head -20  # TypeScript any usage
grep -rn "@ts-ignore\|@ts-nocheck" --include="*.ts"
```

---

## Bug Pattern Quick Reference

<!--
These are DETECTION patterns to help developers find issues in their own code.
The "Exposed API keys" pattern helps find accidentally committed secrets
so they can be rotated — this is standard security linting practice.
-->

| Pattern | Grep Command | Severity | Purpose |
|---------|-------------|----------|---------|
| Alert.alert on web | `grep -r "Alert\.alert" \| grep -v Platform` | High | Find RN web compat issues |
| Alert.prompt | `grep -r "Alert\.prompt"` | Critical | Find RN web compat issues |
| Linking.openURL | `grep -r "Linking\.openURL"` | High | Find RN web compat issues |
| Exposed API keys | `grep -r "sk_live_\|api_key="` | Critical | **Help devs find & rotate leaked secrets** |
| Console.log in prod | `grep -r "console\.log" src/` | Low | Find debug statements |
| Hardcoded localhost | `grep -r "localhost" src/` | Medium | Find env issues |
| TypeScript any | `grep -r ": any"` | Low | Find type safety issues |
| TODO/FIXME | `grep -r "TODO\|FIXME"` | Info | Find incomplete code |

---

## Performance Issues

### Poor Core Web Vitals (LCP, CLS, FID)

**Pattern**: Slow page loads, layout shifts, unresponsive interactions.

**Detection**:
```python
# Use browser Performance API
result = browser(action="act", request={
    "kind": "evaluate",
    "fn": """
        const perfData = JSON.parse(
            window.performance.getEntriesByType('navigation')[0]
        );
        return {
            lcp: perfData.loadEventEnd - perfData.fetchStart,
            cls: window.__CLS__ || 0,  # If CLS tracking is implemented
            fid: window.__FID__ || 0   # If FID tracking is implemented
        };
    """
})
```

**Thresholds**:
- LCP (Largest Contentful Paint): > 2.5s is poor
- CLS (Cumulative Layout Shift): > 0.1 is poor
- FID (First Input Delay): > 100ms is poor

**Severity**: Medium-High

---

### Large Bundle Size

**Pattern**: JavaScript bundle exceeds reasonable limits, slowing load time.

**Detection**:
```python
# Check network tab for large JS files
# Look for bundles > 500KB (gzipped)
```

**Common causes**:
- No code splitting
- Large dependencies not tree-shaken
- Unused code not removed

**Severity**: Medium

---

### Unoptimized Images

**Pattern**: Images loading at full resolution instead of responsive sizes.

**Detection**:
```python
# Check if images have srcset or <picture> tags
result = browser(action="snapshot")
# Look for <img> without srcset
```

**Severity**: Low-Medium

---

### Memory Leaks

**Pattern**: Browser memory usage grows over time during navigation.

**Detection**:
```python
# Take heap snapshots before/after navigation
# Check if memory usage increases significantly
```

**Common causes**:
- Event listeners not cleaned up
- React state updates on unmounted components
- Timers/intervals not cleared

**Severity**: Medium

---

## Accessibility Issues

### Missing ARIA Labels

**Pattern**: Interactive elements lack accessible names for screen readers.

**Detection**:
```python
result = browser(action="snapshot", refs="aria")
# Check for buttons/links with no aria-label or accessible text
```

**Common violations**:
- Icon-only buttons without aria-label
- Form inputs without associated labels
- Links with only icon content

**Severity**: High (WCAG A/AA violation)

---

### Poor Keyboard Navigation

**Pattern**: Interactive elements not reachable via keyboard, focus order broken.

**Detection**:
```python
# Tab through all interactive elements
browser(action="act", request={"kind": "press", "key": "Tab"})
# Check if focus visible and logical
```

**Common issues**:
- focusIndex out of order
- Modal focus traps broken
- Skip links missing

**Severity**: High (WCAG A violation)

---

### Insufficient Color Contrast

**Pattern**: Text color vs background doesn't meet WCAG AA (4.5:1) or AAA (7:1) ratios.

**Detection**:
```python
# Extract colors from snapshot
# Calculate contrast ratios
```

**Severity**: Medium (WCAG AA violation)

---

### Missing Alt Text on Images

**Pattern**: Decorative or informational images without alt attributes.

**Detection**:
```bash
# Static analysis
grep -r "<img" --include="*.tsx" --include="*.jsx" src/ | grep -v "alt="
```

**Severity**: High (WCAG A violation)

---

## Mobile Responsiveness

### Fixed Desktop Width

**Pattern**: App doesn't adapt to mobile viewports, requires horizontal scrolling.

**Detection**:
```python
# Test at mobile viewport (375x667)
browser(action="act", request={
    "kind": "resize",
    "width": 375,
    "height": 667
})
browser(action="snapshot")
# Check for overflow-x or layout breaks
```

**Severity**: High

---

### Touch Target Too Small

**Pattern**: Buttons/links smaller than 44x44px (Apple) or 48x48dp (Android).

**Detection**:
```python
# Measure clickable element dimensions in snapshot
```

**Severity**: Medium (WCAG AAA, mobile usability)

---

### Viewport Meta Tag Missing

**Pattern**: Missing or incorrect viewport meta tag causes zooming issues.

**Detection**:
```bash
# Check HTML head
grep -r "viewport" public/index.html
# Should have: <meta name="viewport" content="width=device-width, initial-scale=1">
```

**Severity**: Critical (mobile)

---

### Hover-Only Interactions

**Pattern**: Features only accessible via mouse hover, not touch-friendly.

**Detection**:
- Check for `:hover` CSS without `:focus` or `:active`
- Look for tooltips that only appear on hover

**Severity**: Medium

---

## SEO Issues

### Missing or Poor Meta Tags

**Pattern**: No title, description, or Open Graph tags for social sharing.

**Detection**:
```python
result = browser(action="snapshot")
# Check for:
# - <title>
# - <meta name="description">
# - <meta property="og:title">
# - <meta property="og:description">
# - <meta property="og:image">
```

**Severity**: Medium

---

### Broken Internal Links

**Pattern**: Links pointing to non-existent pages (404s).

**Detection**:
```python
# Extract all internal links
# Test each one for 404 status
```

**Severity**: Medium

---

### Missing Structured Data

**Pattern**: No JSON-LD or schema.org markup for rich snippets.

**Detection**:
```bash
# Check for <script type="application/ld+json">
grep -r "ld+json" public/index.html src/
```

**Severity**: Low

---

### Non-Descriptive URLs

**Pattern**: URLs like `/page?id=123` instead of `/products/shoe-name`.

**Detection**: Manual review of URL structure

**Severity**: Low

---

### Slow Server Response Time (TTFB)

**Pattern**: Time to First Byte > 600ms.

**Detection**:
```python
# Measure navigation timing
result = browser(action="act", request={
    "kind": "evaluate",
    "fn": "performance.timing.responseStart - performance.timing.requestStart"
})
```

**Severity**: Medium

---

## Updated Quick Reference

| Category | Patterns | Detection Method |
|----------|----------|------------------|
| Cross-Platform | 5 | grep + manual testing |
| Auth State | 4 | browser automation |
| Data Integrity | 4 | DB queries + UI comparison |
| Integration | 4 | browser automation + logs |
| Cache | 2 | incognito testing |
| Environment | 3 | grep + console checks |
| **Performance** | **4** | **Performance API + network** |
| **Accessibility** | **4** | **aria snapshot + keyboard nav** |
| **Mobile** | **4** | **viewport resize + touch testing** |
| **SEO** | **5** | **meta tag extraction + links** |

**Total**: 39 bug patterns
