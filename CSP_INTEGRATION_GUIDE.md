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
Import the analysis results into a master tracking grid.

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
Use the findings to generate a baseline policy.

```powershell
python refactoring_utility/csp_generator.py --allowlist "Output/Scan_Results/Dynamic_Analysis_Report.xlsx" --output "Output/Config" --mode report-only
```

### 2. Integration
1.  **Web Config:** Copy `Output/Config/web.config.report-only.xml` content into your application's `web.config` inside `<customHeaders>`.
2.  **Violation Endpoint:** Ensure your app has a route (e.g., `/csp-violation-report`) to catch log entries (as defined in the generated config).

---

## 📅 Phase 2: Refactor Attributes (Week 3-5)

**Objective:** Eliminate "Easy Wins" (Inline `onclick`, `style` attributes). 
*Note: The tool marks these locations; developers implement the logic files.*

### 1. Run "Mark and Track"
This command scans for attributes and inserts `<!-- TODO: Refactor... -->` comments, and logs them to the changelog.

```powershell
# Dry Run (Preview)
python refactoring_utility/batch_executor.py --analysis "Output/Scan_Results/Code_Inventory.xlsx" --source "C:\Source" --output "Output/Refactored_Phase1" --mode dry-run --phase phase1_attributes

# Apply (Modifies Code)
python refactoring_utility/batch_executor.py --mode apply --phase phase1_attributes
```

### 2. Developer Action
1.  Open `Output/Refactored_Phase1/refactoring_changelog.txt`.
2.  Search for **"TODO COMMENT ADDED"**.
3.  **Action:** Move the logic from the attribute to a central `event-handlers.js` file (as per the Architecture Guide) and remove the attribute.
4.  **Update Tracker:**
    ```powershell
    python refactoring_utility/progress_tracker.py --tracker "Output/CSP_Project_Tracker.xlsx" --update-changelog "Output/Refactored_Phase1/refactoring_changelog.txt"
    ```

---

## 📅 Phase 3: Automated Extraction (Week 6-8)

**Objective:** Extract `<script>` and `<style>` blocks to external files.
*Note: The tool AUTOMATES this. It extracts the code to `.js` files and replaces the block with `<script src="...">`.*

### 1. Execute Extraction
The tool intelligently skips blocks with server-side dependencies (`@Model`, `<%= %>`).

```powershell
python refactoring_utility/batch_executor.py --mode apply --phase phase2_safe_blocks --output "Output/Refactored_Phase2"
```

### 2. Developer Verification
1.  **Check Blocked Items:** Review `refactoring_changelog.txt` for `[BLOCKED]` entries. These contained server-side logic and were skipped (left inline).
2.  **Verify Externalization:** Check `[EXTERNALIZED]` entries. The code is now in `extracted_code/`.
3.  **Include Files:** The tool adds `<script src="...">` automatically. Ensure these paths resolve correctly in your specific ASP.NET routing configuration.

---

## 📅 Phase 4: Nonce Injection (Week 9-10)

**Objective:** Secure the remaining complex/legacy blocks that couldn't be extracted.

### 1. Install Helper Class
Add the generated C# helper to your project.
*   **Source:** `Output/Config/CspHelper.cs`
*   **Action:** Copy to `App_Code/` or your Utilities folder.

### 2. Inject Nonces
Run the simpler tool to convert blocked scripts to nonce-enabled scripts.

```powershell
python refactoring_utility/batch_executor.py --mode apply --phase phase3_nonces --output "Output/Refactored_Phase3"
```

*What this does:* It finds remaining `<script>` tags (detected as "unsafe" in Phase 3) and injects `nonce="@CspHelper.GetNonce()"`.

### 3. Verification
*   Ensure `CspHelper.GetNonce()` integrates with your specific framework (Razor vs WebForms). You may need to tweak the `CspHelper.cs` namespace.

---

## 📅 Phase 5: Enforcement (Week 11-12)

**Objective:** Flip the switch.

### 1. Generate Strict Config
Regenerate the config based on the final clean state.

```powershell
python refactoring_utility/csp_generator.py --allowlist "Output/Scan_Results/Dynamic_Analysis_Report.xlsx" --output "Output/Config_Final" --mode enforcement
```

### 2. Deploy
1.  Replace `web.config` headers with `web.config.enforcement.xml`.
2.  Validate using the **Dashboard** in `Output/CSP_Project_Tracker.xlsx`. All items should be "Completed" or "Skipped".

---

## ⚠️ Known Limitations & Developer Responsibilities

1.  **Logic Logic:** The `batch_executor` does not rewrite JavaScript logic. If you extract a script that relies on a variable defined in another inline script *below* it, it may break. **Always test dependencies.**
2.  **Dynamic HTML:** If your C# code generates HTML strings containing `<script>`, the Static Scanner cannot detect them easily. Use the **Dynamic Analysis** report to catch these execution violations.
3.  **Event Handling:** Refactoring `onclick` (Phase 2) requires manual developer effort to write `addEventListener`. The tool only highlights the location.

---

**Support:**
For tool errors, check `refactoring_utility/README.md`.
For project status, verify `Output/CSP_Project_Tracker.xlsx`.
