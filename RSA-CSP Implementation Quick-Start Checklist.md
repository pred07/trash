# ✅ CSP Implementation Quick-Start Checklist

**Print this and keep it on your desk!**

---

## 📋 Phase 1: Understanding (Day 1)

```
□ Open Code_Inventory.xlsx
□ Review Summary sheet - note the counts
□ Read "CSP_IMPLEMENTATION_GUIDE_FOR_DEVELOPERS.md"
□ Understand what CSP is and why we need it
□ Set up test environment
```

---

## 🔧 Phase 2: Enable Report-Only Mode (Day 2)

```
□ Add Content-Security-Policy-Report-Only header
□ Set to: default-src 'self';
□ Test in browser - open DevTools Console
□ Verify violations are being reported (not blocked)
```

**ASP.NET web.config:**
```xml
<add name="Content-Security-Policy-Report-Only" value="default-src 'self';" />
```

---

## 📝 Phase 3: Fix Inline JavaScript (Week 1-2)

### Inline JS Attributes (onclick, onload, etc.)

```
□ Open "Inline JS (Attributes)" sheet
□ For each row:
  □ Find the file (Column: File Path)
  □ Go to the line (Column: Line Start)
  □ Move code to external JS file
  □ Add event listener instead
  □ Test the functionality
  □ Mark as DONE in Excel
```

**Pattern:**
```html
<!-- BEFORE -->
<button onclick="doSomething()">Click</button>

<!-- AFTER -->
<button id="myBtn">Click</button>
<script src="/js/app.js"></script>

<!-- In app.js -->
document.getElementById('myBtn').addEventListener('click', doSomething);
```

---

### Internal JS Blocks

```
□ Open "Internal JS (Blocks)" sheet
□ For each row:
  □ Copy extracted file from output/Extracted_Code/internal_js/
  □ Paste into /js/ folder
  □ Replace <script> block with <script src="/js/filename.js">
  □ Test the functionality
  □ Mark as DONE in Excel
```

**Pattern:**
```html
<!-- BEFORE -->
<script>
  $(document).ready(function() { init(); });
</script>

<!-- AFTER -->
<script src="/js/init.js"></script>
```

---

## 🎨 Phase 4: Fix Inline CSS (Week 3)

```
□ Open "Inline CSS (Attributes)" sheet
□ Create /css/utilities.css
□ For each row:
  □ Convert style="..." to CSS class
  □ Add class to utilities.css
  □ Replace inline style with class name
  □ Test the styling
  □ Mark as DONE in Excel
```

**Pattern:**
```html
<!-- BEFORE -->
<div style="color: red; font-weight: bold;">Error</div>

<!-- AFTER -->
<div class="text-red text-bold">Error</div>

/* In utilities.css */
.text-red { color: red; }
.text-bold { font-weight: bold; }
```

---

## 🌐 Phase 5: Fix AJAX & Dynamic Code (Week 4)

### AJAX Calls

```
□ Open "AJAX Code" sheet
□ For each "Has Server Dependencies: Yes":
  □ Note the endpoint
  □ Add to CSP connect-src whitelist
  □ Mark as DONE in Excel
```

### Dynamic Code (eval, setTimeout with strings)

```
□ Open "Dynamic Code" sheet
□ For each eval():
  □ Refactor to remove eval()
  □ Use JSON.parse() for JSON
  □ Use function references for setTimeout
  □ Mark as DONE in Excel
```

**Pattern:**
```javascript
// BEFORE (Bad)
eval('var x = 10;');
setTimeout('doSomething()', 1000);

// AFTER (Good)
var x = 10;
setTimeout(doSomething, 1000);
```

---

## 🔐 Phase 6: Build CSP Header (Week 5)

```
□ Open "Whitelist Recommendations" sheet
□ Copy recommended directives
□ Build CSP header
□ Add to web.config or middleware
□ Keep in Report-Only mode
□ Test thoroughly
```

**Template:**
```http
Content-Security-Policy-Report-Only:
  default-src 'self';
  script-src 'self' https://code.jquery.com;
  style-src 'self' https://fonts.googleapis.com;
  connect-src 'self';
  img-src 'self' data: https:;
  font-src 'self' https://fonts.gstatic.com;
  object-src 'none';
```

---

## 🧪 Phase 7: Testing (Week 6)

```
□ Test all pages load
□ Test all buttons work
□ Test all forms submit
□ Test all AJAX calls succeed
□ Test in Chrome
□ Test in Firefox
□ Test in Safari
□ Test in Edge
□ Test on mobile
□ Check DevTools Console - should be ZERO violations
```

**Zero violations = Ready for enforcement!**

---

## 🚀 Phase 8: Production Deployment (Week 7-8)

### Week 7: Gradual Rollout

```
□ Deploy to 10% of users
□ Monitor for violations
□ Fix any issues
□ Deploy to 50% of users
□ Monitor for violations
□ Verify no performance impact
```

### Week 8: Full Enforcement

```
□ Change header from Report-Only to enforcement:
  Content-Security-Policy-Report-Only → Content-Security-Policy
□ Deploy to 100% of users
□ Monitor closely for 48 hours
□ Set up CSP violation reporting endpoint
□ Document final CSP policy
□ Celebrate! 🎉
```

---

## ⚠️ NEVER DO THIS!

```
❌ script-src 'unsafe-inline'  (defeats CSP!)
❌ script-src 'unsafe-eval'    (allows eval!)
❌ script-src *                (allows all domains!)
❌ Skip Report-Only mode       (will break app!)
❌ Test only in Chrome         (browser differences!)
```

---

## 🆘 Troubleshooting

### "My page is blank!"
```
→ Check DevTools Console for CSP violations
→ You're probably in enforcement mode too early
→ Switch back to Report-Only mode
→ Fix violations first
```

### "AJAX calls are failing!"
```
→ Check "AJAX Code" sheet
→ Add endpoints to connect-src
→ Example: connect-src 'self' /api/*;
```

### "Styles are missing!"
```
→ Check for inline styles
→ Move to external CSS
→ Check for external CSS CDNs
→ Add to style-src
```

### "Scripts not loading!"
```
→ Check for inline <script> blocks
→ Move to external JS files
→ Check for external CDNs
→ Add to script-src
```

---

## 📊 Progress Tracker

**Week 1-2: Inline JS**
```
Total: ____ findings
Fixed: ____ 
Remaining: ____
Progress: ____%
```

**Week 3: Inline CSS**
```
Total: ____ findings
Fixed: ____ 
Remaining: ____
Progress: ____%
```

**Week 4: AJAX & Dynamic**
```
Total: ____ findings
Fixed: ____ 
Remaining: ____
Progress: ____%
```

**Week 5: CSP Header**
```
□ Header built
□ Added to config
□ Tested in Report-Only
```

**Week 6: Testing**
```
□ All browsers tested
□ Zero violations
□ Ready for production
```

**Week 7-8: Production**
```
□ 10% rollout
□ 50% rollout
□ 100% rollout
□ Enforcement enabled
```

---

## 🎯 Daily Checklist

**Every Day:**
```
□ Fix 5-10 violations
□ Test each fix
□ Update progress tracker
□ Commit changes to Git
□ Document any issues
```

**Every Week:**
```
□ Review progress with team
□ Update stakeholders
□ Adjust timeline if needed
□ Celebrate wins!
```

---

## 📞 Need Help?

**Resources:**
- CSP_IMPLEMENTATION_GUIDE_FOR_DEVELOPERS.md (detailed guide)
- Code_Inventory.xlsx (your audit report)
- output/Extracted_Code/ (pre-extracted code)
- output/Annotated_Source/ (commented source)

**Online:**
- MDN CSP Guide: https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
- CSP Evaluator: https://csp-evaluator.withgoogle.com/
- OWASP CSP Cheat Sheet: https://cheatsheetseries.owasp.org/

---

## ✅ Final Reminder

**CSP Implementation = Security Win!**

You're making your application:
- ✅ Resistant to XSS attacks
- ✅ More secure for users
- ✅ Compliant with security standards
- ✅ Ready for modern web

**Take it one step at a time. You've got this! 🚀**

---

**Print Date:** _____________
**Developer:** _____________
**Target Completion:** _____________
