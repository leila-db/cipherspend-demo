# Testing & Verification

How to build, run, and verify CipherSpend — including the FHE correctness cross-check and
the failure-injection tests that prove results come only from the real encrypted pipeline.

## Prerequisites

- **Python 3.10+**.
- **OpenFHE 1.5.x** installed as a system/`find_package`-discoverable library.
- A C++17 toolchain + CMake ≥ 3.16 (and `libomp` on macOS).

## Build the Stage-5 FHE binaries

```bash
cmake -S stage5 -B stage5/build -DCMAKE_BUILD_TYPE=Release
cmake --build stage5/build -j
```

This produces `stage5/build/{keygen,encrypt,server,decrypt}`. (The parameter validator is
built the same way from `stage4/`: `cmake -S stage4 -B stage4/build && cmake --build
stage4/build -j`, then run `stage4/build/stage4_validate` to print the authoritative CKKS
parameters.)

## Python environment + run

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
python -m app.server          # http://127.0.0.1:5057
```

## Test battery

### 1. Unit + integration suite

```bash
pip install pytest        # test-only dependency, not required to run the app
python -m pytest -q
```

**Result: 81 passed.** This covers the analytics contract, the packing/overflow
conservation proofs, the plaintext twin matching the direct aggregate, and the categorizer
tests below. Several of these run the **real Stage-5 binaries** end to end.

### 2. Categorizer — generalization, privacy, non-blocking policy

Inside the suite (`tests/test_categorize.py`):

- **Generalizes to unseen/international merchants** — assigns correct categories to merchants
  not present verbatim in training. Measured on the held-out probe sets: unseen US 94%
  (15/16), international ES/DE/FR/IT 100% (18/18).
- **Zero external network** (`test_no_network`) — monkeypatches `socket.connect` /
  `create_connection` to raise, then categorizes; **any network attempt fails the test**.
- **Non-blocking review** — low-confidence rows are analyzed under a best-guess category and
  offered for optional review; they are never dropped and never block analysis.
- **Browser/Python parity** — the in-browser JS classifier and the Python reference produce
  identical categories for the same inputs.

### 3. FHE correctness — plaintext-oracle cross-check

Runs the full pipeline over the synthetic demo dataset and compares the decrypted aggregates
against `reference.aggregate.direct_aggregate` (exact, no encryption):

| Stream | FHE vs exact oracle (max abs) | (max rel) |
|---|---|---|
| category | 4.2×10⁻⁶ | 4.5×10⁻⁸ |
| weekly | 5.3×10⁻⁶ | 6.9×10⁻⁸ |
| income | 3.2×10⁻⁶ | 5.1×10⁻¹⁰ |

The numpy "twin" of the fold matches the oracle to ~10⁻¹³ (layout is exact); the ~10⁻⁶
residual is CKKS noise. See [FHE_ARCHITECTURE.md](FHE_ARCHITECTURE.md#correctness--plaintext-oracle-cross-check).

### 4. Failure injection — results come only from the real pipeline

`tests/test_categorize.py::test_disabling_stage5_binary_blocks_all` points the pipeline at a
non-existent binary directory and asserts that **both** the demo path and the uploaded-data
path **raise** rather than returning any cached/precomputed/plaintext result. There is no
fallback: a missing or failing Stage-5 binary fails the run. (Confirmed by inspection of
`app/pipeline.py`: the dashboard is always recombined from freshly decrypted `.f64` files.)

### 5. Raw three-column upload — full row accounting

A synthetic **1,030-row** CSV with only `date,description,amount` (no category/type columns)
runs through import + categorizer + FHE pipeline:

```
1030 of 1030 rows analyzed (100.0%); dropped rows = 0
categorization: auto=950  other=50 (low-confidence, best-guess, optional review)  flagged/blocking=0
=> all 1030 reached the real FHE pipeline (2 batches, 12 months)
```

This demonstrates the current upload flow **analyzes every row** (no silent discard) and
surfaces an honest "X of Y rows will be analyzed"; the earlier silent-row-discard bug is
fixed and analysis is blocked only when rows are structurally unusable.

> Note on datasets: all figures here come from **synthetic** data — the deterministic
> generator in `app/demo.py` (520 transactions) and the 1,030-row synthetic upload above.
> None of this is real transaction history.

### 6. Cold demo run, in browser

Wipe keys for a cold run, start the server, and click **Try demo dataset**:

```bash
rm -rf app/run/client_home app/run/server_home app/run/jobs
python -m app.server
```

Verified in-browser: the dashboard renders from a fresh run, and the server log shows the
real binaries executing with fresh artifacts —

```
[stage5] exec .../keygen  rc=0 in ~0.6 s      # cold; 0 when reused
[stage5] exec .../encrypt rc=0
[stage5] exec .../server  rc=0
[stage5] exec .../decrypt rc=0
[stage5] artifact key ... client_home/sk.bin  # secret key: client only
                            server_home/{cc,rotkeys}.bin  # NO sk.bin on server
```

The localhost guard was also verified live: a cross-origin `POST /api/*` → **403**, a
forbidden `Host` header → **403**, a same-origin request → **200**.

## Reproducing the benchmark / oracle harness

The cross-check and cold/warm benchmark are driven by a small harness that calls
`app.pipeline.run_pipeline` directly and compares to `reference.aggregate.direct_aggregate`.
Set `CIPHERSPEND_KEEP_ARTIFACTS=1` to preserve a run's job directory (ciphertexts + decrypted
`.f64`) for inspection instead of deleting it.
