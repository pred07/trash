# AJAX Mastery Guide: Detection, Patterns & Analysis

**Target Audience:** Beginner to Intermediate Developers  
**Goal:** Understand how to detect, categorize, and analyze AJAX calls in source code.

---

## 1. What is AJAX?

**AJAX** (Asynchronous JavaScript and XML) is a technique used to send/receive data from a server in the background without refreshing the web page.

While originally named for XML, modern AJAX almost exclusively uses **JSON** (JavaScript Object Notation).

### The Core Concept
1.  **Event Occurs:** User clicks a button or page loads.
2.  **Request Sent:** Code sends a signal to a URL (API Endpoint).
3.  **Server Processes:** Server does work (database lookup, calculation).
4.  **Response Returned:** Server sends data back.
5.  **Callback Executed:** Browser runs code to handle the data.

---

## 2. The Three Generations of AJAX Coding

To build accurate detection patterns, you must recognize all three major ways AJAX is written.

### Type A: The Legacy Way (XMLHttpRequest)
The original, raw method. Verbose and ugly, but still underpins many modern libraries.

```javascript
/* PATTERN 1: Raw XHR */
var xhr = new XMLHttpRequest();           // <--- KEYWORD 1
xhr.open("GET", "/api/user_data", true);  // <--- KEYWORD 2 (Method & URL)
xhr.onreadystatechange = function() {     // <--- CALLBACK
    if (xhr.readyState === 4 && xhr.status === 200) {
        console.log(xhr.responseText);    // <--- DATA HANDLING
    }
};
xhr.send();                               // <--- KEYWORD 3
```

### Type B: The Library Era (jQuery, Axios)
Libraries simplified the syntax. This is extremely common in codebases from 2010-2020.

```javascript
/* PATTERN 2: jQuery */
$.ajax({                                  // <--- KEYWORD
    url: "/api/login",
    type: "POST",
    data: { user: "admin" },
    success: function(response) {         // <--- CALLBACK
        $("#result").html(response);      
    }
});

/* PATTERN 3: Shorthand Methods */
$.get("/api/data");
$.post("/api/save");
$.getJSON("/api/list");
```

### Type C: The Modern Standard (Fetch API)
The current native standard. Uses Promises (`.then`).

```javascript
/* PATTERN 4: Fetch API */
fetch("/api/orders")                      // <--- KEYWORD
    .then(response => response.json())    // <--- PARSING
    .then(data => {                       // <--- CALLBACK
        console.log(data);
    });
```

---

## 3. Detecting the "Purpose" (Analysis Strategy)

Seeing `$.ajax` tells you **that** a call is happening. To understand **what** it is doing, you must analyze the surrounding code.

### Scenario 1: API Data Fetching (The most common)
**Goal:** Get raw data (JSON) to update a chart, table, or form.

*   **Clues:**
    *   **Parsing:** `.json()` calls.
    *   **Variable Names:** `data`, `json`, `response`, `items`.
    *   **Usage:** Loop iterating over the data.
    
    ```javascript
    fetch('/api/users')
        .then(res => res.json())           // <--- Clue: Expecting JSON
        .then(users => {
            users.forEach(u => ...);       // <--- Clue: Iterating data
        });
    ```

### Scenario 2: Dynamic HTML Injection (Server-Side Fragments)
**Goal:** Server returns a chunk of ready-made HTML code to stick into the page.

*   **Clues:**
    *   **Parsing:** `.text()` or no parsing.
    *   **DOM Manipulation:** `.innerHTML`, `.html()`, `.append()`.
    
    ```javascript
    $.get('/components/sidebar.html', function(htmlSnippet) {
        $('#sidebar').html(htmlSnippet);   // <--- Clue: Injecting HTML directly
    });
    ```

### Scenario 3: Dynamic Resource Loading (JS/CSS)
**Goal:** Lazy-load a script or style only when needed.

*   **Clues:**
    *   **Methods:** `$.getScript()`, `import()`, adding `<script>` tags.
    *   **Response Handling:** Often just `eval()` or appending to `document.head`.
    
    ```javascript
    // Clue: Specifically asking for a script file
    $.getScript("plugins/chart-library.js", function() {
        initChart();
    });
    ```

---

## 4. Master Regex Pattern List

To build a filter for "Valid AJAX", use these regex components.

| Type | Keywords to Match (Case Insensitive) | Notes |
| :--- | :--- | :--- |
| **Native** | `fetch\s*\(` | Matches modern fetch calls |
| **Native** | `new\s+XMLHttpRequest` | Matches raw XHR |
| **Native** | `\.open\s*\(` | Matches XHR setup |
| **Library** | `\$\.ajax\s*\(` | jQuery Main |
| **Library** | `\$\.(get|post|getJSON|getScript)\s*\(` | jQuery Shorthands |
| **Library** | `axios(\.(get|post|put|delete))?\s*\(` | Axios Library |
| **Library** | `superagent\.(get|post)` | Superagent Library |
| **Wrapper** | `ajax\s*:\s*function` | Custom Wrappers (Object context) |

### Advanced: Filtering by Purpose

*   **Find specific POST requests (Sensitive Actions):**
    *   Regex: `(method|type)\s*:\s*["']POST["']`
*   **Find JSON endpoints:**
    *   Regex: `\.json\(\)` OR `dataType\s*:\s*["']json["']`

---

## 5. Security Context (Why we scan this)

1.  **XSS (Cross-Site Scripting):** If an AJAX call fetches HTML/Text and injects it using `.innerHTML` (Scenario 2) without sanitization, it's a vulnerability.
2.  **CSRF (Cross-Site Request Forgery):** If `POST` requests (State changing) are made without anti-forgery tokens, attackers can trick users into performing actions.
3.  **Data Exposure:** Is the endpoint returning sensitive user data? 

---

## Summary Checklist for Analysis

When you see an AJAX block, ask:
1.  **Method:** Is it GET (Reading) or POST (Writing)?
2.  **Endpoint:** Where is it going? (`/api/...` vs `/views/...`)
3.  **Payload:** What are we sending? (Passwords? IDs?)
4.  **Response:** Are we treating it as Code (eval), HTML (innerHTML), or Data (JSON)?
