NAIJAPAY SECURE CODE REVIEW
CodeAlpha Cybersecurity Internship, Task 3
Reviewed by: Paul Momoh

SOURCE CODE REVIEWED

This review was performed on NaijaPay, part of the vulnverse collection of practice applications, licensed under MIT.

Original repository: https://github.com/w1t3ack3r/vulnverse


WHY I PICKED THIS PROJECT

I wanted something small enough to get through in a few days, but realistic enough to have genuine vulnerabilities worth finding. NaijaPay fit that well. It's a small PHP-based fintech app that handles logins, registration, and money transfers between users. Because it deals with real money and sensitive data, particularly BVNs, any security flaws could have serious consequences, making it a good learning ground.

I didn't review every file in the repo. I focused on the ones most likely to matter: authentication, registration, and the money transfer logic. Those are where real vulnerabilities tend to live in an app like this, and where I could explain the risk clearly instead of listing things for the sake of a longer report.


WHAT I FOUND

1. Users can move money out of accounts that aren't theirs (Critical)
File: portal/do_transfer.php

This is the most serious issue I found. When a user submits a transfer, the form sends along an account ID saying which account the money should come from. The backend never checks whether that account actually belongs to the person who's logged in. It just trusts the number it's given and pulls money from it.

In practice, this means a logged-in user could tamper with the form (something as simple as editing the value in their browser before submitting) and drain money from any account in the system, not their own account specifically. For a payment app, this is about as serious as a bug gets. It's a direct path to financial loss for real users.

The fix: before processing any transfer, the server needs to check that the from_account_id actually belongs to the logged-in user's session. If it doesn't match, reject the request immediately.


2. Negative transfer amounts break the math (High)
File: portal/do_transfer.php

The transfer amount comes from user input, and there's no check that it's a positive number. If someone entered a negative amount, the balance calculation runs backwards. The sender would end up gaining money instead of losing it, and the recipient would lose money instead of gaining it. Combined with the account ownership issue above, this makes the transfer system exploitable in more than one way.

The fix: validate that the amount is greater than zero before running any balance calculations.


3. User input can execute commands on the server (Critical)
File: portal/resend_receipt.php

This one's the most technical finding, so here's the plain version. This file uses a template engine (Twig) to build the receipt email, and it was set up so that whatever text a user types into the email field gets treated as instructions the server should run, instead of plain text to store. The code also had two functions added that let those instructions execute system-level commands.

In plain terms, a user filling out this form could get the server itself to run commands. That's a much bigger problem than losing data; it means someone could take control of parts of the server.

Reading through the code, this looks like it happened because someone was troubleshooting a template filter issue and added a quick workaround without realizing what it exposed. It's a good reminder that convenient shortcuts in code can carry a real cost when you don't fully understand what they open up.

The fix: raw user input should never be treated as a template to be executed. User input should be inserted as plain data into an existing, trusted template. The custom exec and shell functions added to Twig should be removed entirely; there's no safe reason for user-facing code to have that kind of access.


4. BVNs are stored as plain text (High)
File: register.php

Passwords in this app are properly hashed before being saved, but BVNs (Bank Verification Numbers) go into the database completely unprotected. A BVN is one of the most sensitive pieces of personal financial ID in Nigeria. Unlike a password, a user can't reset it if it leaks. If this database were ever breached, every user's BVN would be exposed in readable form.

The fix: BVNs should be encrypted before storage, not stored in plain text. Given how sensitive this data is, it deserves the same level of care as password handling, if not more.


5. Session isn't refreshed after login (Medium)
File: login.php

After a successful login, the app doesn't regenerate the session ID. This matters because if an attacker already knows a user's session ID from before they logged in (there are ways this can happen), that same session stays valid after login too, potentially letting the attacker ride along on the now-authenticated session.

The fix: regenerate the session ID immediately after confirming a successful login.


6. A cookie is set without the HttpOnly flag, and it's hardcoded (Medium)
File: login.php

There's a cookie being set on every login that's missing the HttpOnly flag, which means it's readable by any JavaScript running on the page. That's a real problem if the site is ever vulnerable to script injection elsewhere. The cookie's value is also the exact same hardcoded string for every user, so it isn't functioning as a real per-user identifier.

The fix: any cookie handling session-related data should have HttpOnly set, and if this cookie is meant to track something per-user, it needs to be a unique, randomly generated value.


7. Raw error messages are shown to users (Low)
Files: register.php, db_connect.php

When something goes wrong on the backend, the actual technical error message gets shown directly to the user instead of a generic message. This can leak details about how the database or app is structured, information that's genuinely useful to an attacker planning a more targeted attack.

The fix: log the real error somewhere private (a log file, not the screen) and show the user a simple, generic message like "Something went wrong, please try again."


WHAT WAS ACTUALLY DONE WELL

It's worth being fair here. This codebase handles several things properly:

- Every database query I reviewed uses prepared statements, which protects against SQL injection consistently throughout the app
- Passwords are hashed correctly and checked properly at login
- The transfer logic uses row locking and database transactions, which correctly prevents two simultaneous transfers from corrupting balances. That's a fairly advanced concept for a beginner-built app to get right
- Output in at least one place is properly escaped, protecting against stored XSS
- Database credentials are pulled from environment variables rather than hardcoded into the codebase


RECOMMENDATIONS AND BEST PRACTICES

Never trust data that comes from the user, including form values, cookies, and IDs. Always verify it server-side before acting on it, even if the frontend already validated it.

Treat user input as data, never as code. Anything that gets parsed, executed, or rendered as a template should never contain raw, unmodified user input.

Store sensitive data encrypted, not just passwords. Anything tied to real-world identity or finance (BVNs, ID numbers, card details) deserves the same protection as login credentials.

Keep error messages generic on the user-facing side. Detailed errors are useful for developers, but they belong in private logs, not on screen.

Regenerate session IDs after login, and set proper cookie flags on anything session-related.

Review third-party library usage carefully, especially when adding custom functions or filters. A small convenience shortcut can quietly expose serious functionality if it isn't fully understood.


CLOSING THOUGHTS

Going through this code, a lot of the fundamentals (hashing, prepared statements, transactions) are genuinely solid. The issues I found look like gaps that happen when you're moving fast, or reaching for a convenient shortcut without fully thinking through what it opens up. That's a common, human way security problems actually happen: someone moving quickly under pressure, not someone careless.

The biggest takeaway for me personally is that the account ownership issue in the transfer logic and the template injection issue are both examples of the same root problem: trusting input without verifying it first. That single idea, "don't trust anything until you've checked it," is the thread running through almost every finding in this review.
