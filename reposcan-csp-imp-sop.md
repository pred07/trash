# RepoScan CSP Implementation: Standard Operating Procedure (SOP)
**Version:** 1.0  
**Target Audience:** Software Developers & Security Engineers  
**Goal:** Achieve 100% Content Security Policy (CSP) Enforcement via Automated & Manual Refactoring.

---

## 1. Introduction & Objectives
Content Security Policy (CSP) is the primary defense against Cross-Site Scripting (XSS). Implementing it on legacy applications is difficult because inline scripts and styles are blocked by default. 

The **RepoScan Toolkit** automates the identification and extraction of this code, allowing you to reach an "Enforced" state without breaking application functionality.

---

## 2. Prerequisites
Before starting, ensure you have:
*   **Python 3.8+** installed.
*   **Pandas** library (`pip install pandas`).
*   **Access to Source Code**: Ensure you have a backup of the original application, as the tool will create a refactored version.

---

## 3. Phase 1: Static Audit & Asset Mapping
The goal is to understand the "Security Debt" of the application.

### Step 1.1: Identify Vulnerabilities
Run the `RepoScan-Analyser` to find every instance of inline code.
```powershell
python RepoScan-Analyser/main.py --static --root "C:\Path\To\Your_App" --output "Output/Initial_Audit"
```
*   **Action**: Open `Output/Initial_Audit/Code_Inventory.xlsx`.
*   **What to check**: Review the **Summary** tab. Note the count of "Inline JS" and "Inline CSS". This is your work-list.

### Step 1.2: Map External Domains
Run the `repo_depth_analyser` to map every third-party URL (e.g., Google Fonts, Cloudflare).
```powershell
# Run the depth analysis tool from the scripts folder
python scripts/repo_depth_analyser.py --root "C:\Path\To\Your_App"
```
*   **Action**: Open the generated Excel report.
*   **Importance**: These domains **must** be added to your CSP `script-src` and `style-src` whitelist.

---

## 4. Phase 2: Automated Refactoring
This phase physically removes code from HTML files and moves it to external files.

### Step 2.1: Attribute Tagging (Phase 1)
Identifies every `onclick`, `onchange`, etc., and adds a marker for the developer.
```powershell
python refactoring_utility/batch_executor.py --mode apply --phase 1 --root "C:\Input_App" --output "C:\Refactored_App"
```

### Step 2.2: Block Extraction (Phase 2)
Automatically moves `<script>` and `<style>` blocks to the `/js` and `/css` directories.
```powershell
python refactoring_utility/batch_executor.py --mode apply --phase 2 --root "C:\Refactored_App" --output "C:\Refactored_App"
```
*   **Verification**: Check headers of your HTML files. You should see `<script src="/js/file_scriptblock_Lxx.js"></script>` instead of inline code.

---

## 5. Phase 3: Developer Implementation (Manual Hardening)
The tool has done the heavy lifting. Now, the developer must implement the **Security Architecture**.

### Step 3.1: Centralized Styles (`csp-styles.css`)
1.  Create `public/Content/csp-styles.css`.
2.  Search the project for `<!-- TODO: [CSP] Refactor inlinestyle ... -->`.
3.  **Action**: Move the CSS to a class in `csp-styles.css` and apply the class to the HTML element.
    *   *Example*: Replace `style="color:red"` with `class="text-danger"`.

### Step 3.2: Centralized Event Delegation (`csp-helpers.js`)
Instead of `onclick="func()"`, use "Data Attributes".
1.  Create `public/Scripts/csp-helpers.js`.
2.  Use the following template for delegation:
```javascript
document.addEventListener('click', (e) => {
    const target = e.target.closest('[data-click]');
    if (target) {
        const fnName = target.getAttribute('data-click');
        if (window[fnName]) window[fnName]();
    }
});
```
3.  **Action**: Change `<li onclick="logout()">` to `<li data-click="logout">`.

---

## 6. Phase 4: Dynamic Sink Remediation
The tool identifies "Dynamic Code Sinks" (e.g., `innerHTML`, `eval`).
*   **Action**: Scan results for `innerHTML`. 
*   **Fix**: If you are just inserting text, change `.innerHTML = var` to `.textContent = var`. This prevents HTML injection even if your CSP is bypassed.

---

## 7. Phase 5: Setting the Policy
Now that the code is clean, implement the header.

### Step 5.1: Report-Only Mode (Testing)
Add this to your `web.config` or server header:
```xml
<add name="Content-Security-Policy-Report-Only" 
     value="default-src 'self'; script-src 'self' [WHITELISTED_DOMAINS];" />
```
*   **Action**: Open the app in a browser. Press **F12** and check the **Console**.
*   **Result**: If you see "CSP Violation" errors, it means you missed an inline attribute or a domain. Fix them before moving to Step 5.2.

### Step 5.2: Enforcement Mode (Live)
Once the console is clear of errors, change the header name from `Content-Security-Policy-Report-Only` to just `Content-Security-Policy`.

---

## 8. Phase 6: Final Verification
Prove that the application is secure.

1.  Run the `RepoScan-Analyser` one last time on the **Refactored** directory.
2.  Open the final `Code_Inventory.xlsx`.
3.  **The Goal**: The count for **Inline JS** and **Inline CSS** must be **0**.

---

## 9. Troubleshooting
*   **App Breaks on Load**: Check if an extracted script had server-side tags (e.g., `<%= %>`). The tool excludes these by default, but manual verification is required.
*   **CDNs blocked**: Ensure you ran Step 1.2. Many legacy apps call `ajax.googleapis.com` without the developer knowing.

---
**Standard Operating Procedure Completed.**  
*By following this workflow, you ensure a provable, auditable, and secure CSP implementation.*
