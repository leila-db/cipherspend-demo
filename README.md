# CipherSpend

A **local, educational demo** of privacy-preserving personal-finance analytics built on
**CKKS fully homomorphic encryption**. It runs on standard
[OpenFHE](https://github.com/openfheorg/openfhe-development) and was designed with
[Niobium's FHEanna](#niobium-ecosystem) FHE application-design methodology.
Your transactions are encrypted on your device; a "server" role computes monthly
aggregates **directly on the ciphertext** and never holds the key to read them; the
results are decrypted only on your device.

> **This is a demo for learning and amusement — not a production financial product, and
> not financial advice.** It runs entirely on one machine. The "client" and "server" are
> two logical roles in the same local process. See [what this does and does not protect](#what-this-demo-does-and-does-not-protect).

---

## What runs where

CipherSpend deliberately keeps the FHE circuit tiny and does everything else in the
clear **on your own device**. Being precise about the split is the whole point:

| Step | Where | On encrypted or plaintext data? |
|---|---|---|
| Parse CSV, sign/type detection | your device | plaintext |
| **Categorize each transaction** (merchant → category) | your device | plaintext |
| Pack transactions into the fixed slot layout, scale to integers | your device | plaintext |
| **Encrypt** each batch (2 ciphertexts) | your device | plaintext → ciphertext |
| **Homomorphic aggregation** (rotate-and-sum fold) | server role | **ciphertext only** |
| **Decrypt** the aggregates | your device | ciphertext → plaintext |
| Ratios, trends, month-over-month, narratives | your device | plaintext |

**The only thing the FHE circuit computes is a blind linear sum**: it folds the packed
per-transaction contributions into per-(month, category), per-(month, week) and
per-(month) income totals. All the "analytics" you see (savings rate, cash-flow index,
cosine similarity between months, story cards) are computed **locally after decryption** —
they are not part of the encrypted computation.

### What the server role sees / does not see

**Cannot see:** transaction amounts, categories, merchants, individual transactions, dates,
your secret key, or any readable result. It receives ciphertext and returns ciphertext.

**Can see (unavoidable metadata):** the **size** of the ciphertext (fixed by the parameters,
not your data), the **number of monthly batches** (a coarse, padding-blurred sense of how
many months of history you have), and **timing** (how long the computation ran — which is
independent of your values, since the circuit is data-oblivious).

---

## Implementation stack

The entire FHE computation is **hand-written C++ against upstream OpenFHE 1.5.1 (CKKS)**.
No special hardware or SDK is needed to build and run it.

| Layer | What it is |
|---|---|
| FHE circuit | Four hand-written C++ programs on **upstream OpenFHE 1.5.1** (CKKS) — `keygen`, `encrypt`, `server`, `decrypt` in `stage5/` |
| Orchestration | Python 3 + Flask (local server, binds `127.0.0.1`) |
| Analytics | Python (`analytics/`, `reference/`) — runs locally on decrypted aggregates |
| Frontend | Vanilla-JS single-page app (no build step) |
| Categorizer | Char n-gram logistic-regression model (JSON), run in-browser JS + Python |
| Build / run | `cmake` + `find_package(OpenFHE)`; `python -m app.server` |

---

## What this demo does and does not protect

**Does:** keep your transaction *values* confidential from the server role that performs
the aggregation, under a **128-bit** CKKS parameter set, with the **secret key confined to
the client** (the server binary refuses to run if a secret key is present in its home).

**Does not:**
- **Not** a defense against a compromised device, another process running as the same
  user, malware, or a machine administrator. Everything is local; whoever controls the
  machine can read the plaintext before encryption or after decryption.
- **Not** integrity/verifiable computation. The trust model is **semi-honest**: FHE keeps
  your values confidential, but it does **not** detect a server that returns a wrong or
  manipulated ciphertext. A buggy or malicious server would produce a wrong result that
  the encryption itself cannot catch.
- **Not** protection of the metadata listed above (ciphertext size, batch count, timing).
- **Not** hosted. There is no remote operator; "client" and "server" are co-located
  logical roles in this demo.

Values are **CKKS-approximate**: the decrypted aggregates carry a tiny numerical error
(measured max ≈ **5×10⁻⁶ currency units** end-to-end against an exact plaintext oracle —
see [FHE_ARCHITECTURE.md](FHE_ARCHITECTURE.md)), so figures are effectively cent-accurate
but not bit-exact.

---

## Transaction categorization

Categories are assigned **on your device with zero network access**. The categorizer is a
**character n-gram logistic-regression text classifier** (plus a small set of hand-authored
keyword rules), trained offline and exported to JSON; the same model runs in the browser as
pure JavaScript and in Python as the reference. **It is not an embedding model and not a
running LLM.** (There is an unused `local_llm_refine` interface in the code — it is an
**unimplemented stub that returns `None`**; no LLM runs on your transactions.)

**Measured generalization** on held-out merchants *not present verbatim in the training
data* (a small probe set, so illustrative rather than a large benchmark):

| Probe set | Accuracy |
|---|---|
| Unseen US merchants (16) | 94% (15/16) |
| International merchants — ES/DE/FR/IT (18) | 100% (18/18) |
| Combined (34) | 97% (33/34) |

Model: 10 categories, 6,952 n-gram features, trained on 3,054 examples.

**Honest limitations:** low-confidence transactions are **not** dropped — they are analyzed
under a best-guess category and surfaced for **optional** review (never blocking). Being
"100% categorized" means every row was assigned a category, **not** that every assignment is
correct. Your manual corrections are cached **locally only** (`app/run/merchant_cache.json`,
git-ignored) and are never transmitted.

---

## Run it

Requirements: Python 3, a built OpenFHE 1.5.1 system install (any 1.5.x should work), and the compiled Stage-5
binaries in `stage5/build/` (`keygen`, `encrypt`, `server`, `decrypt`). See
[TESTING.md](TESTING.md) for building and the full verification battery.

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
python -m app.server
```

Then open `http://127.0.0.1:5057` and click **Try demo dataset** (synthetic 14-month
sample) or **Use my own data** to upload a CSV. The server binds to **127.0.0.1 only**,
runs with Flask debug **off**, and makes no external network calls of any kind.

## Niobium ecosystem

CipherSpend was designed with [FHEanna](https://github.com/NiobiumInc/niobium-skills),
Niobium's FHE application-design skill (`fhe-application-design`). Every source file here is
headed *"Generated by the Niobium FHE Application Design AI Assistant (FHEanna)."* FHEanna set
the crypto choices the app follows: CKKS at ring dimension `2^16`, at least 128-bit security,
no bootstrapping (multiplicative depth 0), a fixed rotation-key set, no relinearization,
secret-key-client-only, and a semi-honest threat model. Those choices are what keep the
circuit small and easy to reason about (see [FHE_ARCHITECTURE.md](FHE_ARCHITECTURE.md)).
Niobium's FHE guides, *"Building Your First FHE Application"* and *"What FHE can and cannot
do"*, informed the depth-0 scope and the privacy framing in
[PRIVACY_CLAIMS.md](PRIVACY_CLAIMS.md).

It runs on standard upstream OpenFHE with no special hardware. The circuit is depth-0 and
data-oblivious, which also makes it a reasonable fit for hardware acceleration if you want to
go that route: the Niobium stack ([`niobium-client`](https://github.com/NiobiumInc/niobium-client),
the [FHETCH Polynomial IR](https://github.com/NiobiumInc/niobium-fhetch), the Fog compilation
service, and the Mistic FHE accelerator) is one such path.

## Documentation

- **[FHE_ARCHITECTURE.md](FHE_ARCHITECTURE.md)** — authoritative CKKS parameters, the
  two-ciphertext slot layout, the exact server op-count, benchmarks, plaintext-oracle
  correctness, and why this circuit is fast.
- **[PRIVACY_CLAIMS.md](PRIVACY_CLAIMS.md)** — a precise, claim-by-claim privacy statement.
- **[TESTING.md](TESTING.md)** — how to build, run, and verify (including the FHE
  correctness cross-check and the failure-injection tests).

## Demo data only

The repository ships **only synthetic data** (a deterministic generator in `app/demo.py`
and synthetic fixtures generated by `reference/fixtures.py`). No real transaction history,
keys, ciphertexts, decrypted results, or caches are included or tracked.

## License

Released under the [Apache License 2.0](LICENSE).
