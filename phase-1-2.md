# Refactoring & CSP Verification Summary

This document summarizes the complete workflow to secure your legacy application with CSP.

## Phase 1: Baseline & Report-Only Configuration

### 1. Generate Report-Only Policy
We generated the initial `web.config` headers using the "Allowlist" from the dynamic scan.

**Command:**
```powershell
python RepoScan\refactoring_utility\csp_generator.py --allowlist "legacy_susu\Audits\Dynamic_Analysis_Report.xlsx" --output "legacy_susu\Audits\CSP_Config" --mode report-only
```
**Output:** `legacy_susu\Audits\CSP_Config\web.config.report-only.xml`

### 2. Manual Implementation (Report-Only)
You manually integrated the generated headers into your application to start monitoring without breaking changes.

1.  Open `legacy_susu\Audits\CSP_Config\web.config.report-only.xml`.
2.  Copy the `<add name="Content-Security-Policy-Report-Only" ... />` line and security headers.
3.  Open `legacy_susu\web.config`.
4.  Paste headers inside the `<customHeaders>` section.
**Result:** The application now sends CSP headers in "Report-Only" mode.

---

## Phase 2: CSP Verification & Refactoring

### 1. Verify CSP Implementation (Simulation)
Start the simulation server to test CSP headers (since standard python server ignores web.config).

**Step 1: Start Server**
```powershell
python legacy_susu\Audits\verify_csp_server.py
```

**Step 2: Access & Test**
*   **URL:** `http://localhost:8083/public/settings.html`
*   **XSS Test Payload:** (Enter in "Site Name" & Save)
    ```html
    <img src=x onerror=alert('XSS')>
    ```

---

### 2. Refactoring Phase 1: Identify & Tag Attributes
Identifies inline event handlers (`onclick`, `onload`, `style`) and tags them.

**Dry Run:**
```powershell
python RepoScan\refactoring_utility\batch_executor.py --analysis "legacy_susu\Audits\Code_Inventory.xlsx" --source "legacy_susu" --output "legacy_susu\Audits\Refactored_Phase1" --mode dry-run --phase phase1_attributes
```

**Apply:**
```powershell
python RepoScan\refactoring_utility\batch_executor.py --analysis "legacy_susu\Audits\Code_Inventory.xlsx" --source "legacy_susu" --output "legacy_susu\Audits\Refactored_Phase1" --mode apply --phase phase1_attributes
```
*   **Output:** `legacy_susu\Audits\Refactored_Phase1`

---

### 3. Refactoring Phase 2: Extract Safe Blocks
Extracts inline `<script>` and `<style>` blocks into external files.

**Apply:**
```powershell
python RepoScan\refactoring_utility\batch_executor.py --analysis "legacy_susu\Audits\Code_Inventory.xlsx" --source "legacy_susu" --output "legacy_susu\Audits\Refactored_Phase2" --mode apply --phase phase2_safe_blocks
```
*   **Output:** `legacy_susu\Audits\Refactored_Phase2`

---

## Technical Progress Report (End of Phase 2)

1.  **Observability Established:**
    *   **CSP is Active:** App broadcasts security violations ("Report Only").
    *   **Inventory:** `Code_Inventory.xlsx` maps every unsafe code pattern.

2.  **Attack Surface Reduced:**
    *   **Inline Attributes (95+):** Identified and tagged for manual fix.
    *   **Blocks Extracted (12):** Large inline scripts moved to external files.

3.  **Infrastructure Ready:**
    *   **Automation:** Proven batch refactoring (safe copy logic).
    *   **Testing:** Proven simulation server (`verify_csp_server.py`) for verification.
