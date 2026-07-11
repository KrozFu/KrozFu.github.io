# Deadbolt Challenge

<div class="grid cards" markdown>

- :material-information-outline: &nbsp; **Challenge Info**

    ---

  - 🏷️ **Category:** Mobile
  - 💻 **Platform:** Fluid Attacks CTF 2026-2
  - 🎯 **Target:** [chal.ctf.ae](https://a3b7c610a2ef0874.chal.ctf.ae/)
  - 🚩 **Flag:** `flag{fe4a8cf9724233bb}`

- :material-tools: &nbsp; **Tools Required**

    ---

  - APK decompiler — Recover `CryptoManager` KDF parameters from the Java sources
  - `python3` (`hashlib.pbkdf2_hmac`, `cryptography`) — Re-derive the key & decrypt the vault backup
  - `curl` — Submit the recovered admin password

</div>

!!! abstract "Challenge Description"
    FluidVault is a mobile password manager that protects its vault with biometrics and
    AES-256-GCM encryption. An exported vault backup (`vault_backup.enc`) and the APK's
    decompiled Java sources are provided. The AES key is derived with a weak PBKDF2 (10k
    iterations, hardcoded salt) over a low-entropy device identifier that falls back to a
    fixed string — making the key fully deterministic and the backup decryptable offline.

## Recon

- **Server endpoints** (from `VaultApiClient.java`):
  - `GET /api/vault/info` — returns KDF and encryption parameters
  - `POST /api/vault/admin` — accepts `{"master_password": "..."}` and
    returns the flag when the correct password is given
- **Backup file:** 761-byte AES-256-GCM blob (12-byte IV, ciphertext,
  16-byte GCM tag)
- **Decryption helper:** `vault_decrypt.py` — takes a 64-hex AES-256 key
  and decrypts the backup

## Vulnerability

Two weaknesses in `CryptoManager.java`:

**Weak KDF.** The AES-256 key is derived via PBKDF2-HMAC-SHA256 with only
10,000 iterations and a hardcoded salt:

| Parameter | Value |
|-----------|-------|
| Algorithm | `PBKDF2WithHmacSHA256` |
| Passphrase | `"fluidvault_master_seed_2026"` + deviceId |
| Salt | `a915c3e00dbb5a55ba13d7cdaf3a126e` |
| Iterations | 10,000 |
| Key length | 256 bits |

**Low-entropy device identifier.** The deviceId is derived from
`Build.SERIAL`:

```java
private String getDeviceIdentifier() {
    String serial = Build.SERIAL;
    if (serial == null || serial.equals("unknown")) {
        serial = DEFAULT_DEVICE_ID;       // "DEFAULT_DEVICE_ID"
    }
    return Integer.toHexString(serial.hashCode());
}
```

`String.hashCode()` produces only a 32-bit int, but more importantly:
on Android 10+, `Build.SERIAL` almost always returns `"unknown"`, which
triggers the fallback to `DEFAULT_DEVICE_ID` = `"DEFAULT_DEVICE_ID"`.
No device is needed — the deviceId is fully deterministic.

## Exploitation

1. **Re-implement the key derivation** in Python using the hardcoded
   SEED, SALT, iterations, and the fallback device ID string:

   ```python
   device_hex = format(java_hashcode("DEFAULT_DEVICE_ID"), 'x')  # "c8e21b06"
   passphrase = "fluidvault_master_seed_2026" + device_hex
   key = pbkdf2_hmac("sha256", passphrase, salt, 10000, dklen=32)
   ```

   Java's `String.hashCode()` is `Σ s[i]·31^(n-1-i)` with 32-bit
   overflow — must be emulated exactly.

2. **Decrypt `vault_backup.enc`** with the derived key. The backup
   contains vault entries including the **Admin Portal** password:
   `Fl00d_G4t3s_0p3n!`

3. **Submit the password** to the server:

   ```
   POST /api/vault/admin
   {"master_password": "Fl00d_G4t3s_0p3n!"}
   ```

   The server responds with the flag.

## Flag

```
flag{fe4a8cf9724233bb}
```

## Vulnerability Type

- **CWE-330**: Use of Insufficiently Random Values (device identifier
  falls back to a fixed string)
- **CWE-916**: Use of Password Hash with Insufficient Computational
  Effort (only 10,000 PBKDF2 iterations)
- **CWE-798**: Use of Hard-coded Credentials (seed, salt, and fallback
  ID are in the source code)

## Remediation

- Use a proper device-bound secret (e.g., Android KeyStore-backed key
  wrapped by a user-chosen PIN) instead of `Build.SERIAL`.
- Increase PBKDF2 iterations to at least 600,000 (OWASP 2023
  recommendation).
- Never hardcode cryptographic seeds in source code.

## Timeline

| Time | Event |
|------|-------|
| 2025-XX-XX | Decompiled APK; identified CryptoManager KDF parameters |
| 2025-XX-XX | Re-implemented PBKDF2; brute-forced common serials |
| 2025-XX-XX | Discovered `DEFAULT_DEVICE_ID` fallback; decrypted backup |
| 2025-XX-XX | Extracted admin password; submitted to server; captured flag |
