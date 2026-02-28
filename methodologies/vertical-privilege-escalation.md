# Vertical Privilege Escalation — Testing Methodology

## Objective

Vertical privilege escalation occurs when a user gains access to functionality, data, or resources intended for a higher privilege level than their own — most commonly, when a standard user gains access to administrative capabilities. The objective of testing for vertical privilege escalation is to determine whether the application consistently enforces role-based access controls across all privileged functions and endpoints, or whether lower-privileged users can access higher-privileged functionality through direct requests, parameter manipulation, or other techniques.

## Attacker Mindset

An attacker targeting vertical privilege escalation is attempting to elevate their effective role within the application without going through a legitimate privilege-granting process (e.g., without being granted an administrator role by another administrator).

The attacker reasons that many access control failures stem from one of three root causes:

1. **Obscurity rather than enforcement**: Administrative functionality exists but is simply not linked in the UI for standard users. The server does not verify the caller's role.
2. **Client-side role assertions**: The application passes role or permission information in the HTTP request (cookies, parameters, headers) and trusts it without server-side verification.
3. **Inconsistent enforcement**: Access control is correctly implemented in most places but is missing from specific endpoints, parameters, or HTTP methods.

The attacker systematically tests all three patterns.

Key questions an attacker considers:

- Are there administrative or privileged URLs accessible by direct request without a valid admin session?
- Does the application store or transmit the user's role in a way that can be modified by the user?
- Are there any parameters, headers, or request fields that indicate privilege level?
- Is there a difference in behaviour between authenticated low-privilege requests and unauthenticated requests when accessing admin endpoints?

## Step-by-Step Testing Methodology

### 1. Enumerate Privileged Functionality

Before testing as a low-privilege user, establish what privileged functionality exists:

- Browse the application as a high-privilege user (if access is available) and record all administrative URLs, endpoints, and actions.
- Review the application's JavaScript, HTML source, and API responses for references to administrative paths not linked in the UI.
- Use directory and endpoint discovery techniques to identify common administrative paths (e.g., `/admin`, `/administrator`, `/manage`, `/console`, `/dashboard/admin`).
- Review the application's `robots.txt`, sitemap, and error messages for path disclosures.

### 2. Attempt Direct Access to Privileged Endpoints

Authenticated as a standard user:

- Send direct HTTP requests to each identified privileged endpoint.
- Observe whether the server returns a 403 Forbidden, 401 Unauthorised, a redirect to a login page, or the privileged content itself.
- If a redirect is received (3xx), inspect the response body before the redirect — some implementations include the privileged content in the initial response.

### 3. Test Parameter-Based Privilege Controls

Look for any indication that the user's role or privilege is communicated via request parameters:

- Review cookies, query strings, POST body fields, and custom HTTP headers for role-related values (e.g., `role=user`, `admin=false`, `privilege_level=1`, `isAdmin=0`).
- Modify these values to assert a higher-privilege role and resend the request.
- Observe whether the application grants elevated access.

### 4. Test Token-Based Privilege Controls

If the application uses JSON Web Tokens (JWT) or similar signed tokens:

- Decode the token payload and inspect claims for role or privilege indicators.
- Attempt to modify the role claim and re-sign the token with a weak key, no signature, or the `alg: none` attack.
- Attempt to forge a token with administrative claims if the signing algorithm or key is weak.

### 5. Test for Unprotected Administrative Functionality

Some administrative interfaces are implemented with obscurity as the only access control:

- Test access to discovered administrative URLs without any modification of the session or parameters.
- Test access while completely unauthenticated.
- Test with a standard user session.
- Note that a URL that requires knowledge to discover but applies no server-side authorisation check is a vulnerability regardless of how difficult the URL is to guess.

### 6. Test HTTP Method Variations

Access control may be applied differently across HTTP methods:

- If a `POST` request to an administrative action is blocked, test the same endpoint using `GET`, `PUT`, or other methods.
- Observe whether any method variation bypasses the access control check.

### 7. Test URL and Path Variations

- Attempt case variations (e.g., `/Admin`, `/ADMIN`).
- Attempt path encoding variations (e.g., `/a%64min`).
- Attempt the addition or removal of trailing slashes.
- Attempt path prefix injection if the application uses URL-based routing rules.

### 8. Test Multi-Step Privileged Workflows Out of Sequence

Privileged workflows that span multiple steps may enforce access control only at the initial step:

- Identify multi-step administrative processes (e.g., creating a user, changing a role, approving an action).
- Attempt to access intermediate or final steps directly without completing the guarded initial step.

## Common Vulnerable Patterns

- Administrative panels accessible at predictable URLs without server-side role verification (e.g., `/admin/panel` returning content to any authenticated user).
- Role values stored in cookies or query parameters and trusted directly without cryptographic integrity protection.
- JWT tokens with weak or absent signature verification, allowing arbitrary claim modification.
- Access control enforced only through UI presentation (hidden navigation elements, disabled buttons) rather than server-side checks.
- Access control rules that are applied to the rendered page but not to the underlying API endpoints the page calls.
- Administrative functionality that is correctly protected via the primary route but accessible via an alternative path or HTTP method.

## Indicators of Vulnerability

- An HTTP 200 response containing administrative content when requested by a standard user session.
- Successful execution of a privileged action (e.g., user creation, account deletion, configuration change) from a standard user session.
- A redirect response (3xx) that embeds privileged content in the pre-redirect body.
- Different response characteristics (body, size, status code) between a standard user request and an unauthenticated request to a privileged endpoint — where both should receive identical denials.
- Application behaviour changes (e.g., new UI elements, different data returned) following the modification of a role-related parameter.

## Defensive Best Practices

- Enforce role-based access control checks server-side on every request to a privileged resource or function, verified against the server-managed session state.
- Never trust role or permission information supplied by the client in parameters, cookies, or tokens without robust server-side verification.
- If using JWTs, enforce strong signing algorithms (RS256 or HS256 with a cryptographically random key of adequate length), validate signatures on every request, and reject tokens with `alg: none`.
- Implement a centralised authorisation framework so that access control logic is applied consistently and cannot be omitted from individual endpoints.
- Apply deny-by-default: all privileged functionality should require an explicit authorisation grant, rather than relying on the absence of a denial.
- Avoid relying on URL obscurity as a security control; assume that all endpoints can be discovered.
- Include privilege escalation test cases in automated security testing pipelines, specifically verifying that standard user sessions cannot access administrative endpoints.
- Conduct regular privilege audits to ensure that role assignments and access control rules remain accurate as the application evolves.
