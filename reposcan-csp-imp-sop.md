# CSP Integration Guide: Empowering Deployment with RepoScan

**Target Audience:** Development Team, Security Leads, DevOps  
**Version:** 1.0  
**Tools Required:** RepoScan Toolkit (Python 3.8+)

---

## Executive Summary

This manual prescribes the standard operating procedure (SOP) for implementing Content Security Policy (CSP) using the **RepoScan** automated toolkit. 

The CSP implementation is divided into **5 Phases** over **12 Weeks**. RepoScan automates the discovery, extraction, refactoring, and configuration generation, typically reducing manual effort by **60-70%**.

---

## 🛠️ Tool Usage Overview

| Utility | Command Script | Purpose |
|---------|----------------|---------|
| **Analyser** | `RepoScan-Analyser/main.py` | Scans codebase, inventories issues, detects external assets. |
| **Tracker** | `refactoring_utility/progress_tracker.py` | Project management, dashboarding, and burndown charts. |
| **Executor** | `refactoring_utility/batch_executor.py` | **The Workhorse**. Automates code modification (comments, extraction, nonces). |
| **Generator** | `refactoring_utility/csp_generator.py` | Creates `web.config` headers and C# helper classes. |

---

## 📅 Phase 0: Diagnostic Scan (Day 1)

**Objective:** Map the battlefield. Identify every line of code involving inline JS, CSS, or external calls.

### 1. Run the Multi-Vector Analysis
Execute the scanner against your application root. The `--all` flag ensures both static code analysis and dynamic crawling (for external assets like fonts/CDNs).

```powershell
python RepoScan-Analyser/main.py --all --root "C:\Path\To\Your\SourceCode" --url "http://localhost:5000" --output "Output/Scan_Results"
```

### 2. Initialize the Project Tracker
Import the analysis results into a master tracking grid. This Excel file will be your "Source of Truth" for progress.

```powershell
python refactoring_utility/progress_tracker.py --tracker "Output/CSP_Project_Tracker.xlsx" --import-analysis "Output/Scan_Results/Code_Inventory.xlsx"
```

### 3. Deliverable Analysis
Open `Output/Scan_Results/Code_Inventory.xlsx` and review:
*   **Inline JS (Attributes):** Volume of `onclick`, `onload`. (Target: Phase 2)
*   **Internal JS (Blocks):** Volume of `<script>` tags. (Target: Phase 3)
*   **Dynamic_Analysis_Report.xlsx:** Check "Failed Resources" tab to see if any assets (fonts, images) are 404ing or need whitelisting.

---

## 📅 Phase 1: Baseline & Report-Only (Week 1-2)

**Objective:** Deploy a "monitoring" policy that breaks nothing but logs violations.

### 1. Generate Configuration
Use the findings to generate a baseline policy that accounts for all observed external domains.

```powershell
python refactoring_utility/csp_generator.py --allowlist "Output/Scan_Results/Dynamic_Analysis_Report.xlsx" --output "Output/Config" --mode report-only
```

### 2. Integration
1.  **Web Config:** Copy the `<add name="Content-Security-Policy-Report-Only" ... />` line from `Output/Config/web.config.report-only.xml` into your application's `web.config` inside the `<system.webServer> <httpProtocol> <customHeaders>` section.
2.  **Violation Endpoint:** Ensure your app has a route or a listener (e.g., `/csp-violation-report`) to catch log entries sent by the browser.

---

## 📅 Phase 2: Refactor Attributes (Week 3-5)

**Objective:** Eliminate "Easy Wins" (Inline `onclick`, `style` attributes). 
*Note: The tool marks these locations; developers implement the logic files.*

### 1. Run "Mark and Track"
This command scans for attributes and inserts `<!-- TODO: [CSP] Refactor... -->` comments directly into the source code, and logs them to the changelog.

```powershell
# Dry Run (Preview changes without modifying files)
python refactoring_utility/batch_executor.py --analysis "Output/Scan_Results/Code_Inventory.xlsx" --source "C:\Source" --output "Output/Refactored_Phase1" --mode dry-run --phase phase1_attributes

# Apply (Modifies Code in the Output directory)
python refactoring_utility/batch_executor.py --mode apply --phase phase1_attributes
```

### 2. Developer Action (The "Human" Step)
1.  Open `Output/Refactored_Phase1/refactoring_changelog.txt`.
2.  Search for **"TODO COMMENT ADDED"** in your IDE.
3.  **Refactoring Pattern:** Move the logic from the attribute to a central `event-handlers.js` file using event delegation.
    *   *Example:* Change `<button onclick="doSomething()">` to `<button data-click="doSomething">`.
4.  **Update Tracker:** Sync your manual changes back to the project tracker.
    ```powershell
    python refactoring_utility/progress_tracker.py --tracker "Output/CSP_Project_Tracker.xlsx" --update-changelog "Output/Refactored_Phase1/refactoring_changelog.txt"
    ```

---

## 📅 Phase 3: Automated Extraction (Week 6-8)

**Objective:** Extract `<script>` and `<style>` blocks to external files.
*Note: The tool AUTOMATES this. It extracts the code to `.js` files and replaces the original block with a `<script src="...">` tag.*

### 1. Execute Extraction
The tool intelligently skips blocks containing server-side dependencies like Razor (`@Model`) or ASP (`<%= %>`) to avoid breaking the build.

```powershell
python refactoring_utility/batch_executor.py --mode apply --phase phase2_safe_blocks --output "Output/Refactored_Phase2"
```

### 2. Developer Verification
1.  **Check Blocked Items:** Review `refactoring_changelog.txt` for `[BLOCKED]` entries. These contained server-side logic and were left inline for safety.
2.  **Verify Externalization:** Check the `extracted_code/` folder in your output. You should see files like `index_html_scriptblock_L50.js`.
3.  **Path Resolution:** Ensure the newly created tags (e.g., `<script src="/js/...">`) align with your web server's static file routing.

---

## 📅 Phase 4: Nonce Injection (Week 9-10)

**Objective:** Secure the remaining complex/legacy blocks that could not be safely extracted in Phase 3.

### 1. Install Helper Class
Add the generated C# helper class to your solution to manage nonce generation per request.
*   **Source:** `Output/Config/CspHelper.cs`
*   **Action:** Copy to your `Utilities` or `App_Code` folder.

### 2. Inject Nonces
Convert the "Blocked" scripts from Phase 3 into nonce-enabled scripts.

```powershell
python refactoring_utility/batch_executor.py --mode apply --phase phase3_nonces --output "Output/Refactored_Phase3"
```

*What this does:* It replaces `<script>` tags with `<script nonce="@CspHelper.GetNonce()">`. The CSP policy will now allow these specific inline scripts because they carry a cryptographically strong, single-use token.

---

## 📅 Phase 5: Enforcement (Week 11-12)

**Objective:** Flip the switch from monitoring to active protection.

### 1. Generate Strict Config
Regenerate the final, strict CSP configuration. This policy will no longer allow `unsafe-inline` or `unsafe-eval` unless nonced.

```powershell
python refactoring_utility/csp_generator.py --allowlist "Output/Scan_Results/Dynamic_Analysis_Report.xlsx" --output "Output/Config_Final" --mode enforcement
```

### 2. Final Deployment
1.  **Switch Headers:** Replace the `Content-Security-Policy-Report-Only` header in `web.config` with the `Content-Security-Policy` header from `Output/Config_Final/web.config.enforcement.xml`.
2.  **Final Audit:** Check the **Dashboard** in `Output/CSP_Project_Tracker.xlsx`. All items should be marked as "Completed" or "Nonced".

---

## ⚠️ Known Limitations & Developer Responsibilities

1.  **Dependency Ordering:** While RepoScan extracts scripts, it cannot detect if `Script_A.js` requires a global variable from `Script_B.js`. Always verify the script load order in your layout file.
2.  **Dynamic Strings:** JavaScript that constructs HTML strings via `.innerHTML = "..."` containing tags may still trigger CSP violations. These must be manually refactored to use `.textContent` or `document.createElement`.
3.  **Third-Party Plugins**: Some older jQuery plugins inject styles directly into the DOM. These may require the addition of `'unsafe-inline'` to the `style-src` directive if they cannot be nonced.

---

**Support & Documentation:**
Detailed logs for every automated change are located in the `Output/refactoring_changelog.txt`.  
For project burndown metrics, refer to the **Charts** tab in `CSP_Project_Tracker.xlsx`.
