# Security & Privacy Review

Pre-publication review of CipherSpend, a **local educational FHE demo**. Priorities, as
scoped: material oversights, misleading claims, and accidental exposure of local data —
over exotic attacks. Reviewers covered the tracked source, the localhost runtime, upload/
subprocess handling, the categorizer, and the git working tree **and full history**.

**Outcome: no Critical or High severity issues in the code.** The one Critical item was a
working-tree hygiene gap (an un-ignored secret key on disk), now fixed. Two Medium items
(one misleading-claim, one localhost hardening) and several Low/Informational items are
resolved below.

## Threat model (honest)

- **Local, single-user, single-machine.** "Client" and "server" are two logical roles in
  the same process on the same machine. There is no remote operator and nothing is hosted.
- **Semi-honest server.** FHE keeps transaction *values* confidential from the server role.
  It does **not** provide integrity: a wrong or manipulated result is **not** detected by
  the encryption. No verifiable-computation claim is made.
- **In scope:** confidentiality of transaction values against the aggregation role; keeping
  the secret key client-only; not leaking local data on publication; not overclaiming.
- **Explicitly out of scope:** a compromised device, a same-user process, malware, or a
  machine admin (all can read plaintext before encryption / after decryption); and the
  unavoidable metadata the server observes (ciphertext size, batch count, timing).

## Findings

Severity, location, impact, fix, and status. Line numbers refer to the state at review time.

### Critical

**C1 — Un-ignored secret key + ciphertext in the working tree.** `artifacts/` (top-level,
produced by the Stage-4 validator) contained `sec.bin` (CKKS secret key), `ct.bin`
(ciphertext), and the rotation keys, and was **neither tracked nor git-ignored** — a
`git add -A` would have staged a secret key into a public repo.
**Fix (done):** `.gitignore` now ignores `artifacts/`, `**/artifacts/`, `**/run/`,
`**/client_home/`, `**/server_home/`, and `*.bin`/`*.ct`/`*.f64`/`merchant_cache.json`
belt-and-suspenders, so no key/ciphertext can be added from any directory. Verified: `git
check-ignore` now covers the file, and no tracked file became newly ignored.

### Medium

**M1 — Landing copy implied a remote operator ("even from us").** `web/app.js` (landing
kicker + sub-hero). For a local demo with no remote operator, "encrypted … even from us"
overstated the trust model.
**Fix (done):** reworded to "Analyzed under encryption — unlocked only on your device" and
"A local demo … the analytics run on ciphertext, and only your device holds the key." The
duplicate copy in `prototype/index.html` is **excluded from publication** (see below).

**M2 — No Host/Origin validation on the localhost API.** `app/server.py`. Requests used
`get_json(force=True)`, so a web page the user has open could issue cross-origin
`text/plain` POSTs to `http://127.0.0.1:5057/api/*`; with DNS-rebinding, an attacker origin
could resolve to loopback. (Mitigated: no user data is persisted server-side — results are
in-memory under a 122-bit `uuid4` token — so the practical risk was CPU burn, not
exfiltration.)
**Fix (done):** added a `@app.before_request` guard that (1) allowlists the `Host` header
(defeats DNS-rebinding) and (2) rejects cross-site `Origin`/`Sec-Fetch-Site` on `/api/*`
(defeats CSRF); replaced `force=True` with a strict `_json_body()`; set
`MAX_CONTENT_LENGTH = 64 MB`. Verified live: cross-origin POST → 403, forbidden Host → 403,
same-origin → 200.

### Low / Informational

**L1 — Secret key written with default umask.** `stage5/keygen.cpp`, `app/pipeline.py`.
The CKKS secret key could be world-readable on a multi-user host.
**Fix (done):** `keygen.cpp` now `chmod`s the client home to `0700` and `sk.bin` to `0600`;
`pipeline.py` re-applies the same after keygen (belt-and-suspenders). Verified: fresh
`client_home` is `drwx------`, `sk.bin` is `-rw-------`.

**L2 — Result token travels in a URL query string.** `app/server.py` `GET /api/result?token=`.
The token is a bearer capability and lands in the request-line log. Local-only, in-memory,
122-bit unguessable token. **Status: documented** as a known local-scope limitation (the
result is never persisted; the log stays on the machine). Not changed to avoid churn on a
Low item.

**L3 — "no sensitive plaintext persists" was optimistic.** `app/pipeline.py`. The
pre-encryption packed vectors and the decrypted `.f64` aggregates are cleartext written to
the git-ignored `app/run/jobs/<id>/` during a run (best-effort deleted; `CIPHERSPEND_KEEP_
ARTIFACTS=1` preserves them). **Fix (done):** comment corrected to state this accurately.

**L4 — Server-view banner called the illustrative hex "the ciphertext the server actually
holds".** `web/app.js`. The scrambled hex is a client-side hash, not real OpenFHE bytes.
**Fix (done):** reworded to "an illustrative stand-in … a visual placeholder, not the real
OpenFHE bytes."

**I1 — No user-facing integrity caveat.** `analytics/schema.py`, Privacy page. **Fix
(done):** added a limitation line: the encryption assures confidentiality against a
semi-honest server but does not verify the server computed correctly.

## Areas checked and found OK

- Binds `127.0.0.1` only; Flask `debug=False` (no Werkzeug debugger / stack-trace leak);
  worker returns a sanitized `type(ex).__name__: ex` and never echoes transaction data.
- No wildcard CORS anywhere.
- **Zero external network:** no `requests`/`urllib`/`socket`/`http(s)://` in any tracked
  Python; every frontend `fetch` is a same-origin relative path; the categorizer loads its
  model locally and makes no network calls (enforced by `tests/test_categorize.py::test_no_network`).
- **Secret-key trust boundary:** `server.cpp` refuses to run (exit 2) if `server_home/sk.bin`
  exists; `pipeline.py` re-checks the same invariant; `keygen.cpp` writes `sk.bin` only to
  the client home.
- **No arbitrary-file / key serving:** the only file route is `send_from_directory(WEB, …)`
  (blocks traversal); keys live outside `web/`; `/api/*` is 404'd by the static route.
- **Subprocess safety:** all `subprocess.run` calls use array args, never `shell=True`.
- **Job-id / token:** server-generated `uuid4().hex` (122-bit, unguessable); the job-dir
  name is that hex (`[0-9a-f]`, no traversal); **uploaded filenames never reach the server**
  (the file is read client-side; only its text is sent).
- **Upload limits:** `MAX_TRANSACTIONS = 200_000`; `MAX_CONTENT_LENGTH = 64 MB`; job dirs
  removed after each run.
- **Logging:** only artifact paths, sizes, mtimes, exit codes, timings — never key material,
  ciphertext, or decrypted values.
- **Crypto parameters:** the mandated secure set only (N=65536, 128-bit, no bootstrapping,
  depth-0 circuit, 6 rotation keys, no relin); no insecure/"testing" flags; no cleartext
  crypto fallback.
- **`local_llm_refine`:** confirmed an unimplemented stub returning `None`; no UI/doc claims
  an LLM runs on transactions.

## Data-leak audit

The working tree **and full git history** were audited (as of the pre-publication audit commit `2f501f8`).

- **Git history is clean.** Every file ever added is identical to the currently-tracked set;
  there were zero deletions. **No secret key, ciphertext, real CSV, cache, or log ever
  entered history.** The largest blob ever committed is the categorizer model
  (`web/categorizer.json`, ~592 KB). → A fresh sanitized history is **not required** for
  safety.
- **Only tracked data file:** `web/categorizer.json` — a char-n-gram linear model (categories,
  generic n-gram vocab, coefficients, keyword rules, and an integer training-example count).
  No amounts, account numbers, or personal identifiers. Safe to publish.
- **Synthetic-only demo data.** All CSV fixtures and the demo generator are synthetic; they
  are git-ignored regardless.
- **Sensitive files present on disk but git-ignored (never committed), which must never be
  force-added:** `app/run/merchant_cache.json` (a local merchant→category cache that can
  contain **real** bank-feed descriptors from any real CSV the user tested with),
  `**/sk.bin`/`sec.bin` (secret keys), serialized ciphertexts, decrypted `.f64` files, and
  build dirs. All are covered by the hardened `.gitignore`.
- **Secrets / PII scan of tracked files:** every `secret|password|token|key` hit is a
  crypto-context term or a UI/session/design token — no credentials; no card/account/routing
  numbers; no third-party emails or phone numbers; no `.env`/`.pem` files.
- **Local-path leak:** the internal `HANDOFF.md` hard-codes `/Users/<user>/…` paths. It is an
  internal development document and is **excluded from the published tree** (along with
  `prototype/` and other internal design docs), so nothing hard-codes a local path in what is
  published.

**Publication rule followed:** the public repo is built from an explicit file allow-list — not
`git add -A` — and `git ls-files` on the result is scanned before pushing. See
[TESTING.md](TESTING.md) and the publication notes in the release process.
