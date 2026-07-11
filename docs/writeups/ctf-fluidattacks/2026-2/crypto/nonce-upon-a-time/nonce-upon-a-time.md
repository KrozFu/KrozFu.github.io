# Nonce Upon a Time Challenge

<div class="grid cards" markdown>

- :material-information-outline: &nbsp; **Challenge Info**

    ---

  - 🏷️ **Category:** Crypto
  - 💻 **Platform:** Fluid Attacks CTF 2026-2
  - 🎯 **Target:** [chal.ctf.ae](https://8caecc81907bc1d4.chal.ctf.ae/)
  - 🚩 **Flag:** `flag{4eb972f5cda22208}`

- :material-tools: &nbsp; **Tools Required**

    ---

  - `python3` (`requests`) — Signature collection from `/api/sign`
  - `fpylll` — LLL/BKZ lattice reduction (Hidden Number Problem)
  - `ecdsa` / `hashlib` — Admin token forgery with SHA-256 digest signing

</div>

!!! abstract "Challenge Description"
    A license-activation server signs JSON tokens with ECDSA (NIST P-256). `/api/sign` signs
    any message except those with `"type": "admin"`; `/api/activate` returns the flag for a
    validly signed admin token. The nonce generation uses `random.getrandbits(256) >> 8`,
    producing nonces in `[0, 2^248)` — an 8-bit bias that leaks the private key through a
    Hidden Number Problem lattice attack.

## Recon

- `GET /api/info` reveals the curve (P-256), order, generator, and public
  key.
- `POST /api/sign` with `{"message": "..."}` returns `r`, `s` and the
  SHA-256 hash.
- The server provides the **source code** showing the nonce weakness and
  a commented-out RFC 6979 fallback.

## Vulnerability

The nonce `k` satisfies `0 ≤ k < 2^248`. Since the curve order
`n ≈ 2^256`, each nonce leaks its 8 most significant bits (they are
always 0). This is a Hidden Number Problem (HNP): given equations

```
k_i ≡ r_i·s_i⁻¹ · d + z_i·s_i⁻¹ (mod n),   0 ≤ k_i < 2^248
```

find the private key `d`.

## Exploitation

1. **Collect signatures** — 200 calls to `/api/sign` (rate-limited to
   10/sec).
2. **Build the HNP lattice** — dimension `N+2` with scaling factor
   `S = n//B ≈ 255` to balance the matrix entries:

   ```
   [ S·a_0  S·a_1  …  S·a_{N-1} | 1  0 ]
   [ S·n    0       …  0         | 0  0 ]
   [ 0      S·n     …  0         | 0  0 ]
   [ …                              …   ]
   [ 0      0       …  S·n       | 0  0 ]
   [ S·b_0  S·b_1  …  S·b_{N-1} | 0  B ]
   ```

   The short vector `(S·k_0, …, S·k_{N-1}, d, B)` appears in the
   reduced basis.
3. **BKZ reduction** — `BKZ(block_size=40)` with `N=40` signatures.
4. **Recover `d`** — iterate reduced rows; a row where all `k_i < 2^248`
   and whose `d` matches the public key (up to y-negation) is the key.
   If `d·G` yields the negated y-coordinate, use `n-d` instead.
5. **Forge admin token** — sign `{"type":"admin"}` with the recovered key.
   **Critical**: the server uses SHA-256 internally, but the `ecdsa`
   library defaults to SHA-1.  Use `sk.sign_digest(hashlib.sha256(msg).digest())`
   to produce a server-compatible signature.

The lattice attack was implemented solely with `fpylll` (no SageMath).

## Flag

```
flag{4eb972f5cda22208}
```

## Vulnerability Type

- **CWE-330**: Use of Insufficiently Random Values
- **CWE-1240**: Use of a Cryptographic Primitive with a Risky Implementation
- **OWASP A02:2021**: Cryptographic Failures

## Remediation

- Use RFC 6979 deterministic nonces (as the commented-out code suggests).
- Ensure the signing library uses the same hash algorithm as the verifier.

## Timeline

| Time | Event |
|------|-------|
| 2025-XX-XX | Session start; collected 200 signatures |
| 2025-XX-XX | Initial lattice attempts (LLL, BKZ) failed on spurious short vectors |
| 2025-XX-XX | BKZ-40 with N=40 recovered the private key |
| 2025-XX-XX | Discovered SHA-1 vs SHA-256 mismatch; forged admin token |
