# RepoScan: Technical Action & Command Reference

This document details every command, mode, input, and output artifact generated during the CSP implementation process.

---

## 📅 Phase 0: Diagnostic (Assessment)

**Goal:** Create the initial "Code Inventory" and identify external resource dependencies.

| Step | Command / Action | Inputs (Args) | Outputs Generated | Explanation & Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **0.1** | `python RepoScan-Analyser/main.py --all --root "C:\Src\MyApp" --url "http://localhost:5000" --output "Output/Scan"` | **Input:** Source Code Folder (`--root`)<br>**Mode:** `--all` (Static + Dynamic)<br>**Helper:** Local URL (`--url`) for crawler | **1. Master Inventory:** `Output/Scan/Code_Inventory.xlsx`<br>**2. Resource List:** `Output/Scan/Dynamic_Analysis_Report.xlsx`<br>**3. Raw Assets:** `Output/Scan/extracted_code/*` | **"The MRI Scan"**<br>Scans the code and crawls the running app. It creates a complete list of every inline script and external link (fonts, CDNs). |
| **0.2** | `python refactoring_utility/progress_tracker.py --tracker "Output/Project_Tracker.xlsx" --import-analysis "Output/Scan/Code_Inventory.xlsx"` | **Input:** Inventory Excel (`--import-analysis`)<br>**Target:** New Tracker File (`--tracker`) | **1. Dashboard:** `Output/Project_Tracker.xlsx`<br>*(Status: 0% Complete)* | **"The Dashboard"**<br>Converts the technical inventory into a management spreadsheet with status columns (Open, In Progress, Done). |

---

## 📅 Phase 1: Baseline (Monitoring)

**Goal:** Implement a safe "Report-Only" policy to gather real-world data without breaking the site.

| Step | Command / Action | Inputs (Args) | Outputs Generated | Explanation & Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **1.1** | `python refactoring_utility/csp_generator.py --allowlist "Output/Scan/Dynamic_Analysis_Report.xlsx" --output "Output/Config" --mode report-only` | **Input:** Dynamic Report (`--allowlist`)<br>**Mode:** `--mode report-only`<br>**Target:** Config Folder (`--output`) | **1. Web Config:** `Output/Config/web.config.report-only.xml`<br>**2. Helper Class:** `Output/Config/CspHelper.cs` | **"The Safety Net"**<br>Reads the list of external domains found by the crawler (e.g., Google Fonts) and writes a valid XML configuration file aimed at *logging only*. |
| **1.2** | **Deploy:** Copy XML content to `web.config`. | **Input:** XML content from Step 1.1 | **1. Active Headers:** Browser detects `Content-Security-Policy-Report-Only` | **"Go Live (Safe)"**<br>The site is now monitoring for attacks/errors. No features are blocked. |

---

## 📅 Phase 2: Attribute Tagging (Identification)

**Goal:** Locate and mark inline attributes (`onclick`, `style`) so developers can fix them.

| Step | Command / Action | Inputs (Args) | Outputs Generated | Explanation & Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **2.1** | `python refactoring_utility/batch_executor.py --analysis "Output/Scan/Code_Inventory.xlsx" --source "C:\Src\MyApp" --output "Output/Refactored_Phase1" --mode dry-run --phase phase1_attributes` | **Input:** Source Code (`--source`)<br>**Inventory:** Analysis Excel (`--analysis`)<br>**Mode:** `--mode dry-run` | **1. Console Log:** Lists all files that *would* be touched.<br>**2. No Files Changed:** Output folder is empty or copy only. | **"The Rehearsal"**<br>Simulates the tagging process. Use this to ensure the tool sees the correct files before letting it write anything. |
| **2.2** | `python refactoring_utility/batch_executor.py ... --output "Output/Refactored_Phase1" --mode apply --phase phase1_attributes` | **Input:** Source Code (`--source`)<br>**Target:** Output Copy (`--output`)<br>**Mode:** `--mode apply` | **1. Refactored Code:** `Output/Refactored_Phase1/*`<br>*(Contains `<!-- TODO -->` comments)*<br>**2. Changelog:** `Output/Refactored_Phase1/refactoring_changelog.txt` | **"Mark the Targets"**<br>Creates a *copy* of your app in the output folder. It injects comments right above every issue: "TODO: Fix this onclick". |
| **2.3** | `python refactoring_utility/progress_tracker.py --tracker "Output/Project_Tracker.xlsx" --update-changelog "Output/Refactored_Phase1/refactoring_changelog.txt"` | **Input:** Changelog (`--update-changelog`)<br>**Target:** Tracker file | **1. Updated Dashboard:** Tracker statuses updated to **"In Progress"**. | **"Sync Status"**<br>Tells project management that these items have been identified and assigned to developers. |

---

## 📅 Phase 3: Block Extraction (Automation)

**Goal:** Automatically move large, safe scripts and styles to external files.

| Step | Command / Action | Inputs (Args) | Outputs Generated | Explanation & Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **3.1** | `python refactoring_utility/batch_executor.py ... --mode dry-run --phase phase2_safe_blocks` | **Mode:** `--mode dry-run` | **1. Console Log:** "Would externalize: index.html:58" | **"Safety Check"**<br>Verifies which blocks are safe to move vs. which ones have server code (`[BLOCKED]`). |
| **3.2** | `python refactoring_utility/batch_executor.py --source "C:\Src\MyApp" --output "Output/Refactored_Phase2" --mode apply --phase phase2_safe_blocks` | **Input:** Source Code (`--source`)<br>**Target:** New Output Folder (`--output`)<br>**Mode:** `--mode apply` | **1. JS Files:** `Output/Refactored_Phase2/public/js/*.js`<br>**2. CSS Files:** `Output/Refactored_Phase2/public/css/*.css`<br>**3. HTML:** `<script>` tags replaced with `<script src="...">` | **"Heavy Lifting"**<br>The tool physically cuts the code from the HTML and pastes it into new `.js` files. It then updates the HTML to link to them. |

---

## 📅 Phase 4: Manual Refactoring (Developer)

**Goal:** Address the complex items that automation (Phase 2 & 3) flagged but couldn't fix automatically.

| Step | Command / Action | Inputs (Args) | Outputs Generated | Explanation & Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **4.1** | **Developer searches** for `TODO` tags in code. | Input: `Output/Refactored_Phase2` (Latest Code) | **1. Common Logic:** `common-csp.js`<br>**2. Clean HTML:** `onclick` attributes removed. | **"The Clean Up"**<br>Developers do the skilled work: rewriting logic to separate structure (HTML) from behavior (JS). |
| **4.2** | **Manual Tracker Update** | Input: Excel Sheet | **1. Updated Dashboard:** Item marked **"Completed"**. | **"Definition of Done"**<br>Signals that the feature is tested and secure. |

---

## 📅 Phase 5: Enforcement (Final Security)

**Goal:** Enable strict blocking of unauthorized code.

| Step | Command / Action | Inputs (Args) | Outputs Generated | Explanation & Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **5.1** | `python refactoring_utility/csp_generator.py ... --mode enforcement --disable-nonce` | **Mode:** `--mode enforcement`<br>**Flag:** `--disable-nonce` (For Static Apps) | **1. Strict Config:** `Output/Config/web.config.enforcement.xml` | **"Lock It Down"**<br>Generates the strictest possible policy (`default-src 'self'`). |
| **5.2** | **Deploy:** Replace `web.config` headers. | Input: XML content | **1. Secure App** | **"Mission Accomplished"**<br>The application now rejects any script that isn't from your own server. Total XSS protection. |
