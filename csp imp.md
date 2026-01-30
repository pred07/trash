# CSP Implementation Guide for Legacy .NET Applications

## Table of Contents
1. [Understanding the CSP Challenge](#1-understanding-the-csp-challenge-in-legacy-net-apps)
2. [Your Tracker's Role](#2-your-trackers-role-analysis--refactoring-prediction)
3. [Inline Code Refactoring Strategy](#3-inline-code-refactoring-strategy)
4. [AJAX: Is It a Real Problem?](#4-ajax-is-it-a-real-problem)
5. [Using the Crawler for URL Whitelisting](#5-using-the-crawler-for-url-whitelisting)
6. [Correlation: Static + Dynamic Analysis](#6-correlation-static--dynamic-analysis)
7. [CSP Implementation Limitations](#7-csp-implementation-limitations)
8. [Organized Plan for CSP Implementation](#8-organized-plan-for-csp-implementation)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. Understanding the CSP Challenge in Legacy .NET Apps

### What CSP Does:
Content Security Policy (CSP) is a browser security mechanism that restricts where scripts, styles, and other resources can load from. The strictest (and most secure) CSP policy looks like:

```http
Content-Security-Policy: 
  default-src 'self'; 
  script-src 'self'; 
  style-src 'self';
```

This means:
- ❌ **Blocks ALL inline JavaScript** (`<script>alert('hi')</script>`, `onclick="..."`)
- ❌ **Blocks ALL inline CSS** (`<style>...</style>`, `style="color:red"`)
- ❌ **Blocks `eval()`, `new Function()`, `innerHTML` with scripts**
- ✅ **Allows only external files** from your own domain (`<script src="/js/app.js">`)

### Why Legacy .NET Apps Struggle:
Your ACE application has **1223 inline/internal code issues**. Each one violates CSP. Without fixing them, you have two bad choices:
1. **Enable CSP** → App breaks completely (buttons don't work, styles disappear)
2. **Use `'unsafe-inline'`** → CSP becomes nearly useless (defeats the security purpose)

---

## 2. Your Tracker's Role: Analysis & Refactoring Prediction

### What the Code_Inventory.xlsx Tells You:

| **Tab** | **What It Shows** | **Refactoring Difficulty** |
|---------|-------------------|----------------------------|
| **Inline JS (Attributes)** | `onclick="doSomething()"` | ⭐ **EASY** - Move to external `.js` with event listeners |
| **Internal JS (Blocks)** | `<script>` blocks in `.aspx` files | ⭐⭐ **MEDIUM** - Extract to `.js`, handle server dependencies |
| **External JS (Files)** | Already in `.js` files | ✅ **DONE** - Just verify CSP allows them |
| **Inline CSS (Attributes)** | `style="color:red"` | ⭐ **EASY** - Move to CSS classes |
| **Internal CSS (Blocks)** | `<style>` blocks | ⭐⭐ **MEDIUM** - Extract to `.css` |
| **AJAX Code** | Fetch/XHR calls | ⭐⭐⭐ **HARD** - May have server dependencies |

### Accuracy of Predictions:

Your tracker is **highly accurate** for:
- ✅ **Counting issues** (1223 is real)
- ✅ **Identifying inline vs. internal vs. external**
- ✅ **Detecting AJAX patterns**
- ✅ **Flagging server dependencies** (`@Model`, `@ViewBag`)

It **cannot automatically**:
- ❌ Understand business logic (you must review)
- ❌ Guarantee zero bugs after refactoring
- ❌ Handle dynamic runtime code generation

**Realistic Automation:** 50-70% is accurate. The tracker does the **discovery and extraction**, but **human review** is needed for:
- Testing extracted code
- Handling edge cases
- Verifying functionality

---

## 3. Inline Code Refactoring Strategy

### Easy Wins (Can Automate 80%):

#### A. Inline JS Attributes (`onclick`, `onload`, etc.)

**Before:**
```html
<button onclick="alert('Hello')">Click Me</button>
```

**After:**
```html
<button id="myBtn">Click Me</button>
<script src="/js/app.js"></script>
```

**app.js:**
```javascript
document.getElementById('myBtn').addEventListener('click', function() {
    alert('Hello');
});
```

**Your Tracker Helps:**
- ✅ Lists ALL inline attributes with full HTML context
- ✅ Extracts the code to `extracted_code/` folder
- ✅ You just need to wire up event listeners

---

#### B. Inline CSS Attributes

**Before:**
```html
<div style="color: red; font-size: 14px;">Text</div>
```

**After:**
```html
<div class="error-text">Text</div>
```

**styles.css:**
```css
.error-text {
    color: red;
    font-size: 14px;
}
```

**Your Tracker Helps:**
- ✅ Lists all `style="..."` attributes
- ✅ You create CSS classes and replace

---

### Medium Difficulty (Needs Review):

#### C. Internal Script Blocks with Server Code

**Before (ASPX file):**
```html
<script>
    var userId = '@Model.UserId';
    var apiUrl = '@Url.Action("GetData", "Home")';
    
    $.ajax({
        url: apiUrl,
        success: function(data) { /* ... */ }
    });
</script>
```

**Problem:** `@Model` and `@Url.Action` are **server-side Razor syntax**. They execute on the server before the page is sent to the browser.

**Solution:**
1. **Move dynamic values to `data-*` attributes:**
```html
<div id="app" data-user-id="@Model.UserId" data-api-url="@Url.Action("GetData", "Home")"></div>
<script src="/js/app.js"></script>
```

2. **Read them in external JS:**
```javascript
// app.js
var app = document.getElementById('app');
var userId = app.dataset.userId;
var apiUrl = app.dataset.apiUrl;

fetch(apiUrl)
    .then(response => response.json())
    .then(data => { /* ... */ });
```

**Your Tracker Helps:**
- ✅ Flags "Server Severity: High" for blocks with `@Model`
- ✅ Extracts the code so you can see what needs data attributes
- ⚠️ **You must manually** identify which variables are server-generated

---

## 4. AJAX: Is It a Real Problem?

### How AJAX Works:
AJAX (Asynchronous JavaScript And XML) is just **JavaScript making HTTP requests** to fetch data without reloading the page.

**Example:**
```javascript
fetch('/api/users')
    .then(response => response.json())
    .then(users => {
        // Update the page with user data
    });
```

### Does AJAX Create Dynamic CSS/JS?
**No, not directly.** But it can:
- ✅ **Fetch HTML** and inject it (`innerHTML`) → CSP blocks this if HTML contains `<script>`
- ✅ **Fetch JSON** and build DOM → CSP allows this
- ✅ **Dynamically load scripts** (`document.createElement('script')`) → CSP blocks this

### CSP Impact on AJAX:

| **AJAX Pattern** | **CSP Blocks?** | **Fix** |
|------------------|-----------------|---------|
| `fetch('/api/data')` | ❌ No | Works fine |
| `$.ajax({ url: '/api/data' })` | ❌ No | Works fine |
| `innerHTML = '<script>alert(1)</script>'` | ✅ **YES** | Use `textContent` or sanitize |
| `eval(ajaxResponse)` | ✅ **YES** | Never use `eval()` |
| `new Function(ajaxResponse)()` | ✅ **YES** | Refactor logic |

### Your Tracker's AJAX Detection:
- ✅ Finds all `fetch()`, `$.ajax()`, `XMLHttpRequest`
- ✅ Flags if they're in inline blocks (need extraction)
- ✅ Marks "Server Dependencies" (e.g., `@Url.Action`)

**Real Problem:** AJAX itself is fine. The problem is:
1. **Inline AJAX code** (violates CSP) → Extract to `.js`
2. **Dynamic script injection** (rare but dangerous) → Refactor

---

## 5. Using the Crawler for URL Whitelisting

### What the Crawler Does:
When you run `--dynamic-analysis --url http://localhost:5000`, the crawler:
1. **Visits all pages** (follows links)
2. **Records all resources** loaded:
   - Internal JS: `/js/app.js`, `/scripts/legacy.js`
   - External JS: `https://cdn.jsdelivr.net/jquery.min.js`
   - Internal CSS: `/css/styles.css`
   - External CSS: `https://fonts.googleapis.com/css`
3. **Correlates with static analysis** (matches findings from Code_Inventory)

### How This Helps CSP:

**Output: `Dynamic_Analysis_Report.xlsx`**

| **Resource** | **Type** | **Source** | **CSP Directive** |
|--------------|----------|------------|-------------------|
| `/js/app.js` | Script | Internal | `script-src 'self'` |
| `https://cdn.jsdelivr.net/jquery.min.js` | Script | External | `script-src https://cdn.jsdelivr.net` |
| `/css/main.css` | Style | Internal | `style-src 'self'` |
| `https://fonts.googleapis.com` | Style | External | `style-src https://fonts.googleapis.com` |

**You Build CSP From This:**
```http
Content-Security-Policy: 
  default-src 'self'; 
  script-src 'self' https://cdn.jsdelivr.net; 
  style-src 'self' https://fonts.googleapis.com;
```

**Accuracy:** The crawler finds **runtime-loaded resources** that static analysis misses (e.g., scripts loaded by other scripts).

---

## 6. Correlation: Static + Dynamic Analysis

### Why Correlation Matters:

| **Scenario** | **Static Analysis** | **Dynamic Analysis** | **Correlation Result** |
|--------------|---------------------|----------------------|------------------------|
| Script in code but never loaded | ✅ Found | ❌ Not found | **Dead code** (safe to ignore) |
| Script loaded at runtime only | ❌ Missed | ✅ Found | **Add to CSP whitelist** |
| Script in both | ✅ Found | ✅ Found | **Confirmed** (high priority) |

**Your `Dynamic_Analysis_Report.xlsx` has 3 tabs:**
1. **Matched:** Static findings confirmed by crawler (high confidence)
2. **New (Web-Only):** Resources loaded at runtime (add to CSP)
3. **Missing (Code-Only):** Dead code or conditional (review manually)

**This gives you:**
- ✅ **Complete CSP whitelist** (all external domains)
- ✅ **Prioritization** (focus on "Matched" items first)
- ✅ **Dead code detection** (cleanup opportunity)

---

## 7. CSP Implementation Limitations

### What's Possible:

| **Task** | **Feasibility** | **Your Tracker Helps?** |
|----------|-----------------|-------------------------|
| Extract inline `onclick` to `.js` | ✅ **EASY** | ✅ Yes (lists all + extracts) |
| Extract `<script>` blocks to `.js` | ✅ **MEDIUM** | ✅ Yes (flags server deps) |
| Move inline `style` to CSS classes | ✅ **EASY** | ✅ Yes (lists all) |
| Build CSP whitelist for external resources | ✅ **EASY** | ✅ Yes (crawler finds all) |
| Handle `eval()` / `new Function()` | ⚠️ **HARD** | ⚠️ Detects but can't auto-fix |
| Refactor server-side Razor in JS | ⚠️ **HARD** | ⚠️ Flags but needs manual review |

### What's Difficult:

1. **Dynamic Code Generation:**
   - If your app uses `eval(serverResponse)`, you must rewrite the logic
   - CSP **cannot** allow this securely

2. **Third-Party Libraries:**
   - Old jQuery plugins might use `eval()` internally
   - You may need to update or replace them

3. **Testing:**
   - After refactoring, you must test **every page** to ensure functionality
   - Automation helps with extraction, but testing is manual

---

## 8. Organized Plan for CSP Implementation

### Phase 1: Discovery (DONE ✅)
- ✅ Run `RepoScan-Analyser` (static analysis)
- ✅ Review `Code_Inventory.xlsx` (1223 issues found)
- ✅ Run crawler (dynamic analysis)
- ✅ Review `Dynamic_Analysis_Report.xlsx` (correlation)

### Phase 2: Easy Wins (Target: 40% of issues)
**Week 1-2:**
- Extract all **Inline JS Attributes** (`onclick`, etc.) to external `.js`
- Extract all **Inline CSS Attributes** (`style="..."`) to CSS classes
- Use your `extracted_code/` folder as a starting point

**Automation:**
- Your tracker already extracts the code
- Write a script to generate event listener boilerplate:
  ```javascript
  // Auto-generated from tracker
  document.getElementById('btn1').addEventListener('click', function() {
      // [Paste extracted code here]
  });
  ```

### Phase 3: Medium Difficulty (Target: 30% of issues)
**Week 3-4:**
- Extract **Internal JS Blocks** with **low server severity**
- For blocks with `@Model` / `@Url.Action`:
  - Move dynamic values to `data-*` attributes
  - Read them in external JS

**Your Tracker Helps:**
- Filter by "Server Severity: Low" or "None"
- Prioritize blocks without Razor syntax

### Phase 4: Hard Cases (Target: 20% of issues)
**Week 5-6:**
- Review **high server severity** blocks manually
- Refactor `eval()` / `innerHTML` patterns
- Update third-party libraries if needed

**Your Tracker Helps:**
- Lists all dynamic code sinks (`eval`, `innerHTML`)
- You decide case-by-case

### Phase 5: CSP Rollout
**Week 7:**
1. **Build CSP from crawler output:**
   ```http
   script-src 'self' https://cdn.jsdelivr.net https://ajax.googleapis.com;
   style-src 'self' https://fonts.googleapis.com;
   ```

2. **Test in Report-Only mode:**
   ```http
   Content-Security-Policy-Report-Only: ...
   ```
   - Logs violations without blocking
   - Fix any missed issues

3. **Enable enforcing mode:**
   ```http
   Content-Security-Policy: ...
   ```

### Phase 6: Remaining 10%
**Week 8+:**
- Handle edge cases flagged by CSP reports
- Consider `'nonce-'` or `'hash-'` for unavoidable inline code (rare)

---

## 9. Key Takeaways

### Your Doubts Answered:

| **Question** | **Answer** |
|--------------|-----------|
| Will the tracker help analysis? | ✅ **YES** - It gives you a complete inventory with severity ratings |
| Can we refactor easily? | ✅ **50-70% YES** - Inline attributes are easy, server-dependent blocks need review |
| Is AJAX a real problem? | ⚠️ **DEPENDS** - AJAX calls are fine, but inline AJAX code violates CSP |
| Does AJAX create dynamic CSS/JS? | ❌ **NO** - But it can inject HTML (which CSP blocks if it has scripts) |
| Will CSP block AJAX? | ❌ **NO** - CSP blocks inline scripts, not HTTP requests |
| How to get correlation? | ✅ **Use crawler** - Matches static findings with runtime resources |
| What's possible? | ✅ Inline → External extraction, CSP whitelist generation |
| What's difficult? | ⚠️ `eval()`, server-side Razor in JS, testing |

### Your 50-70% Automation is Realistic:
- **Automated:** Discovery, extraction, CSP whitelist generation
- **Manual:** Reviewing server dependencies, testing, edge cases

### Next Steps:
1. **Review `Code_Inventory.xlsx`** - Sort by "Server Severity: None" for easy wins
2. **Run crawler** - Get the full CSP whitelist
3. **Start with Inline JS Attributes** - Lowest risk, highest impact
4. **Test incrementally** - Don't refactor everything at once

---

## Additional Resources

### Tools in This Repository:
- **RepoScan-Analyser** - Static code analysis
- **Crawler** - Dynamic resource discovery
- **Refactoring Tracker** - Prioritization and progress tracking

### Recommended Reading:
- [MDN: Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [CSP Evaluator](https://csp-evaluator.withgoogle.com/)
- [OWASP CSP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html)

---

**Document Version:** 1.0  
**Last Updated:** 2026-01-30  
**Target Application:** ACE (Legacy .NET)  
**Total Issues Identified:** 1223
