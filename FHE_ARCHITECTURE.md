# FHE Architecture

Authoritative description of the CKKS scheme, parameters, circuit, and measured behavior.
Every number below was produced by the code in this repo (`stage4/stage4_validate.cpp`
for parameters, `stage5/` for the circuit, and the benchmark/oracle harness described in
[TESTING.md](TESTING.md)).

## Scheme and parameters

Scheme: **CKKS** (`CryptoContextCKKSRNS`, OpenFHE 1.5.x). Configured in
[`stage5/keygen.cpp`](stage5/keygen.cpp) and independently dumped by
[`stage4/stage4_validate.cpp`](stage4/stage4_validate.cpp):

| Parameter | Value |
|---|---|
| Ring dimension `N` | **65536** (explicitly set, never inferred) |
| Slots (`N/2`) | 32768 |
| Security level | **HEStd_128_classic** (128-bit) |
| Multiplicative depth (set) | 1 (library minimum; **the circuit uses depth 0**) |
| Scaling technique | FLEXIBLEAUTO |
| Key-switch technique | HYBRID |
| Scaling modulus size | 50 bits |
| First modulus size | 60 bits |
| **Ciphertext modulus chain** | **2 RNS limbs, total log₂Q ≈ 110 bits** |
| Bootstrapping | **none** |
| Rotation keys | exactly **{128, 256, 512, 1024, 2048, 4096}** (6 keys) |
| Relinearization / `EvalMult` key | **none** (there is no ciphertext×ciphertext multiply) |

Measured serialized sizes (from `stage4_validate`): crypto context 573 B, public key
≈ 2.10 MB, secret key ≈ 1.05 MB, one fresh ciphertext ≈ 2.10 MB, the 6 rotation keys
≈ 37.75 MB total. Encrypt→decrypt round-trip error over all 32768 slots: max abs ≈ 1.0×10⁻⁹.

## Data layout — two ciphertexts per batch

Each month-group is packed into **two ciphertexts** using an interleaved slot layout
(constants in [`reference/layout.py`](reference/layout.py) and
[`stage5/pipeline_common.hpp`](stage5/pipeline_common.hpp), which agree):

- **ct1 — category stream.** 12 months × 10 categories = 120 buckets, each with `L1 = 32`
  lanes (capacity for up to 32 transactions per (month, category) before spilling to a new
  batch). Folded with rotations `{128, 256, 512, 1024, 2048}`.
- **ct2 — weekly + income stream.** 12 months × 5 week-bins = 60 weekly buckets + 12 income
  buckets = 72 buckets, each with `L2 = 64` lanes. Folded with rotations
  `{128, 256, 512, 1024, 2048, 4096}`.

Amounts are scaled to bounded integers before encryption. When a (month, category, ...)
lane count exceeds its capacity, the packer **spills into an additional batch** — nothing is
truncated or double-counted (verified by `tests/test_overflow.py` and
`tests/test_conservation.py`).

## The server circuit — depth 0, linear only

The server ([`stage5/server.cpp`](stage5/server.cpp)) performs, **per batch**, a
rotate-and-sum *fold* on each of the two ciphertexts:

```
fold(ct, rotations):           # e.g. server.cpp
    acc = ct
    for r in rotations:
        acc = EvalAdd(acc, EvalRotate(acc, r))
    return acc
```

That is the entire homomorphic computation. **Op count per batch:** 5 rotations + 5 adds on
ct1, 6 rotations + 6 adds on ct2 = **11 `EvalRotate` + 11 `EvalAdd`**. For the demo dataset
(520 transactions → 2 batches):

| Homomorphic op | Count (2 batches) |
|---|---|
| `EvalRotate` | 22 |
| `EvalAdd` | 22 |
| `EvalMult` (ct × plaintext) | **0** |
| `EvalMult` (ct × ct) | **0** |
| Rescale / ModReduce | **0** |
| Relinearize | **0** |
| Bootstrap | **0** |

There is **no multiplication of any kind**, hence no relinearization, no rescaling, and no
consumption of modulus levels — which is why the circuit lives entirely at depth 0 on a
2-limb ciphertext, and why no relin key is generated. There is **no plaintext or cached
fallback**: a missing/failing binary raises `PipelineError` and the run fails; the dashboard
is always rendered from freshly decrypted ciphertext (verified in-browser — see below).

## Measured performance (real binaries)

End-to-end wall-clock over the demo dataset (520 transactions, 2 batches), running the
actual compiled Stage-5 binaries:

| Stage | Cold (fresh keys) | Warm (keys reused) |
|---|---|---|
| Key generation (context + 6 rot keys) | ~0.2–0.6 s | 0 (reused) |
| Local pack + encrypt (4 ciphertexts) | ~0.10 s | ~0.10 s |
| **Homomorphic compute (server)** | ~0.29 s | ~0.29 s |
| Decrypt (4 ciphertexts) | ~0.20 s | ~0.20 s |
| **Total** | **~0.8–1.1 s** | **~0.6 s** |

Ciphertext transferred per run: ≈ 8.39 MB in (4 ct) and ≈ 8.39 MB out (4 ct). Rotation keys
(sent once) ≈ 37.75 MB. Key generation is data-independent and cached across runs.

## Correctness — plaintext-oracle cross-check

The decrypted FHE aggregates are cross-checked against an **exact plaintext oracle**
(`reference.aggregate.direct_aggregate`, which sums raw amounts with no encryption). Over
the demo dataset:

| Stream | Buckets | Float-twin vs oracle (max abs) | **FHE vs oracle (max abs)** | FHE vs oracle (max rel) |
|---|---|---|---|---|
| category | 240 | 2.3×10⁻¹³ | **4.2×10⁻⁶** | 4.5×10⁻⁸ |
| weekly | 120 | 9.1×10⁻¹³ | **5.3×10⁻⁶** | 6.9×10⁻⁸ |
| income | 24 | 0 | **3.2×10⁻⁶** | 5.1×10⁻¹⁰ |

The float "twin" (a numpy emulation of the fold) matches the oracle to ~10⁻¹³, proving the
slot **layout is exact**; the ~10⁻⁶ residual on the real pipeline is the **CKKS
approximation noise** — negligible for a currency dashboard (values agree to the cent). Cold
and warm runs produce the same result (differing only by encryption noise ~10⁻⁶).

## Why this circuit is fast (comparison to a deeper FHE app)

For scale, compare against *VitalVault*, a CKKS wellness-scoring app that computes a
**nonlinear** score. Same ring dimension (65536), but structurally very different:

| | CipherSpend (this app) | VitalVault |
|---|---|---|
| Multiplicative depth | **circuit 0** (set 1) | **20** |
| Ciphertext modulus chain | **2 RNS limbs** (~110-bit Q) | ~21 limbs (deep chain) |
| Server operations | 11 rotations + 11 adds per batch | per marker: a **degree-13 Chebyshev** penalty (`EvalChebyshevFunction`) + ct×ct multiplies |
| ct×ct multiplies | **0** | many (`penalty·weight`, `z·z`, iterated `facc·facc` dependent squaring) across **25 markers** |
| Relinearization key | **none** | required (`EvalMultKeyGen`) |
| Rescaling | **none** | after every multiply |

The speed gap is structural, and compounds two ways:

1. **Fewer, cheaper op *types*.** CipherSpend does only *linear* operations (rotations and
   additions). VitalVault evaluates high-degree nonlinear polynomials on ciphertext — each
   Chebyshev evaluation is a chain of ct×ct multiplies, each of which needs a tensor product
   + relinearization (a key-switch) + a rescale. A multiply is far more expensive than a
   rotation, and there are hundreds of them (× 25 markers) versus **zero** here.
2. **A shallower modulus chain.** Every homomorphic operation costs roughly in proportion to
   the number of RNS limbs. CipherSpend operates on **2** limbs; a depth-20 circuit operates
   on ~21, so *even the individual rotations/adds* would be ~10× cheaper here — before
   accounting for the fact that CipherSpend has no multiplies at all.

In short: a data-oblivious depth-0 linear fold on a minimal 2-limb ciphertext is about as
cheap as CKKS gets, which is why this demo completes a full encrypted run in well under a
second warm.
