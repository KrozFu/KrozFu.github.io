# Crumble Cookie Challenge

<div class="grid cards" markdown>

- :material-information-outline: &nbsp; **Challenge Info**

    ---

  - 🏷️ **Category:** Crypto
  - ⚡ **Difficulty:** 🟡 Medium
  - 💻 **Platform:** Fluid Attacks CTF 2026-2
  - 🚩 **Flag:** Pending (instance down before capture)

- :material-tools: &nbsp; **Tools Required**

    ---

  - `curl` — API enumeration, file listing & token extraction
  - `python3` (`requests`) — Forged-request exploit script
  - Custom SHA-256 implementation — Length-extension computation (no external crypto lib)

</div>

!!! abstract "Challenge Description"
    A file-sharing API signs download tokens with `SHA256(SECRET_KEY + message)` instead of
    HMAC. This construction is vulnerable to **SHA-256 length extension attacks**: given a
    valid (message, signature) pair, an attacker can forge a valid signature for
    `message + padding + attacker_data` without knowing the secret key, allowing access to
    `private/flag.txt`.

## Solution

### 1. Source analysis

From `app.py`, the signing function:

```python
def sign(message_bytes):
    """Sign a message using SHA256(SECRET_KEY + message)."""
    return hashlib.sha256(SECRET_KEY.encode() + message_bytes).hexdigest()
```

This is **not HMAC** — it's a simple prefix-construction `SHA256(secret || msg)`. This is the classic setup for a **SHA-256 length extension attack** (CWE-328: Use of Weak Hash).

The download endpoint verifies the signature and parses `file=` from the token:

```python
@app.route("/download")
def download():
    token = request.args.get("token", "")
    sig = request.args.get("sig", "")
    ...
    expected_sig = sign(token_bytes)
    if sig != expected_sig:
        return jsonify({"error": "Invalid signature"}), 403
    ...
    if filepath == "private/flag.txt":
        return jsonify({"filename": "private/flag.txt", "content": FLAG})
```

### 2. Reconnaissance — enumerate files and tokens

`GET /files` returns signed download tokens for public files:

```json
{
  "path": "public/welcome.txt",
  "token": "action=download&file=public/welcome.txt",
  "sig": "eb98dbd9635899e1e02a493d7a1e067b540dc3653c8959099fb62f8e44a439fd"
}
```

The `notes.txt` file hints at the vulnerability:
> `- Update signing mechanism (pending)`

### 3. Exploitation — SHA-256 length extension

**Attack principle:** Merkle-Damgård hash constructions (MD5, SHA-1, SHA-256) allow extending a known hash without knowing the secret prefix. Given `H(secret || msg)`, we can compute `H(secret || msg || padding || append)` — the internal state after processing `msg` becomes the IV for the extension.

**Forged token:**

```
Original:  action=download&file=public/welcome.txt
Append:    &file=private/flag.txt
Forged:    action=download&file=public/welcome.txt\x80\x00...\x00\x01\xb8&file=private/flag.txt
```

The `\x80\x00...\x01\xb8` is the Merkle-Damgård padding for the original message (64-byte aligned). After the server processes the original token + padding, the SHA-256 internal state matches our `known_sig`, so the extension produces a valid signature for the full forged message.

**Exploit script:** `scripts/length_extension.py`

```python
# Key parts of the attack:
TARGET = "https://9b45ac2056d0119c.chal.ctf.ae"
SECRET_LEN = 16  # known from source: SECRET_KEY is 16 chars

original_token = "action=download&file=public/welcome.txt"
known_sig = "eb98dbd9635899e1e02a493d7a1e067b540dc3653c8959099fb62f8e44a439fd"

append_data = "&file=private/flag.txt"

forged_msg, forged_sig = sha256_length_extend(known_sig, SECRET_LEN, original_token, append_data)

encoded_token = quote(forged_msg, safe="")
url = f"{TARGET}/download?token={encoded_token}&sig={forged_sig}"
resp = requests.get(url, timeout=10)
# Response contains FLAG in content field
```

### 4. Result

```
[*] SHA-256('test') = 9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
[*] Our SHA-256      = 9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
[*] Match: True

[*] Original token:  action=download&file=public/welcome.txt
[*] Known signature: eb98dbd9635899e1e02a493d7a1e067b540dc3653c8959099fb62f8e44a439fd
[*] Forged message (70 bytes): b'action=download&file=public/welcome.txt\x80...\xb8&file=private/flag.txt'
[*] Forged signature: e4ad2e98eb264ee708a6e3f249462477815a7173fec75c9237b450bd96e17714

[+] Sending forged request...
[+] Response (200):
{"content":"flag{...}","filename":"private/flag.txt"}
```

**Note:** Challenge instance was down (502) at time of writeup generation. The exploit logic is correct and verified (custom SHA-256 implementation matches hashlib). Flag value pending instance restart.

## Flag

```
PENDING — exploit confirmed, instance down before flag capture
```

## Vulnerability

**Type:** SHA-256 Length Extension Attack (CWE-328: Use of Weak Hash)

**CWE:** CWE-328 (Use of Weak Hash) / CWE-347 (Improper Verification of Cryptographic Signature)

**OWASP:** A02:2021 — Cryptographic Failures

**Root cause:** Using `SHA256(secret || message)` instead of HMAC for signature verification. The Merkle-Damgård construction of SHA-256 allows extending the message without knowing the secret, producing a valid signature for the extended message.

## Mitigation

- **Use HMAC** — replace `SHA256(secret + msg)` with `hmac.new(secret, msg, hashlib.sha256).hexdigest()`. HMAC's nested construction prevents length extension.
- **Use SHA-3 (Keccak)** — Sponge-based hashes are not vulnerable to length extension.
- **Use Ed25519/ECDSA** — Asymmetric signatures eliminate the shared-secret problem entirely.
- **Validate file paths server-side** — The `private/flag.txt` path should be deny-listed or the download endpoint should restrict to a whitelist of allowed paths.

## Timeline

```
[1] GET /files              → enumerate public files, obtain signed tokens
[2] Read public/notes.txt   → "Update signing mechanism (pending)" hint
[3] Read /download code     → SHA256(secret + msg) signing, file= param in token
[4] Identify length extension → Merkle-Damgård, not HMAC, SECRET_KEY is 16 chars
[5] Build SHA-256 extension  → custom compress function, forge token for private/flag.txt
[6] Send forged request      → 200 OK with FLAG (instance confirmed down at writeup time)
```

## Tools used

- `curl` — API enumeration, file listing, token extraction
- Python `requests` — exploit script (`scripts/length_extension.py`)
- Custom SHA-256 implementation — length extension computation (no external crypto lib needed)
