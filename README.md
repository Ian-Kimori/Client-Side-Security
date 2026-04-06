# Client-Side-Security

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

## (i). Finding the "Sink" (The Tap)
You first need to find where the "water" is coming out. In DevTools:
1.  Press **F12** and go to the **Sources** tab.
2.  Press **Ctrl + Shift + F** (this opens a search bar at the bottom).
3.  Search for e.g `.innerHTML`. 
4.  If you see a result like e.g `document.getElementById('error').innerHTML = expando;`, click it. That is your **Sink**.

---

## (ii). Tracing to the "Source" (The Intake)
Now you need to know where the variable like e.g `expando` came from.

1.  Press Ctrl + Shift + F (Global Search).

2.  In the search box that appears at the bottom, type: `var expando = ` or just `expando =`.

3.  Look through the results. You are looking for the line where expando is assigned a value for the first time.

---

## Analyze the Assignment

Once you find that line, look at what is on the right side of the = sign. This tells you if the test is a Pass or a Fail.

### Scenario A (Pass): expando = "jQuery" + Math.random();

Background: The data comes from a random generator inside the browser. An attacker cannot "inject" anything into a random number.

Result: CLEAN PASS. Secure.

### Scenario B (Fail): expando = location.hash; or expando = window.name;

Background: The data comes from a "Source" that an attacker can change.

Result: FAIL. Vulnerable to DOM XSS.

---

## (iii). Use the "Call Stack" (The Time Machine) (The Pipe)
If the variable is passed through many different functions, tracing it manually is hard. Use the Call Stack instead.

1.  **Set a Breakpoint:** Click the line number of the sink to set a Breakpoint (a blue/purple arrow).

2.  **Trigger the Code:** Refresh the page or trigger the action. The browser will "freeze" on that line.

3.  On the right side of the screen, look for the **Scope** section. It lists every variable active right now. Find `expando`. What is its value?

4.  Look at the "Call Stack" pane on the right. It shows the chain of functions that led to this moment.

5.  Click the function below the current one in the list. The editor will jump to where the data was sent from.

6.  Repeat this until you reach a function that grabs data directly from the URL or Storage.

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
## c. Visualizing the Pipe in your Code "Call Stack"

Think of the Pipe as everything that happens between the moment the data is born (the Source) and the moment it hits the screen (the Sink).

When you use the Call Stack, you are literally walking through the Pipe. Each "layer" in that stack is a different section of the pipe where the data might be changed, moved, or cleaned.

---

## The "Water System" Analogy
* **The Source (The Intake):** Where an attacker puts "dirty water" into the system (e.g., URL Bar, Local Storage, or Cookies).
* **The Code "Call Stack" (The Pipe):** The JavaScript logic. If the pipe has a **Filter** (Sanitization), the water stays clean.
* **The Sink (The Tap):** Where the data is finally executed or displayed. If dirty water reaches a "Tap" like `.innerHTML`, it "splashes" the user with a script.

To conduct a "clean" end-to-end audit, we use the **Plumber’s Strategy**. You are looking for a connection between the **Intake** (Source) and the **Tap** (Sink). 

A **Pass** occurs when you find a **Filter** in the **Pipe** (the code between the source and sink) or if the **Source** itself is non-user-controlled (like your `new Date()` example).

---

### 1. DOM XSS / JS Execution / HTML Injection
* **The Tap (Sink):** Search (`Ctrl+Shift+F`) for `.innerHTML`, `document.write`, or `eval()`.
* **The Pipe (Trace):** Set a **Breakpoint** on the Sink. When it hits, look at the **Call Stack**. Click the functions below the top one to move "upstream."
* **The Intake (Source):** Look for `location.hash`, `location.search`, or `window.name`.
* **The Test:** Enter `site.com/#<img src=x onerror=alert(1)>`.
* **✅ STOP (PASS):** If you see a function in the Call Stack that uses `.textContent` instead of `.innerHTML`, or a library like `DOMPurify.sanitize()`.
* **❌ FAIL:** If the "dirty water" from the URL reaches the Sink without any changes.



---

### 2. Client-Side URL Redirect
* **The Tap:** Search for `location.href =` or `location.replace()`.
* **The Intake:** Look for parameters like `?next=` or `?url=`.
* **The Test:** Change the URL to `?next=https://google.com`.
* **✅ STOP (PASS):** If the code checks the destination against a whitelist (e.g., `if (url.startsWith("/dashboard"))`).
* **❌ FAIL:** If the browser automatically redirects you to Google.

---

### 3. CSS Injection
* **The Tap:** Search for `.style.setProperty` or code creating `<style>` tags.
* **The Intake:** Any reflected input (like a username or theme color).
* **The Test:** Inject `<style>body{display:none;}</style>`.
* **✅ STOP (PASS):** If the `<` and `>` characters are encoded to `&lt;` and `&gt;` in the **Elements** tab.
* **❌ FAIL:** If the website content disappears because your CSS was executed.

---

### 4. Client-Side Resource Manipulation
* **The Tap:** Search for `script.src =` or `script.text =` or `iframe.src =`.
* **The Intake:** A URL parameter that defines a path (e.g., `?lang=en`).
* **The Test:** Try `?lang=../../attacker.com/evil`.
* **✅ STOP (PASS):** If the path is hardcoded or "mapped" (e.g., `var path = config[lang]`).
* **❌ FAIL:** If the **Network** tab shows the browser trying to load a script from the attacker's domain.

---

### 5. CORS (Cross-Origin Resource Sharing)
* **The Test:** In Burp Suite, add the header `Origin: https://evil.com` to your request.
* **✅ STOP (PASS):** If the response does **not** contain `Access-Control-Allow-Origin` or if it is fixed to a specific trusted domain.
* **❌ FAIL:** If you see `Access-Control-Allow-Origin: https://evil.com` AND `Access-Control-Allow-Credentials: true`.

---

### 6. Clickjacking
* **The Test:** Create a local `.html` file: `<iframe src="https://yoursite.com"></iframe>`.
* **✅ STOP (PASS):** If the frame is blank and the **Console** says "Refused to display... X-Frame-Options: SAMEORIGIN."
* **❌ FAIL:** If the website loads perfectly inside your local iframe.



---

### 7. Web Messaging (postMessage)
* **The Tap:** Search for `window.addEventListener("message", ...)` or `window.addEventListener("hashchange", ...)` or `window.addEventListener("popstate", ...)`.
* **The Pipe:** Look for an origin check: `if (event.origin !== "https://trusted.com") return;`.
* **✅ STOP (PASS):** If that origin check exists and is strict.
* **❌ FAIL:** If there is no origin check or if it uses `*` (wildcard).

---

### 8. WebSockets
* **The Intake:** Go to **Network > WS**. Click the connection and look at **Messages**.
* **✅ STOP (PASS):** If messages contain a unique `token` or `nonce` that prevents attackers from spoofing messages.
* **❌ FAIL:** If you can "Replay" a message in Burp (like `{"action": "transfer_funds"}`) and the server accepts it without a fresh token.

---

### 9. Local Storage
* **The Intake:** Go to **Application > Local Storage**.
* **✅ STOP (PASS):** If you see no sensitive data (passwords, PII) or if the data is encrypted.
* **❌ FAIL:** If you find a key like `is_admin: false`, change it to `true`, refresh, and get admin access.



---

### Summary: Where to Stop?
1.  **Stop at the Source:** If the variable is hardcoded or uses a safe generator (like `new Date()`), the test is a **PASS**.
2.  **Stop in the Pipe:** If you see a "Filter" function (like `DOMPurify` or `textContent`) in the Call Stack, the test is a **PASS**.
3.  **Stop at the Sink:** If you reach the Sink and the data is still "dirty" (matches your URL input), the test is a **FAIL**.

Which specific "Pipe" are you looking at right now? Give me the code snippet and I'll tell you if you can stop.
