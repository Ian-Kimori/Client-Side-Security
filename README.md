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

---

## 1. DOM-based XSS & JavaScript Execution
**The Goal:** Prove that data from a URL source reaches a dangerous execution sink.
1.  **Find the Tap (Sink):** In DevTools **Sources**, search (`Ctrl+Shift+F`) for `.innerHTML`, `document.write(`, or `eval(`.
2.  **Trace to Intake (Source):** Look for variables that pull from `location.hash` or `location.search`.
3.  **The Test:** Append a unique string to the URL: `site.com/#ZXC123`.
4.  **Verify:** Set a **Breakpoint** on the sink. If the variable equals `"ZXC123"`, the pipe is connected.
5.  **Clean Fail:** If you change the URL to `#<img src=x onerror=alert(domain)>` and the alert pops, it is a **FAIL**.

## 2. HTML & CSS Injection
**The Goal:** Inject "dye" into the water to change the UI or steal data.
1.  **HTML Test:** Find a field that reflects input (e.g., "Welcome, [Name]"). Input `<u>Test</u>`.
    * **FAIL:** The text is underlined in the browser.
2.  **CSS Test:** Input `<style>body{background:red;}</style>`.
    * **FAIL:** The page background turns red. This proves an attacker can inject styles to "hide" real buttons or "overlay" fake ones.

## 3. Client-Side URL Redirect
**The Goal:** Prove the "Exit Pipe" can be pointed to a malicious external site.
1.  **The Source:** Look for URL parameters like `?next=` or `?redirect_to=`.
2.  **The Pipe:** Search **Sources** for `window.location.href =` or `location.replace(`.
3.  **The Test:** Change the parameter to `?next=https://google.com`.
4.  **Clean Result:** If the browser leaves the target domain and loads Google, it is a **FAIL**.

## 4. Cross-Origin Resource Sharing (CORS)
**The Goal:** Check if the "Back Door" is open for other sites to steal private data.
1.  **The Test:** Use **Burp Suite** or **curl**. Add the header: `Origin: https://evil.com`.
2.  **The Response:** Look at the headers returned by the server.
3.  **Clean Fail:** If you see `Access-Control-Allow-Origin: https://evil.com` **AND** `Access-Control-Allow-Credentials: true`, any site can steal the user's session data. **FAIL**.

## 5. Clickjacking
**The Goal:** Prove the website can be "hidden" under a fake cover to trick users.
1.  **The Test:** Create a local `.html` file on your PC:
    `<html><body><iframe src="https://target-site.com" style="opacity:0.5;"></iframe></body></html>`
2.  **Verify:** Open the file in Edge/Chrome.
3.  **Clean Result:** If you see the website inside the frame, it's a **FAIL**. If the browser blocks it (check Console for `X-Frame-Options` errors), it's a **PASS**.



## 6. Web Messaging (postMessage)
**The Goal:** Test the "Intercom" between different windows/iframes.
1.  **Find Listeners:** Search **Sources** for `window.addEventListener("message", ...)`.
2.  **The Test:** In the **Console**, type: `window.postMessage({"action":"delete"}, "*")`.
3.  **Clean Fail:** If the app performs the action without checking `event.origin`, it is a **FAIL**.

## 7. WebSockets
**The Goal:** Intercept the real-time "Radio Stream" of data.
1.  **Monitor:** Go to DevTools **Network > WS** tab.
2.  **The Test:** Perform an action and watch the messages.
3.  **Clean Fail:** If sensitive data (passwords, tokens) is sent in plain text, or if you can "Replay" a message in Burp to repeat an action, it is a **FAIL**.

## 8. Local Storage
**The Goal:** Check if sensitive "Bottled Water" is left in an unlocked cabinet.
1.  **Inspect:** Go to DevTools **Application > Local Storage**.
2.  **The Check:** Look for keys like `user_role`, `email`, or `session_id`.
3.  **Manipulation:** Manually change a value (e.g., `is_admin: "false"` to `"true"`) and refresh.
4.  **Clean Fail:** If the UI grants you new powers, the app trusts the client too much. **FAIL**.

---

### Summary Checklist for a "Clean" Audit
* **Is it a Pass?** If the "Tap" (Sink) uses `.textContent` or a library like **DOMPurify**, the water is cleaned before the user sees it.
* **Is it a Fail?** If "Water" flows from the "Intake" (URL/Storage) straight to a "Tap" (`innerHTML`/`eval`) with no filters in between.

Which "Pipe" would you like to inspect first on your target site?
