# zk-pow

Zero-knowledge Proof-of-Work library for the Pearl blockchain. Implements the PoUW (Proof-of-Useful-Work) protocol where GPU matrix multiplications serve as both AI compute and blockchain mining work.

## Overview

Pearl's mining protocol requires miners to perform matrix multiply-accumulate (MatMul) operations on integer matrices. The `zk-pow` crate provides:

- **Proof generation** - produce a ZK proof that a valid MatMul was performed over committed matrices
- **Proof verification** - verify a ZK proof against a block header and mining configuration
- **Circuit compilation** - build and cache Plonky2/STARKy circuits for fast repeated verification
- **FFI bindings** - Go (via C FFI) and Python (via PyO3) interfaces

## Architecture

```
zk-pow/
src/
  api/           # Public types and entry points
    proof.rs         # Core data structures (IncompleteBlockHeader, ZKProof, MMAType)
    prove.rs         # Proof generation
    verify.rs        # Proof verification
    proof_utils.rs   # Jackpot hash, public param compilation
    sanity_checks.rs # Difficulty and parameter validation
  circuit/       # Plonky2 / STARKy circuit definitions
    pearl_circuit.rs # Recursive SNARK circuit
    pearl_stark.rs   # STARK component
    pearl_noise.rs   # Low-rank noise computation
    pearl_layout.rs  # Circuit layout macro
    circuit_utils.rs # CircuitCache
  ffi/           # Foreign function interface
    mine.rs          # Mining helpers
    plain_proof.rs   # PlainProof (pre-ZK intermediate)
    pybind.rs        # PyO3 Python bindings
  bin/
    build_cache.rs   # CLI: pre-compile and serialise circuit cache
    prove_verify.rs  # CLI: end-to-end prove + verify smoke test
bindings/
  go/            # cbindgen-generated Go FFI layer
```

## Key Concepts

### IncompleteBlockHeader

The block header fields committed to before mining begins: version, prev_block, merkle_root, timestamp, and nbits. The miner fills in the proof fields to complete the header.

### MiningConfiguration

Parameters committed to before mining: matrix dimensions (m, n, k), noise rank, MMA type, and row/column periodic patterns. Serialized as 52 bytes little-endian.

### PlainProof to ZKProof Pipeline

1. Miner performs `(A + E) * (B + F)` MatMul using committed noise matrices
2. A `PlainProof` is produced containing Merkle-opened matrix rows/columns
3. `PlainProof` is converted to a compact `ZKProof` (60 KB or less) via Plonky2 recursive SNARKs
4. `ZKProof` is embedded in the block certificate and verified by all nodes

### Circuit Cache

Plonky2 circuits are expensive to compile. The `build_cache` binary pre-compiles 13 first-level and 2 second-level circuits and serialises them to `src/circuit/cache.bin`. At runtime the verifier loads this cache for instant verification.

## Building

```bash
# Build the library
cargo build --release

# Pre-compile and cache circuits (required before verification)
cargo run --release --bin build_cache src/circuit/cache.bin

# Run smoke test (prove + verify)
cargo run --release --bin prove_verify

# Build with embedded circuit cache
cargo build --release --features embedded_cache

# Build with Python bindings
cargo build --release --features pyo3
```

## Feature Flags

| Feature | Description |
|---|---|
| `embedded_cache` | Embed the compiled circuit cache binary directly into the library |
| `pyo3` | Enable Python bindings via PyO3 |

## Go FFI

The `bindings/go/` directory contains a cbindgen-generated C header and Go wrapper. Build with:

```bash
task build:zk-gobind
```

The Go node uses this to verify ZK certificates during block validation.

## Testing

```bash
cargo test
```

Note: tests require the circuit cache to be built first (`cargo run --release --bin build_cache`).
