# Broken Access Control — Testing Methodology

## Objective

Broken Access Control occurs when an application fails to enforce restrictions on what authenticated users are permitted to do or access. The objective of testing for this vulnerability class is to determine whether a user can act outside of their intended permissions — accessing resources, performing actions, or viewing data that should be restricted based on their role or identity.

## Attacker Mindset

An attacker targeting access control weaknesses attempts to answer one core question: *Can I access or perform actions that I should not be permitted to?*

This involves reasoning about the difference between what the application presents to a user and what the underlying server actually enforces. Attackers assume that client-side restrictions (hidden UI elements, disabled buttons, redirects) are cosmetic and that server-side enforcement is the only meaningful boundary. They systematically probe every endpoint and parameter to test whether the server validates authorisation independently of the client state.

Key questions an attacker considers:

- Is access control enforced on every request, or only at the point of login?
- Does the application trust user-supplied identifiers (IDs, roles, filenames)?
- Are there administrative or privileged endpoints that are obscured but not protected?
- Can parameter manipulation grant access to resources belonging to another user?

## Step-by-Step Testing Methodology

### 1. Map the Application's Access Model

Before testing, establish a baseline understanding of the intended access control structure:

- Identify all user roles (e.g., anonymous, standard user, administrator).
- Document which resources and actions each role should legitimately access.
- Note any role-specific URLs, endpoints, or parameters observed during normal browsing.

### 2. Capture and Catalogue Requests

Using an intercepting proxy (e.g., Burp Suite):

- Browse the application as each available role and record all HTTP requests.
- Pay particular attention to requests that reference identifiers (numeric IDs, GUIDs, usernames, filenames) or contain role or permission indicators.
- Note any requests that are made in the background (AJAX, API calls) that may not be visible in the rendered UI.

### 3. Test Horizontal Access Control

Horizontal access control governs access between users of the same privilege level:

- Identify requests that reference a resource owned by the authenticated user (e.g., `/account?id=42`).
- Replace the identifier with one belonging to a different user.
- Send the modified request and observe whether the server returns the other user's data.
- Repeat for all object identifiers found across the application.

### 4. Test Vertical Access Control

Vertical access control governs access between users of different privilege levels:

- Identify URLs, endpoints, and functions available to higher-privileged users (e.g., `/admin`, `/manage/users`, `/config`).
- Attempt to access these endpoints directly while authenticated as a lower-privileged user.
- Observe whether the server denies access, redirects, or returns the privileged resource.

### 5. Test Parameter-Based Access Control

Some applications implement access control through parameters rather than server-side session state:

- Look for parameters such as `role=user`, `admin=false`, `isAdmin=0`, or similar.
- Modify these values and resend the request.
- Observe whether the application grants elevated privileges.

### 6. Test for Method-Based Bypasses

Access control may be implemented for specific HTTP methods but not others:

- If a `GET` request to a privileged endpoint is blocked, attempt the same request using `POST`, `PUT`, `HEAD`, or custom methods.
- Observe whether different HTTP methods bypass the restriction.

### 7. Test URL and Path Variations

Access control rules may not account for all URL representations:

- Try path traversal variations (e.g., `/admin/../admin`, `/ADMIN`, `/admin/`).
- Try adding unexpected path prefixes or suffixes.
- Observe whether any variation circumvents the control.

### 8. Test State-Dependent Access

Some access control decisions depend on application state:

- Attempt to access multi-step workflows out of sequence (e.g., skipping directly to a final confirmation step).
- Attempt to reuse or replay previously valid requests after a role change or logout.

## Common Vulnerable Patterns

- Direct object references in URL parameters or request bodies without server-side authorisation checks (e.g., `/invoice?id=1001`).
- Access control enforced only in the front end (hidden menus, disabled UI elements) with no corresponding server-side validation.
- Administrative functionality accessible via unlinked but unprotected URLs.
- Role or permission values stored in user-controlled parameters, cookies, or JWT claims without integrity verification.
- Inconsistent enforcement across different representations of the same endpoint (e.g., enforced on `/admin` but not `/Admin`).
- Missing access checks on API endpoints that are assumed to be internal.

## Indicators of Vulnerability

- An HTTP 200 response to a request for a resource belonging to another user.
- Privileged page content returned to a lower-privileged user, even if accompanied by a redirect (check 3xx responses for embedded content).
- Different response sizes or bodies when replaying a privileged request as an unprivileged user, compared to a fully denied request.
- Application-level error messages revealing the existence of restricted resources.
- Successful form submissions or state changes resulting from replayed administrative requests.

## Defensive Best Practices

- Enforce access control decisions server-side on every request, independent of client state or UI presentation.
- Implement a centralised, reusable authorisation mechanism rather than dispersed ad hoc checks.
- Deny access by default; explicitly grant permissions rather than explicitly denying them.
- Avoid exposing direct object references to users where possible; use indirect references mapped server-side.
- Validate that the authenticated session's identity is authorised to access every requested object.
- Log and alert on repeated access control failures as a potential indicator of enumeration or attack.
- Conduct regular access control audits, including automated testing with tools configured to test cross-user and cross-role access.
- Apply the principle of least privilege to all user roles and service accounts.
