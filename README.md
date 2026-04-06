# Client-Side-Security

To test client-side security "cleanly," you have to act like a plumber inspecting a building's water system. You aren't just looking for a leak; you are tracing where the water enters, where it flows, and where it eventually comes out of the tap.

---

## The "Water System" Analogy
* **The Source (The Intake):** This is where the user (or attacker) puts "water" into your pipes. (e.g., the URL Bar, Cookies, or Local Storage).
* **The Code (The Pipe):** The JavaScript functions that move that data around. If the pipe has a "Filter" (Sanitization), the water stays clean.
* **The Sink (The Tap):** Where the data is finally used or displayed. If dirty water reaches a "Tap" like `.innerHTML`, it splashes the user with a virus (XSS).

Think of it like a **water pipe**:
1.  **The Source (The Intake):** Where the user (attacker) puts data into the pipe (e.g., the URL).
2.  **The Code (The Pipe):** The JavaScript functions that move the data around.
3.  **The Sink (The Tap):** Where the data comes out and is "executed" by the browser.

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
2.  In the search box that appears at the bottom, type: var expando = or just expando =.
3.  Look through the results. You are looking for the line where expando is assigned a value for the first time.

Analyze the Assignment

Once you find that line, look at what is on the right side of the = sign. This tells you if the test is a Pass or a Fail.

### Scenario A (Pass): expando = "jQuery" + Math.random();

Background: The data comes from a random generator inside the browser. An attacker cannot "inject" anything into a random number.

Result: CLEAN PASS. Secure.

### Scenario B (Fail): expando = location.hash; or expando = window.name;

Background: The data comes from a "Source" that an attacker can change.

Result: FAIL. Vulnerable to DOM XSS.

or

1.  **Set a Breakpoint:** Click the line number next to that `.innerHTML` code. A blue arrow appears.
2.  **Trigger the Code:** Refresh the page. The website will "freeze," and that line will turn highlighed.
3.  **Look at the "Scope" Pane:** On the right side of the screen, look for the **Scope** section. It lists every variable active right now. Find `expando`. What is its value?
4.  **Look at the "Call Stack":** Right below Scope is the **Call Stack**. It shows you the function that ran *just before* this one. Click the name below the top one.

---

Does your code look like one giant, long line of text? If so, clicking that **`{ }`** button is the first step to making sense of it. What do you see when you search for `location.hash`?

To understand client-side security, you have to think like a data-tracker. In the browser, "Source-to-Sink" is the path that data travels. If that path isn't "cleaned" (sanitized or encoded) along the way, the application is vulnerable.

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

---

## 1. DOM-based XSS & JS Execution
**The Goal:** Prove that "dirty water" from the URL reaches an execution "tap" like `eval()` or `.innerHTML`.
1.  **Source:** Add a "Honeypot" to your URL: `site.com/#ZXC123`.
2.  **Trace:** In DevTools **Sources**, search (`Ctrl+Shift+F`) for `.innerHTML` or `eval(`.
3.  **The Tap:** Set a **Breakpoint** on that line. Refresh the page.
4.  **Verify:** Look at the **Scope** window. If the variable equals `"ZXC123"`, the pipe is connected.
5.  **Clean Fail:** If you can change the URL to `#<img src=x onerror=alert(1)>` and an alert pops, it's a **FAIL**.

## 2. HTML & CSS Injection
**The Goal:** Inject "dye" into the water to change how the room looks (defacement or data theft).
1.  **Source:** Any input field that reflects on the page (e.g., "Hello, [Name]").
2.  **HTML Test:** Input `<h1>Hacked</h1>`. If the text becomes huge, the "Tap" isn't filtering tags.
3.  **CSS Test:** Input `<style>body{background:red;}</style>`.
4.  **Clean Result:** If the page turns red, the "Tap" is vulnerable to CSS Injection. It’s a **FAIL**.

## 3. Client-Side URL Redirect
**The Goal:** Prove the "Exit Pipe" can be pointed to a dangerous location.
1.  **Source:** Look for URL parameters like `?next=` or `?redirect=`.
2.  **The Test:** Change it to `?next=https://google.com`.
3.  **The Tap:** In **Sources**, search for `window.location.href =`.
4.  **Clean Result:** If the browser leaves the site and goes to Google, it's a **FAIL**.

## 4. Cross-Origin Resource Sharing (CORS)
**The Goal:** Check if the "Back Door" is left open for other websites to steal water.
1.  **The Test:** Use **Burp Suite** or **curl**. Add the header: `Origin: https://evil.com`.
2.  **The Check:** Look at the response headers.
3.  **Clean Fail:** If you see `Access-Control-Allow-Origin: https://evil.com` AND `Access-Control-Allow-Credentials: true`, any site can steal the user's data. **FAIL**.

## 5. Clickjacking
**The Goal:** Prove your "Water Meter" can be hidden under a fake cover.
1.  **The Test:** Create a local `.html` file on your PC:
    `<html><body><iframe src="https://target-site.com" style="opacity:0.5;"></iframe></body></html>`
2.  **The Check:** Open it in a browser.
3.  **Clean Result:** If you see the website inside the frame, it's a **FAIL**. If the browser blocks it, it's a **PASS**.



## 6. WebSockets & Web Messaging
**The Goal:** Test the "Intercom System" between different parts of the building.
1.  **Web Messaging:** Search **Sources** for `window.addEventListener("message", ...)`.
2.  **The Test:** In the **Console**, type: `window.postMessage("HACK", "*")`.
3.  **Clean Fail:** If the app processes "HACK" without checking who sent it (`event.origin`), it's a **FAIL**.
4.  **WebSockets:** In DevTools **Network > WS**, watch the messages. If you can "Replay" a message in Burp to perform an action, it's a **FAIL**.

## 7. Local Storage
**The Goal:** Check if sensitive "Bottled Water" is left in an unlocked cabinet.
1.  **The Test:** Go to DevTools **Application > Local Storage**.
2.  **The Check:** Look for keys like `role: "user"` or `session_token`.
3.  **Manipulation:** Manually change `role` to `"admin"` and refresh.
4.  **Clean Fail:** If you suddenly see "Admin" buttons, the app trusts the user too much. **FAIL**.



---

### Summary Checklist for a "Clean" Audit
* **Is it a Pass?** If the "Tap" (Sink) uses `.textContent` or a sanitizer like **DOMPurify**, the water is cleaned before it touches the user.
* **Is it a Fail?** If the "Water" goes from the "Intake" (URL/Storage) straight to the "Tap" (`innerHTML`/`eval`) with no filters in between.

Which "Pipe" would you like to inspect next?
