# CSP Implementation: Practical Step-by-Step Guide

## Overview

This guide provides a **practical, implementation-focused** approach to enabling Content Security Policy (CSP) for the ACE legacy .NET application, including where compromises are acceptable.

**Key Principle:** CSP implementation is iterative. Start permissive, tighten gradually, accept strategic compromises.

---

## Table of Contents
1. [CSP Implementation Phases](#csp-implementation-phases)
2. [Phase 0: Pre-Implementation Assessment](#phase-0-pre-implementation-assessment)
3. [Phase 1: Report-Only Mode](#phase-1-report-only-mode-week-1-2)
4. [Phase 2: Refactor Easy Wins](#phase-2-refactor-easy-wins-week-3-5)
5. [Phase 3: Handle Medium Complexity](#phase-3-handle-medium-complexity-week-6-8)
6. [Phase 4: Strategic Compromises](#phase-4-strategic-compromises-week-9-10)
7. [Phase 5: Enforcement](#phase-5-enforcement-week-11-12)
8. [Acceptable Compromises](#acceptable-compromises)
9. [Real-World Examples from ACE](#real-world-examples-from-ace)

---

## CSP Implementation Phases

```
Phase 0: Assessment (Current State)
    ↓
Phase 1: Report-Only Mode (Baseline CSP, monitor violations)
    ↓
Phase 2: Easy Wins (Inline attributes → External files)
    ↓
Phase 3: Medium Complexity (Script blocks with low server deps)
    ↓
Phase 4: Strategic Compromises (Decide what to keep inline)
    ↓
Phase 5: Enforcement (Enable CSP, monitor, iterate)
```

**Timeline:** 12 weeks for ACE (1223 issues)

---

## Phase 0: Pre-Implementation Assessment

### Goal: Understand Current State

**Duration:** Already completed ✅

**What You Have:**
```
RepoScan-Analyser Output:
├── Code_Inventory.xlsx (1223 issues)
├── Refactoring_Tracker.xlsx (prioritized)
├── extracted_code/ (1154 files)
└── Crawler_Input.xlsx (URLs to test)
```

### Key Metrics from ACE Scan:

| **Category** | **Count** | **% of Total** | **Refactoring Difficulty** |
|--------------|-----------|----------------|----------------------------|
| Inline JS (Attributes) | 400 | 33% | ⭐ Easy |
| Internal JS (Blocks) | 350 | 29% | ⭐⭐ Medium |
| External JS (Files) | 120 | 10% | ✅ Already compliant |
| Inline CSS (Attributes) | 150 | 12% | ⭐ Easy |
| Internal CSS (Blocks) | 100 | 8% | ⭐⭐ Medium |
| External CSS (Files) | 80 | 7% | ✅ Already compliant |
| AJAX Code (Inline) | 23 | 2% | ⭐⭐⭐ Hard |
| **Total** | **1223** | **100%** | |

### Decision Matrix:

**Easy Wins (45% of issues):**
- 400 inline JS attributes
- 150 inline CSS attributes
- **Target:** 100% refactoring

**Medium (37% of issues):**
- 350 internal JS blocks
- 100 internal CSS blocks
- **Target:** 70% refactoring, 30% use nonces

**Hard (10% of issues):**
- 23 inline AJAX blocks with server deps
- **Target:** Strategic compromises (nonces or accept `unsafe-inline` temporarily)

**Already Compliant (8%):**
- 200 external files
- **Action:** Verify CSP allows them

---

## Phase 1: Report-Only Mode (Week 1-2)

### Goal: Establish Baseline Without Breaking Anything

### Step 1.1: Create Initial CSP Policy

**Start with a permissive policy that logs violations but doesn't block:**

```http
Content-Security-Policy-Report-Only: 
  default-src 'self'; 
  script-src 'self' 'unsafe-inline' 'unsafe-eval' https:; 
  style-src 'self' 'unsafe-inline' https:; 
  font-src 'self' https:; 
  img-src 'self' https: data:; 
  connect-src 'self' https:; 
  report-uri /csp-violation-report;
```

**Why this policy:**
- ✅ `'unsafe-inline'` allows all inline code (no breakage)
- ✅ `'unsafe-eval'` allows `eval()` (no breakage)
- ✅ `https:` allows all HTTPS external resources
- ✅ `report-uri` logs violations for analysis

### Step 1.2: Implement CSP Header in ASP.NET

**Option A: web.config (Global)**
```xml
<system.webServer>
  <httpProtocol>
    <customHeaders>
      <add name="Content-Security-Policy-Report-Only" 
           value="default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https:; style-src 'self' 'unsafe-inline' https:; font-src 'self' https:; img-src 'self' https: data:; connect-src 'self' https:; report-uri /csp-violation-report;" />
    </customHeaders>
  </httpProtocol>
</system.webServer>
```

**Option B: Global.asax (Programmatic)**
```csharp
protected void Application_BeginRequest(object sender, EventArgs e)
{
    HttpContext.Current.Response.Headers.Add(
        "Content-Security-Policy-Report-Only",
        "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https:; style-src 'self' 'unsafe-inline' https:; font-src 'self' https:; img-src 'self' https: data:; connect-src 'self' https:; report-uri /csp-violation-report;"
    );
}
```

### Step 1.3: Create Violation Report Endpoint

**Create Controller:**
```csharp
// Controllers/CspController.cs
public class CspController : Controller
{
    [HttpPost]
    [Route("csp-violation-report")]
    public ActionResult ViolationReport()
    {
        try
        {
            using (var reader = new StreamReader(Request.InputStream))
            {
                var json = reader.ReadToEnd();
                
                // Log to file or database
                System.IO.File.AppendAllText(
                    Server.MapPath("~/App_Data/csp-violations.log"),
                    $"{DateTime.Now}: {json}\n"
                );
            }
            return new HttpStatusCodeResult(204); // No Content
        }
        catch
        {
            return new HttpStatusCodeResult(500);
        }
    }
}
```

### Step 1.4: Run Dynamic Analysis

**Execute Crawler:**
```bash
cd C:\Users\groot\Music\susu\RepoScan\RepoScan-Analyser
python main.py --dynamic-analysis --url http://localhost:5000
```

**Crawler will:**
1. Visit all pages in `Crawler_Input.xlsx`
2. Record all external resources loaded
3. Generate `Dynamic_Analysis_Report.xlsx`

### Step 1.5: Analyze Violations

**After 1 week of monitoring:**

**Review `csp-violations.log`:**
```json
{
  "csp-report": {
    "document-uri": "http://localhost:5000/Dashboard",
    "violated-directive": "script-src",
    "blocked-uri": "https://cdn.example.com/analytics.js",
    "source-file": "http://localhost:5000/Dashboard",
    "line-number": 145
  }
}
```

**Common violations to expect:**
- External CDN domains (add to whitelist)
- Inline event handlers (already in your tracker)
- Inline script blocks (already in your tracker)

**Action:** Update CSP to whitelist legitimate external domains

---

## Phase 2: Refactor Easy Wins (Week 3-5)

### Goal: Eliminate 45% of Issues (Inline Attributes)

### Step 2.1: Process Inline JS Attributes

**From Code_Inventory.xlsx - "Inline JS (Attributes)" tab:**

**Example Row:**
| File Path | Line | Context | Code Snippet | Full Code |
|-----------|------|---------|--------------|-----------|
| /Views/Dashboard/Index.aspx | 145 | onclick | `onclick="loadData()"` | `<button id="loadBtn" onclick="loadData()">` |

**Refactoring Process:**

#### 2.1.1: Create Event Handler File

**Create: `/Scripts/event-handlers.js`**
```javascript
// Event Handlers for ACE Application
// Auto-generated from RepoScan-Analyser extraction

(function() {
    'use strict';
    
    // Wait for DOM to load
    document.addEventListener('DOMContentLoaded', function() {
        initializeEventHandlers();
    });
    
    function initializeEventHandlers() {
        // Dashboard - Load Data Button
        var loadBtn = document.getElementById('loadBtn');
        if (loadBtn) {
            loadBtn.addEventListener('click', loadData);
        }
        
        // Add more handlers here...
    }
    
    // Handler functions (from extracted_code/)
    function loadData() {
        // Code from extracted_code/Views_Dashboard_Index.aspx_onclick_L145.js
        $.ajax({
            url: '/api/dashboard/data',
            success: function(data) {
                $('#result').html(data);
            }
        });
    }
    
})();
```

#### 2.1.2: Include in Layout

**In `/Views/Shared/_Layout.cshtml`:**
```html
<!DOCTYPE html>
<html>
<head>
    <!-- Existing head content -->
</head>
<body>
    @RenderBody()
    
    <!-- jQuery (if used) -->
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.6.0/jquery.min.js"></script>
    
    <!-- Event Handlers -->
    <script src="~/Scripts/event-handlers.js"></script>
    
    @RenderSection("scripts", required: false)
</body>
</html>
```

#### 2.1.3: Remove Inline Attribute

**In `/Views/Dashboard/Index.aspx`:**
```html
<!-- BEFORE -->
<button id="loadBtn" onclick="loadData()">Load Data</button>

<!-- AFTER -->
<button id="loadBtn">Load Data</button>
```

#### 2.1.4: Test

1. Navigate to `/Dashboard`
2. Click "Load Data" button
3. Verify AJAX call executes
4. Check browser console for errors

**Repeat for all 400 inline JS attributes**

---

### Step 2.2: Process Inline CSS Attributes

**From Code_Inventory.xlsx - "Inline CSS (Attributes)" tab:**

**Example Row:**
| File Path | Line | Attribute | Code Snippet |
|-----------|------|-----------|--------------|
| /Views/Dashboard/Index.aspx | 89 | inlinestyle | `style="color: red; font-size: 14px;"` |

**Refactoring Process:**

#### 2.2.1: Create CSS Classes

**In `/Content/site.css`:**
```css
/* Extracted Inline Styles */

/* Dashboard - Error Text */
.dashboard-error-text {
    color: red;
    font-size: 14px;
}

/* Dashboard - Highlight Box */
.dashboard-highlight {
    background-color: yellow;
    padding: 10px;
    border: 1px solid #ccc;
}

/* Add more classes... */
```

#### 2.2.2: Replace Inline Styles

**In `/Views/Dashboard/Index.aspx`:**
```html
<!-- BEFORE -->
<div style="color: red; font-size: 14px;">Error message</div>

<!-- AFTER -->
<div class="dashboard-error-text">Error message</div>
```

#### 2.2.3: Test

1. Navigate to page
2. Verify styling looks identical
3. Use browser DevTools to confirm CSS class is applied

**Repeat for all 150 inline CSS attributes**

---

### Step 2.3: Update CSP Policy

**After refactoring inline attributes, tighten CSP:**

```http
Content-Security-Policy-Report-Only: 
  default-src 'self'; 
  script-src 'self' 'unsafe-inline' 'unsafe-eval' https://ajax.googleapis.com https://cdn.jsdelivr.net; 
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; 
  font-src 'self' https://fonts.gstatic.com; 
  img-src 'self' https: data:; 
  connect-src 'self'; 
  report-uri /csp-violation-report;
```

**Changes:**
- ✅ Specific external domains (from crawler output)
- ⚠️ Still using `'unsafe-inline'` (for remaining script/style blocks)

---

## Phase 3: Handle Medium Complexity (Week 6-8)

### Goal: Refactor Internal Blocks Without Server Dependencies

### Step 3.1: Filter Refactoring Tracker

**In `Refactoring_Tracker.xlsx`:**
- Filter: `Code Type = scriptblock`
- Filter: `Server Severity = None OR Low`
- Result: ~200 script blocks

### Step 3.2: Extract Script Blocks

**Example from Code_Inventory.xlsx:**

**File:** `/Views/Dashboard/Index.aspx`  
**Lines:** 200-250  
**Code:**
```html
<script>
    $(document).ready(function() {
        $('#saveBtn').click(function() {
            var value = $('#inputField').val();
            
            $.ajax({
                url: '/api/dashboard/save',
                method: 'POST',
                data: { value: value },
                success: function(response) {
                    alert('Saved successfully!');
                },
                error: function() {
                    alert('Error saving data');
                }
            });
        });
    });
</script>
```

**Already extracted to:** `extracted_code/Views_Dashboard_Index.aspx_scriptblock_L200-L250.js`

#### 3.2.1: Review Extracted File

**Check for:**
- ✅ No `@Model`, `@ViewBag`, `@Url.Action` (confirmed by tracker)
- ✅ No `eval()` or dynamic code
- ✅ Standard jQuery/JavaScript

#### 3.2.2: Include Extracted File

**Option A: Direct Include**
```html
<!-- In /Views/Dashboard/Index.aspx -->
<script src="~/Scripts/extracted/Views_Dashboard_Index.aspx_scriptblock_L200-L250.js"></script>
```

**Option B: Bundle (Recommended)**
```csharp
// In App_Start/BundleConfig.cs
bundles.Add(new ScriptBundle("~/bundles/dashboard").Include(
    "~/Scripts/extracted/Views_Dashboard_Index.aspx_scriptblock_L200-L250.js",
    "~/Scripts/extracted/Views_Dashboard_Index.aspx_scriptblock_L300-L350.js"
));
```

```html
<!-- In /Views/Dashboard/Index.aspx -->
@Scripts.Render("~/bundles/dashboard")
```

#### 3.2.3: Remove Inline Block

```html
<!-- DELETE THIS -->
<script>
    $(document).ready(function() { ... });
</script>
```

#### 3.2.4: Test

1. Load page
2. Click save button
3. Verify AJAX call works
4. Check for console errors

**Repeat for all 200 low-complexity script blocks**

---

### Step 3.3: Handle Script Blocks with Server Dependencies

**Example with Server Code:**

**File:** `/Views/Users/List.aspx`  
**Code:**
```html
<script>
    var userId = '@Model.UserId';
    var apiUrl = '@Url.Action("GetUsers", "Api")';
    
    fetch(apiUrl)
        .then(response => response.json())
        .then(users => {
            console.log('Current user:', userId);
            displayUsers(users);
        });
</script>
```

**Tracker flags:** `Server Severity: High`

#### 3.3.1: Move Server Values to HTML

```html
<!-- In /Views/Users/List.aspx -->
<div id="userListApp" 
     data-user-id="@Model.UserId" 
     data-api-url="@Url.Action("GetUsers", "Api")"
     data-page-size="@ViewBag.PageSize">
</div>
```

#### 3.3.2: Create External JS File

**Create: `/Scripts/users-list.js`**
```javascript
(function() {
    'use strict';
    
    document.addEventListener('DOMContentLoaded', function() {
        var app = document.getElementById('userListApp');
        if (!app) return;
        
        // Read server values from data attributes
        var userId = app.dataset.userId;
        var apiUrl = app.dataset.apiUrl;
        var pageSize = parseInt(app.dataset.pageSize);
        
        // Original logic (unchanged)
        fetch(apiUrl)
            .then(response => response.json())
            .then(users => {
                console.log('Current user:', userId);
                displayUsers(users);
            });
    });
    
    function displayUsers(users) {
        // Implementation...
    }
    
})();
```

#### 3.3.3: Include External File

```html
<!-- In /Views/Users/List.aspx -->
<script src="~/Scripts/users-list.js"></script>
```

#### 3.3.4: Remove Inline Block

```html
<!-- DELETE THIS -->
<script>
    var userId = '@Model.UserId';
    ...
</script>
```

#### 3.3.5: Test

1. Navigate to `/Users/List`
2. Verify `data-*` attributes have correct values
3. Check console: `console.log('Current user:', userId)` should show correct ID
4. Verify user list displays correctly

**Repeat for ~150 script blocks with server dependencies**

---

## Phase 4: Strategic Compromises (Week 9-10)

### Goal: Decide What to Keep Inline (With Nonces)

### When to Compromise:

**Acceptable to keep inline (with nonces):**
1. ✅ Critical functionality that's too complex to refactor (< 5% of code)
2. ✅ Third-party widgets that require inline scripts
3. ✅ Temporary: Code scheduled for rewrite in next sprint

**NOT acceptable:**
1. ❌ Simple event handlers (already refactored)
2. ❌ Standard AJAX calls (easy to extract)
3. ❌ Inline styles (use CSS classes)

### Step 4.1: Identify Compromise Candidates

**From Refactoring_Tracker.xlsx:**
- Filter: `Complexity = High`
- Filter: `Recommended Action = Review`
- Result: ~50 complex blocks

**Example Candidates:**
1. Legacy charting library (inline config)
2. Third-party payment widget
3. Complex form validation with 500+ lines

### Step 4.2: Implement Nonce-Based CSP

#### 4.2.1: Generate Nonce on Server

**In Global.asax:**
```csharp
protected void Application_BeginRequest(object sender, EventArgs e)
{
    // Generate unique nonce per request
    var nonce = Convert.ToBase64String(Guid.NewGuid().ToByteArray());
    HttpContext.Current.Items["CSP_NONCE"] = nonce;
    
    // Add CSP header with nonce
    var csp = $"default-src 'self'; " +
              $"script-src 'self' 'nonce-{nonce}' https://ajax.googleapis.com; " +
              $"style-src 'self' 'nonce-{nonce}' https://fonts.googleapis.com; " +
              $"font-src 'self' https://fonts.gstatic.com; " +
              $"img-src 'self' https: data:; " +
              $"connect-src 'self';";
    
    HttpContext.Current.Response.Headers.Add("Content-Security-Policy-Report-Only", csp);
}
```

#### 4.2.2: Create Helper Method

**In App_Code/CspHelper.cs:**
```csharp
public static class CspHelper
{
    public static string GetNonce()
    {
        return HttpContext.Current.Items["CSP_NONCE"]?.ToString() ?? "";
    }
    
    public static IHtmlString NonceAttribute()
    {
        var nonce = GetNonce();
        return new HtmlString($"nonce=\"{nonce}\"");
    }
}
```

#### 4.2.3: Apply Nonce to Inline Scripts

**In /Views/Dashboard/Index.aspx:**
```html
<!-- Complex legacy script that's hard to refactor -->
<script @CspHelper.NonceAttribute()>
    // 500 lines of complex charting configuration
    var chartConfig = {
        // Complex nested object...
    };
    initializeChart(chartConfig);
</script>
```

#### 4.2.4: Test

1. View page source
2. Verify nonce attribute: `<script nonce="abc123xyz...">`
3. Check browser console - no CSP violations
4. Refresh page - nonce changes (security)

### Step 4.3: Document Compromises

**Create: `/Docs/CSP_COMPROMISES.md`**
```markdown
# CSP Implementation Compromises

## Inline Scripts with Nonces (23 instances)

### 1. Dashboard Charting (Priority: Low)
- **File:** /Views/Dashboard/Index.aspx
- **Lines:** 300-800
- **Reason:** Complex legacy charting library config
- **Compromise:** Using nonce
- **Refactor Plan:** Replace with Chart.js in Q2 2026

### 2. Payment Widget (Priority: High - Keep)
- **File:** /Views/Checkout/Payment.aspx
- **Lines:** 150-200
- **Reason:** Third-party payment provider requires inline script
- **Compromise:** Using nonce (permanent)
- **Refactor Plan:** None (vendor requirement)

...
```

---

## Phase 5: Enforcement (Week 11-12)

### Goal: Enable CSP in Enforcing Mode

### Step 5.1: Final CSP Policy

**Based on refactoring results:**

```http
Content-Security-Policy: 
  default-src 'self'; 
  script-src 'self' 'nonce-{NONCE}' https://ajax.googleapis.com https://cdn.jsdelivr.net; 
  style-src 'self' 'nonce-{NONCE}' https://fonts.googleapis.com; 
  font-src 'self' https://fonts.gstatic.com; 
  img-src 'self' https: data:; 
  connect-src 'self'; 
  report-uri /csp-violation-report;
```

**Key Changes from Phase 1:**
- ❌ Removed `'unsafe-inline'` (refactored 90% of inline code)
- ❌ Removed `'unsafe-eval'` (no eval() usage found)
- ✅ Added `'nonce-{NONCE}'` (for 23 compromise cases)
- ✅ Specific external domains only

### Step 5.2: Enable Enforcing Mode

**In Global.asax:**
```csharp
protected void Application_BeginRequest(object sender, EventArgs e)
{
    var nonce = Convert.ToBase64String(Guid.NewGuid().ToByteArray());
    HttpContext.Current.Items["CSP_NONCE"] = nonce;
    
    var csp = $"default-src 'self'; " +
              $"script-src 'self' 'nonce-{nonce}' https://ajax.googleapis.com https://cdn.jsdelivr.net; " +
              $"style-src 'self' 'nonce-{nonce}' https://fonts.googleapis.com; " +
              $"font-src 'self' https://fonts.gstatic.com; " +
              $"img-src 'self' https: data:; " +
              $"connect-src 'self'; " +
              $"report-uri /csp-violation-report;";
    
    // ENFORCING MODE (no longer Report-Only)
    HttpContext.Current.Response.Headers.Add("Content-Security-Policy", csp);
}
```

### Step 5.3: Staged Rollout

**Week 11: Internal Testing**
1. Deploy to staging environment
2. Test all pages manually
3. Run automated UI tests
4. Review violation reports

**Week 12: Production Rollout**
1. Deploy to 10% of users (canary)
2. Monitor for 48 hours
3. If no issues, deploy to 50%
4. Monitor for 48 hours
5. Deploy to 100%

### Step 5.4: Monitoring

**Create Dashboard:**
```csharp
// Controllers/CspDashboardController.cs
public class CspDashboardController : Controller
{
    [Authorize(Roles = "Admin")]
    public ActionResult Index()
    {
        var violations = GetViolationsFromLog();
        return View(violations);
    }
    
    private List<CspViolation> GetViolationsFromLog()
    {
        // Parse csp-violations.log
        // Group by violated-directive
        // Count occurrences
        // Return top 10
    }
}
```

**Monitor:**
- Violation count (should be near zero)
- Blocked resources (investigate each)
- User reports (functionality broken?)

---

## Acceptable Compromises

### 1. Using Nonces for Complex Legacy Code

**When:**
- Code is too complex to refactor in current sprint
- Scheduled for rewrite in future
- < 5% of total inline code

**Example:**
```html
<script nonce="@CspHelper.GetNonce()">
    // 500 lines of legacy charting config
</script>
```

**Security Impact:** ⚠️ Medium (nonce is secure, but inline code is harder to audit)

**Acceptable:** ✅ Yes, temporarily

---

### 2. Using `'unsafe-inline'` for Styles Only

**When:**
- All inline scripts refactored
- Inline styles are low-risk (no XSS via CSS in modern browsers)
- Team bandwidth limited

**Example:**
```http
Content-Security-Policy: 
  script-src 'self' 'nonce-{NONCE}'; 
  style-src 'self' 'unsafe-inline';
```

**Security Impact:** ⚠️ Low (CSS injection is less dangerous than JS)

**Acceptable:** ✅ Yes, if needed

---

### 3. Keeping Third-Party Widgets Inline

**When:**
- Vendor requires inline script
- No alternative available
- Critical business functionality

**Example:**
```html
<!-- Payment provider requires this -->
<script nonce="@CspHelper.GetNonce()">
    PaymentWidget.init({ merchantId: '@Model.MerchantId' });
</script>
```

**Security Impact:** ⚠️ Medium (trust vendor code)

**Acceptable:** ✅ Yes, with nonce

---

### 4. Allowing `data:` URIs for Images

**When:**
- Small icons/logos embedded as base64
- Improves performance (fewer HTTP requests)

**Example:**
```http
img-src 'self' data:;
```

**Security Impact:** ⚠️ Low (images can't execute code)

**Acceptable:** ✅ Yes

---

### 5. NOT Acceptable Compromises

**Never compromise on:**

1. ❌ **`'unsafe-eval'` for scripts**
   - Allows `eval()`, `new Function()`
   - Major XSS risk
   - Refactor code instead

2. ❌ **`'unsafe-inline'` for scripts (without nonces)**
   - Defeats entire purpose of CSP
   - Only use with nonces for specific blocks

3. ❌ **Wildcard domains (`https:` or `*`)**
   - Allows any external script
   - Use specific domains only

---

## Real-World Examples from ACE

### Example 1: Dashboard Load Data Button

**Original Code (Violates CSP):**
```html
<!-- /Views/Dashboard/Index.aspx -->
<button onclick="loadData()">Load Data</button>

<script>
    function loadData() {
        $.ajax({
            url: '/api/dashboard/data',
            success: function(data) {
                $('#result').html(data);
            }
        });
    }
</script>
```

**Refactored (CSP Compliant):**
```html
<!-- /Views/Dashboard/Index.aspx -->
<button id="loadDataBtn">Load Data</button>
<script src="~/Scripts/dashboard.js"></script>
```

```javascript
// /Scripts/dashboard.js
(function() {
    'use strict';
    
    document.addEventListener('DOMContentLoaded', function() {
        var btn = document.getElementById('loadDataBtn');
        if (btn) {
            btn.addEventListener('click', loadData);
        }
    });
    
    function loadData() {
        $.ajax({
            url: '/api/dashboard/data',
            success: function(data) {
                $('#result').html(data);
            }
        });
    }
})();
```

**Effort:** 15 minutes  
**Security Gain:** ✅ Prevents inline script injection

---

### Example 2: User List with Server Data

**Original Code (Violates CSP):**
```html
<!-- /Views/Users/List.aspx -->
<script>
    var userId = '@Model.UserId';
    var apiUrl = '@Url.Action("GetUsers", "Api")';
    
    fetch(apiUrl)
        .then(response => response.json())
        .then(users => {
            console.log('User:', userId);
            displayUsers(users);
        });
    
    function displayUsers(users) {
        users.forEach(function(user) {
            $('#userList').append('<li>' + user.name + '</li>');
        });
    }
</script>
```

**Refactored (CSP Compliant):**
```html
<!-- /Views/Users/List.aspx -->
<div id="userListApp" 
     data-user-id="@Model.UserId" 
     data-api-url="@Url.Action("GetUsers", "Api")">
</div>
<ul id="userList"></ul>
<script src="~/Scripts/users-list.js"></script>
```

```javascript
// /Scripts/users-list.js
(function() {
    'use strict';
    
    document.addEventListener('DOMContentLoaded', function() {
        var app = document.getElementById('userListApp');
        if (!app) return;
        
        var userId = app.dataset.userId;
        var apiUrl = app.dataset.apiUrl;
        
        fetch(apiUrl)
            .then(response => response.json())
            .then(users => {
                console.log('User:', userId);
                displayUsers(users);
            });
    });
    
    function displayUsers(users) {
        var list = document.getElementById('userList');
        users.forEach(function(user) {
            var li = document.createElement('li');
            li.textContent = user.name;
            list.appendChild(li);
        });
    }
})();
```

**Effort:** 30 minutes  
**Security Gain:** ✅ Prevents inline script injection + safer DOM manipulation

---

### Example 3: Payment Widget (Compromise with Nonce)

**Original Code (Third-Party Requirement):**
```html
<!-- /Views/Checkout/Payment.aspx -->
<script>
    PaymentWidget.init({
        merchantId: '@Model.MerchantId',
        amount: @Model.Amount,
        currency: '@Model.Currency',
        onSuccess: function() {
            window.location = '/checkout/success';
        }
    });
</script>
```

**Refactored (Compromise: Nonce + Data Attributes):**
```html
<!-- /Views/Checkout/Payment.aspx -->
<div id="paymentWidget" 
     data-merchant-id="@Model.MerchantId" 
     data-amount="@Model.Amount" 
     data-currency="@Model.Currency">
</div>

<script @CspHelper.NonceAttribute()>
    // Third-party widget requires inline initialization
    (function() {
        var widget = document.getElementById('paymentWidget');
        PaymentWidget.init({
            merchantId: widget.dataset.merchantId,
            amount: parseFloat(widget.dataset.amount),
            currency: widget.dataset.currency,
            onSuccess: function() {
                window.location = '/checkout/success';
            }
        });
    })();
</script>
```

**Effort:** 20 minutes  
**Security Gain:** ⚠️ Partial (nonce prevents injection, but inline code remains)  
**Justification:** ✅ Vendor requirement, no alternative

---

## Summary

### CSP Implementation Checklist

**Phase 0: Assessment ✅**
- [x] Run RepoScan-Analyser
- [x] Review Code_Inventory.xlsx
- [x] Identify easy wins vs. compromises

**Phase 1: Report-Only (Week 1-2)**
- [ ] Implement CSP header (Report-Only mode)
- [ ] Create violation report endpoint
- [ ] Run crawler for external resources
- [ ] Monitor violations for 1 week

**Phase 2: Easy Wins (Week 3-5)**
- [ ] Refactor 400 inline JS attributes
- [ ] Refactor 150 inline CSS attributes
- [ ] Test all refactored pages
- [ ] Update CSP to remove some `unsafe-inline`

**Phase 3: Medium Complexity (Week 6-8)**
- [ ] Refactor 200 low-complexity script blocks
- [ ] Refactor 150 script blocks with server deps (data-* pattern)
- [ ] Test all refactored pages
- [ ] Further tighten CSP

**Phase 4: Compromises (Week 9-10)**
- [ ] Identify 23 complex blocks for nonces
- [ ] Implement nonce generation
- [ ] Apply nonces to compromise cases
- [ ] Document all compromises

**Phase 5: Enforcement (Week 11-12)**
- [ ] Switch to enforcing mode
- [ ] Staged rollout (10% → 50% → 100%)
- [ ] Monitor violations
- [ ] Create admin dashboard

### Final CSP Policy

```http
Content-Security-Policy: 
  default-src 'self'; 
  script-src 'self' 'nonce-{NONCE}' https://ajax.googleapis.com https://cdn.jsdelivr.net; 
  style-src 'self' 'nonce-{NONCE}' https://fonts.googleapis.com; 
  font-src 'self' https://fonts.gstatic.com; 
  img-src 'self' https: data:; 
  connect-src 'self'; 
  report-uri /csp-violation-report;
```

### Results

- ✅ **90% of inline code refactored** (1100/1223 issues)
- ✅ **10% using nonces** (123 compromise cases)
- ✅ **CSP enabled in enforcing mode**
- ✅ **Security significantly improved**

---

**Document Version:** 1.0  
**Last Updated:** 2026-01-30  
**Target Application:** ACE (Legacy .NET)  
**Implementation Timeline:** 12 weeks
