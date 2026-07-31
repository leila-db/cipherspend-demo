# Privacy Claims

A precise, claim-by-claim statement of what CipherSpend does and does not protect, so that
nothing in the UI or docs is taken to mean more than it does. This is a **local educational
demo**, not a production financial product and not financial advice.

## Claims we make

1. **Transaction values are computed under encryption.** The monthly aggregates (per
   category, per week, per month income) are produced by a homomorphic **rotate-and-sum
   fold on CKKS ciphertext**. The server role never sees the plaintext amounts. Parameters:
   CKKS, ring 65536, **HEStd_128_classic (128-bit)**, no bootstrapping. See
   [FHE_ARCHITECTURE.md](FHE_ARCHITECTURE.md).

2. **The secret key stays on the client.** `keygen` writes `sk.bin` only to the client home;
   the server home receives the context and rotation keys only. The `server` binary
   **refuses to run** (exit 2) if a secret key is present in its home, and the Python
   pipeline re-checks the same invariant. The key file is created `0600` in a `0700`
   directory.

3. **Categorization is on-device and offline.** Merchant→category assignment runs locally
   with **zero network access** (enforced by an automated test). It is a character n-gram
   logistic-regression classifier + keyword rules — **not** an embedding model or a running
   LLM.

4. **Nothing leaves the machine.** The server binds `127.0.0.1` only, makes no external
   calls, and logs only metadata (paths, sizes, timings, exit codes) — never key material,
   ciphertext, or decrypted values.

## What the server role can still observe

FHE hides the *values*, not the *shape* of the computation. The server unavoidably sees:

- **Ciphertext size** — fixed by the CKKS parameters, independent of your data.
- **Number of monthly batches** — a coarse, padding-blurred indicator of how many months of
  history you have (fixed-shape padding blurs it; it is not the transaction count).
- **Timing** — how long the computation ran. The circuit is **data-oblivious** (the same 11
  rotations + 11 adds per batch regardless of values), so timing does not reveal values.

## What we explicitly do NOT claim

1. **Not protection of a compromised device or same-user process.** Everything runs locally.
   Anyone who controls the machine — malware, another process as the same user, an admin —
   can read the plaintext before encryption or after decryption. FHE does not change that.

2. **Not integrity or verifiable computation.** The model is **semi-honest**. FHE keeps your
   values confidential but does **not** detect a server that returns a wrong or altered
   ciphertext. A buggy or malicious server would yield a wrong result the encryption cannot
   catch. This limitation is surfaced in the app's own limitations list.

3. **Not exactness.** CKKS is approximate. Decrypted aggregates carry a small numerical error
   (measured max ≈ 5×10⁻⁶ currency units end-to-end vs. an exact plaintext oracle) — cent-
   accurate, but not bit-exact.

4. **Not a hosted service.** There is no remote operator. "Client" and "server" are
   co-located logical roles in this demo. UI copy has been written to avoid implying a
   remote company that "can't see your data."

5. **Not "100% categorized" = "100% accurate."** Every transaction is assigned a category
   (so nothing is silently dropped), but that does not mean every assignment is correct.
   Low-confidence rows are analyzed under a best-guess category and offered for **optional**
   review — never blocking. Measured generalization on held-out merchants is ~94–100% on a
   small probe set (see [README](README.md#transaction-categorization)); real-world inputs
   will include harder cases.

6. **Not protection of the metadata** listed in the section above.

## Local data handling

- **Only synthetic data ships** in this repository. No real transactions, keys, ciphertexts,
  decrypted results, or caches are tracked (verified against the full git history).
- **Your uploads never leave your device**, are held in the browser's memory (and, during a
  run, as transient ciphertext + cleartext under the git-ignored `app/run/jobs/<id>/`, then
  deleted), and are never transmitted anywhere.
- **Your manual category corrections** are cached locally only in the git-ignored
  `app/run/merchant_cache.json` and are never transmitted.
