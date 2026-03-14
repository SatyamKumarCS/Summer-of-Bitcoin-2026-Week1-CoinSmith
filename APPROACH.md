# My Approach — Coin Smith (Summer of Bitcoin 2026)

> A detailed walkthrough of how I designed, implemented, and optimized the Coin Smith PSBT Transaction Builder.

---

## Table of Contents

1. [Understanding the Problem](#1-understanding-the-problem)
2. [Architecture & Design Decisions](#2-architecture--design-decisions)
3. [Fixture Parsing & Validation](#3-fixture-parsing--validation)
4. [Coin Selection Strategy](#4-coin-selection-strategy)
5. [Fee, Change & Dust Handling](#5-fee-change--dust-handling)
6. [Transaction Weight & vBytes Estimation](#6-transaction-weight--vbytes-estimation)
7. [RBF & Locktime Construction](#7-rbf--locktime-construction)
8. [PSBT Construction (BIP-174)](#8-psbt-construction-bip-174)
9. [Report Generation & Warnings](#9-report-generation--warnings)
10. [Web UI & Server](#10-web-ui--server)
11. [Edge Cases & Challenges](#11-edge-cases--challenges)
12. [Testing Strategy](#12-testing-strategy)
13. [Key Learnings](#13-key-learnings)

---

## 1. Understanding the Problem

The challenge requires building a **safe PSBT transaction builder** that takes a fixture describing a wallet's UTXO set, payment targets, change address template, and fee rate, and produces:

- A **selected input set** (coin selection)
- A valid **PSBT (BIP-174)** containing an unsigned transaction with prevout metadata
- A **JSON report** explaining every aspect of the constructed transaction
- A **Web UI** to visualize and justify the result

The key difficulty lies in the **interplay between fee estimation and change output creation** -- adding or removing a change output changes the transaction size, which changes the required fee, which can flip the decision about whether change is viable. This feedback loop must be handled carefully.

---

## 2. Architecture & Design Decisions

### Modular Design

I split the codebase into focused, single-responsibility modules:

```
builder.py         -> Orchestrates the full build pipeline (CLI entry point)
server.py          -> Web server exposing the same engine via REST API
src/
  fixture.py       -> Defensive fixture parsing and validation
  coin_selection.py -> Greedy UTXO selection algorithm
  fee.py           -> Fee/change calculation with iterative convergence
  estimator.py     -> Transaction weight and vbytes estimation
  rbf_locktime.py  -> nSequence and nLockTime logic
  transaction.py   -> Raw unsigned transaction serialization
  psbt.py          -> BIP-174 PSBT construction
  report.py        -> JSON report assembly and warning generation
static/
  index.html       -> Single-page web UI
```

**Why this structure?**
- The CLI and web app share the **exact same build engine** -- no code duplication.
- Each module handles one concern, making it easy to test and debug independently.
- The pipeline flows linearly: parse -> select -> compute fee -> build PSBT -> generate report.

### Data Flow

```
Fixture JSON
      |
      v
 +-----------+
 | fixture   | <- Parse and validate all inputs defensively
 +-----+-----+
       |
       v
 +-------------+
 | rbf_locktime| <- Determine nSequence and nLockTime settings
 +-----+-------+
       |
       v
 +-------------+       +-----------+
 | fee         | ----> | estimator | <- Weight-based vbytes estimation
 | (+ coin_sel)|       +-----------+
 +-----+-------+
       |
       v
 +-----------+       +-------------+
 | psbt      | ----> | transaction | <- Serialize unsigned tx
 +-----------+       +-------------+
       |
       v
 +-----------+
 | report    | <- Assemble JSON report with warnings
 +-----------+
       |
       v
  JSON Output
```

---

## 3. Fixture Parsing & Validation

**`fixture.py` -- `parse_fixture()`**

I implemented defensive validation that rejects malformed inputs with structured error codes:

- **Top-level structure**: Must be a JSON object with required fields (`utxos`, `payments`, `change`, `fee_rate_sat_vb`)
- **UTXO validation**: Each UTXO must have a valid 64-char hex txid, non-negative vout, positive value, valid hex scriptPubKey, and known script type
- **Payment validation**: Each payment must have a valid scriptPubKey, known script type, and positive value
- **Change template validation**: Must have a valid scriptPubKey and known script type
- **Fee rate**: Must be a positive number
- **Optional fields**: `rbf` (bool, default false), `locktime` (non-negative int), `current_height` (non-negative int), `policy.max_inputs`
- **Network**: Defaults to "mainnet" if absent; validated against known networks

The hex validation uses a regex pattern to catch malformed hex strings early:

```python
HEX_RE = re.compile(r'^[0-9a-fA-F]+$')
```

Every validation failure raises a `FixtureError` with a descriptive code and message, which flows through to the CLI output as structured error JSON.

---

## 4. Coin Selection Strategy

**`coin_selection.py` -- `select_coins_greedy()`**

I implemented a **greedy largest-first** strategy:

1. Sort all UTXOs by value in descending order
2. Accumulate UTXOs until the running total exceeds the target amount
3. Respect `max_inputs` policy constraints

This approach has tradeoffs:
- **Pros**: Minimizes input count (fewer inputs = smaller tx = lower fee), tends to consolidate dust over time
- **Cons**: Not optimal for privacy (always uses largest UTXOs), doesn't minimize change

The target amount fed to coin selection is a rough estimate that includes payments plus an initial fee guess. The `fee.py` module handles the iterative refinement (see next section).

---

## 5. Fee, Change & Dust Handling

**`fee.py` -- `select_and_compute()` and `compute_fee_and_change()`**

This was the trickiest part of the challenge due to the **feedback loop** between change and fee:

### The Problem

Adding a change output increases the transaction size, which increases the required fee. But if the increased fee pushes the change below the dust threshold (546 sats), we must remove the change output, which decreases the transaction size, which decreases the required fee, which means more leftover goes to fee.

### My Solution: Two-Pass Approach

`compute_fee_and_change()` tries both scenarios for a given set of selected inputs:

1. **Attempt 1 (with change)**: Estimate vbytes with an extra change output, compute the fee, check if leftover is above dust threshold
2. **Attempt 2 (send-all)**: If change is dust, estimate vbytes without change output, verify inputs cover payments + minimum fee, absorb all leftover as fee

### Iterative Convergence

`select_and_compute()` wraps this in an iterative loop:

1. Make a rough fee estimate using 1 hypothetical input
2. Select coins targeting `pay_total + rough_fee`
3. Try to compute fee and change with the selected coins
4. If it fails (insufficient funds), bump the target and re-select (up to 10 attempts)
5. As a last resort, grab the largest UTXOs within `max_inputs` and try send-all

This handles boundary conditions where the naive one-pass approach would fail.

---

## 6. Transaction Weight & vBytes Estimation

**`estimator.py` -- `estimate_weight()` and `estimate_vbytes()`**

I used a deterministic weight-based estimator with per-script-type weights derived from typical mainnet transactions:

### Input Weights (weight units)

| Script Type | Weight | Notes |
|------------|--------|-------|
| p2pkh | 592 | Legacy, no witness data |
| p2wpkh | 271 | Native SegWit v0 |
| p2sh-p2wpkh | 363 | Wrapped SegWit |
| p2tr | 230 | Taproot (Schnorr sig is smaller) |
| p2wsh | 271 | SegWit v0 script hash |

### Output Weights (weight units)

| Script Type | Weight | Notes |
|------------|--------|-------|
| p2pkh | 136 | 34 bytes * 4 |
| p2wpkh | 124 | 31 bytes * 4 |
| p2sh-p2wpkh | 128 | 32 bytes * 4 |
| p2tr | 172 | 43 bytes * 4 |
| p2wsh | 172 | 43 bytes * 4 |

### Transaction Overhead

- Base overhead: 40 WU (version + locktime + vin/vout counts = 10 bytes * 4)
- SegWit marker/flag: +2 WU (if any input is SegWit)

```
vbytes = ceil(weight / 4)
fee = ceil(fee_rate * vbytes)
```

---

## 7. RBF & Locktime Construction

**`rbf_locktime.py` -- `compute_rbf_locktime()`**

Following the spec's interaction matrix exactly:

| rbf | locktime present | current_height | nSequence | nLockTime |
|-----|-----------------|----------------|-----------|-----------|
| false/absent | no | -- | 0xFFFFFFFF | 0 |
| false/absent | yes | -- | 0xFFFFFFFE | locktime |
| true | no | yes | 0xFFFFFFFD | current_height |
| true | yes | -- | 0xFFFFFFFD | locktime |
| true | no | no | 0xFFFFFFFD | 0 |

### Locktime Classification

Based on the final nLockTime value:
- `"none"` if nLockTime == 0
- `"block_height"` if 0 < nLockTime < 500,000,000
- `"unix_timestamp"` if nLockTime >= 500,000,000

### Anti-Fee-Sniping

When `rbf: true` and `current_height` is provided but no explicit `locktime`, we set `nLockTime = current_height`. This follows Bitcoin Core's default behavior to prevent miners from re-mining recent blocks for higher fees.

---

## 8. PSBT Construction (BIP-174)

**`psbt.py` -- `build_psbt()`**

I constructed PSBTs from scratch following BIP-174:

1. **Magic bytes**: `psbt\xff`
2. **Global map**: Key 0x00 = unsigned transaction (serialized by `transaction.py`)
3. **Per-input maps**: Key 0x01 = `witness_utxo` (8-byte value + scriptPubKey with length prefix) for every input
4. **Per-output maps**: Empty separators (0x00 byte each)

### Transaction Serialization

**`transaction.py` -- `serialize_unsigned_tx()`**

Standard Bitcoin transaction serialization without witness:
- Version (4 bytes, little-endian int32)
- Input count (CompactSize varint)
- For each input: reversed txid (32 bytes) + vout (4 bytes) + empty scriptSig + sequence (4 bytes)
- Output count (CompactSize varint)
- For each output: value (8 bytes, little-endian uint64) + scriptPubKey with length
- Locktime (4 bytes)

### CompactSize Encoding

```python
< 0xFD     -> 1 byte
<= 0xFFFF  -> 0xFD + 2 bytes LE
<= 0xFFFFFFFF -> 0xFE + 4 bytes LE
else       -> 0xFF + 8 bytes LE
```

---

## 9. Report Generation & Warnings

**`report.py` -- `build_report()` and `generate_warnings()`**

The JSON report includes all required fields:
- Selected inputs with full UTXO details
- Outputs with payment/change distinction
- Fee and fee rate (actual rate = fee_sats / vbytes)
- RBF signaling status and locktime info
- PSBT as base64 string

### Warning Codes

| Code | Condition |
|------|-----------|
| `HIGH_FEE` | fee_sats > 1,000,000 OR effective fee_rate > 200 sat/vB |
| `DUST_CHANGE` | Change output exists with value < 546 sats |
| `SEND_ALL` | No change output created (all leftover consumed as fee) |
| `RBF_SIGNALING` | Transaction opts into Replace-By-Fee |

---

## 10. Web UI & Server

### Server Architecture

**`server.py`** -- A lightweight HTTP server using Python's built-in `http.server`:

- `GET /api/health` -- Returns `{"ok": true}`
- `POST /api/build` -- Accepts fixture JSON, runs the full build pipeline, returns the report
- `GET /` -- Serves the single-page UI
- CORS headers enabled for development
- Error responses use the same structured error format as the CLI

The server writes posted fixture data to a temp file and reuses the exact same `build_transaction()` function from the CLI, ensuring identical behavior.

### Web UI Features

- Load a fixture JSON via text input or file upload
- Visual breakdown of selected inputs and outputs
- Change output clearly identified with a badge
- Fee, fee rate, and vbytes display
- RBF signaling status indicator
- Locktime value and type display
- Warning badges for all emitted warnings
- PSBT base64 output with copy functionality

---

## 11. Edge Cases & Challenges

### Handled Edge Cases

| Scenario | How I Handled It |
|----------|-----------------|
| Change below dust threshold | Automatically switch to send-all mode |
| Adding change changes fee | Two-pass approach (with/without change) |
| Insufficient funds with max_inputs | Try largest UTXOs within limit, fall back to send-all |
| Fee estimation doesn't converge | Iterative retry with progressively higher targets (up to 10 attempts) |
| Locktime boundary (499,999,999 vs 500,000,000) | Strict comparison: < 500M = block_height, >= 500M = unix_timestamp |
| RBF with send-all | RBF fields applied regardless of change output presence |
| Multiple payment outputs | All payments included before optional change output |
| Unknown script types in UTXOs | Fall back to p2wpkh weight estimate |
| Malformed fixture JSON | Structured error response with code and message |
| Zero-length hex or odd-length hex | Caught by regex validation |

### Key Challenges

1. **Fee/change feedback loop** was the hardest part. A naive approach of "compute fee, compute change, done" fails on boundary cases where the change is just barely above or below dust. The two-pass approach with iterative convergence handles all edge cases reliably.

2. **Weight estimation accuracy** is critical -- if the weight estimate is too low, the fee rate falls below the target; if too high, the wallet burns sats unnecessarily. I calibrated the per-type weights against Bitcoin Core's estimates.

3. **PSBT serialization** required careful attention to byte ordering (txids reversed for internal representation), CompactSize encoding, and proper separator placement.

---

## 12. Testing Strategy

- **Unit tests**: 15+ tests covering coin selection, fee/change calculation, PSBT structure, RBF/locktime logic, and fixture validation
- **Public fixtures**: All provided fixtures pass the grader (`grade.sh`)
- **All 24 grading fixtures passed** on the automated evaluation
- **Edge case tests**: Created scenarios for dust boundary, send-all, max_inputs enforcement, RBF signaling combinations, and locktime classification

---

## 13. Key Learnings

1. **Fee estimation is a constraint satisfaction problem.** The interdependence between transaction size, fee, and change output presence creates a system that must be solved iteratively, not in a single pass.

2. **UTXO management is a wallet engineering problem.** Coin selection involves tradeoffs between fee minimization, privacy, UTXO consolidation, and change output creation. There is no single "correct" strategy.

3. **BIP-174 (PSBT) is elegant in its simplicity.** The key-value map structure makes it extensible and straightforward to construct, even without a full Bitcoin library.

4. **RBF and locktime interact in non-obvious ways.** The interaction matrix between `rbf`, `locktime`, and `current_height` has five distinct cases, each requiring different nSequence and nLockTime values.

5. **Defensive validation is essential.** Real-world inputs are messy. Validating every field before processing prevents confusing errors deep in the pipeline.

---

*This document was written by Satyam Kumar as part of the Summer of Bitcoin 2026 Developer Challenge.*
