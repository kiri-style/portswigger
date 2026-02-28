# Insecure Direct Object Reference (IDOR) — Testing Methodology

## Objective

Insecure Direct Object Reference (IDOR) is a specific manifestation of broken access control in which an application exposes a direct reference to an internal object — such as a database record, file, or account — in a way that an attacker can manipulate to access or modify resources they are not authorised to interact with. The objective of testing for IDOR is to determine whether the application validates authorisation at the object level for every request, or whether any user can reference and retrieve resources belonging to others by modifying an identifier.

## Attacker Mindset

An attacker targeting IDOR vulnerabilities focuses on the gap between *identification* and *authorisation*. Identifying a resource (e.g., knowing its ID) is not equivalent to being authorised to access it, but applications that implement IDOR vulnerabilities treat these as the same condition.

The attacker looks for any user-supplied value that references a server-side object and tests whether the server verifies ownership or permission for that reference. The attacker treats every parameter containing a numeric, sequential, or otherwise predictable identifier as a potential attack surface.

Key questions an attacker considers:

- Does the application use predictable, sequential, or guessable identifiers?
- Is the server enforcing that the requesting user owns or is permitted to access the referenced object?
- Are object references present in URL parameters, request bodies, headers, or cookies?
- Can identifiers be enumerated or inferred from patterns observed during normal use?

## Step-by-Step Testing Methodology

### 1. Identify Direct Object References

During authenticated browsing, catalogue all locations where identifiers appear:

- URL path segments (e.g., `/users/1042/profile`)
- Query string parameters (e.g., `?order_id=5581&format=pdf`)
- Request body fields (e.g., `{"account_id": 99}`)
- HTTP headers (e.g., custom headers such as `X-User-Id`)
- File references (e.g., `?filename=statement_march.pdf`)

Note the format and apparent type of each identifier (numeric, alphanumeric, UUID, filename).

### 2. Establish a Baseline

For each identified reference, record:

- The response when accessing an object that belongs to the authenticated user.
- The response format, status code, and content length.
- Any error messages returned when an object does not exist.

This baseline enables comparison against responses when testing unauthorised references.

### 3. Test Horizontal IDOR

Attempt to access objects belonging to other users of the same privilege level:

- Increment or decrement numeric IDs by one or a small amount.
- For UUIDs or opaque identifiers, obtain a second test account and use its identifiers.
- Replace the authenticated user's identifier with that of another user.
- Confirm that any successful response contains data belonging to the other user rather than the original.

### 4. Test Vertical IDOR

Attempt to access objects that belong to administrative or higher-privileged accounts:

- If administrative accounts are known to exist (e.g., `admin`, `superuser`), attempt to reference objects owned by those accounts.
- Look for identifiers that may correspond to system-level resources (e.g., low numeric IDs such as 1 or 2 often correspond to administrative accounts in sequential schemes).

### 5. Test Indirect References

Applications sometimes use indirect references (e.g., `ref=3`) that are mapped server-side to actual object IDs. Test whether:

- The mapping itself can be manipulated (e.g., by supplying a reference not assigned to the authenticated user).
- The application exposes the real identifier elsewhere (e.g., in JavaScript, API responses, or HTML source).

### 6. Test IDOR in Non-Retrieval Operations

IDOR is not limited to read operations. Test whether unauthorised access is possible through:

- Update operations (e.g., modifying another user's profile by supplying their ID in a `PUT` or `POST` request)
- Delete operations (e.g., deleting another user's resource)
- State-changing operations (e.g., confirming or cancelling another user's order)

### 7. Test Static File References

If the application serves user-specific files via direct URLs:

- Identify the URL pattern for files belonging to the authenticated user.
- Attempt to access files belonging to other users by modifying path segments or filenames.
- Check whether the server enforces session-based authorisation on file delivery or relies solely on path obscurity.

### 8. Examine Encoded or Hashed Identifiers

Some applications obscure identifiers using Base64 encoding, hashing, or custom schemes:

- Decode any apparent encoding (Base64, URL encoding, hexadecimal).
- Identify the underlying format of the decoded value.
- Re-encode modified values and test whether the application accepts them.

## Common Vulnerable Patterns

- Sequential numeric primary keys used as direct references without ownership checks (e.g., `/invoice/1001`, `/invoice/1002`).
- UUIDs or GUIDs used in the belief that unguessability is sufficient access control.
- File downloads served via direct URL with the filename or path as a parameter.
- API endpoints that accept object IDs from the request body without validating the requesting user's relationship to that object.
- Password reset or account management flows that reference accounts by ID rather than by session context.
- Export or report generation endpoints that reference another user's data by ID.

## Indicators of Vulnerability

- A successful HTTP response (200, 201, or similar) to a request referencing another user's object identifier.
- Response content that contains personally identifiable or account-specific data belonging to a different user.
- Successful modification or deletion of a resource owned by another user, confirmed by subsequent retrieval.
- Consistent response size differences between accessing a valid object and a non-existent object, enabling enumeration of valid identifiers.
- API error messages that confirm the existence of an object (e.g., "Access denied" vs. "Not found") when unauthorised access is attempted.

## Defensive Best Practices

- Perform object-level authorisation checks on every request that references a resource, verifying that the authenticated session is permitted to access the specific object.
- Do not rely on identifier unguessability (UUIDs, hashes) as a substitute for authorisation.
- Prefer indirect reference maps: generate session-specific tokens that map to internal object IDs server-side, so that raw database IDs are never exposed to the client.
- Implement a consistent authorisation layer that can be applied uniformly across all resource types and operations.
- Log and monitor for patterns consistent with identifier enumeration (e.g., sequential ID sweeping across a short time window).
- Apply the principle of least privilege: users should only be able to reference objects within their permitted scope.
- Include IDOR test cases in automated regression testing, using multiple test accounts to verify cross-user access is denied.
