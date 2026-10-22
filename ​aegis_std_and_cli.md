# Aegis-S Standard Library & CLI Compiler Driver (`aegisc`)

## 1. Overview of the CLI Driver (`aegisc`)
The Aegis-S compiler toolchain is controlled via a unified command-line interface named **`aegisc`**. Unlike traditional compilers that expose raw optimization flags, `aegisc` enforces strict security profiles, automated SMT verification checks, adversarial red-team fuzzing passes, and quantum-resistant binary emission by default.

---

## 2. CLI Command Reference

### A. Building a Module
To compile an Aegis-S source file into an enclaved, lattice-signed binary:
```bash
aegisc build vault.aegis --target x86_64-enclave-secure --release
```

**Under-the-Hood Execution Flow:**
1. **Frontend Phase (`aegis-parser`):** Tokenizes and parses `.aegis` syntax, applying lexical entropy mapping.
2. **SMT Verification Phase (`aegis-smt`):** Translates all `requires` and `ensures` contracts into Z3 verification conditions and runs formal proofs.
3. **Adversarial Fuzzing Phase (`RedTeam-NN`):** Deploys internal AI agents to simulate zero-day exploits and check for side-channel timing leaks.
4. **Binary Emission Phase (`aegis-emission`):** Injects ZKRI memory-wipe headers and signs the final machine code using CRYSTALS-Dilithium lattice-based cryptography.

### B. Running Security Audits Without Emission
If developers want to run formal verification and red-team fuzzing checks without producing a binary (ideal for CI/CD pipelines):
```bash
aegisc audit vault.aegis --strict-entropy
```

---

## 3. Aegis-S Standard Library Specification

The Aegis-S standard library (`std`) provides core cryptographic primitives and memory-safe container types built directly on top of the ZKRI runtime environment.

### A. Module: `Crypto.Homomorphic`
Provides types and functions for performing arithmetic operations on encrypted ciphertexts without decrypting them.
```aegis
module Crypto.Homomorphic;

pub struct Ciphertext<T> {
    payload: Bytes<512>,
    proof: ZkProof
}

pub func add_encrypted(a: Ciphertext<Decimal>, b: Ciphertext<Decimal>) -> Ciphertext<Decimal>
    requires a.proof.is_valid()
    requires b.proof.is_valid()
    ensures result.proof.is_valid()
{
    isolate {
        // Hardware-accelerated homomorphic addition
        let raw_sum = native_homomorphic_add(a.payload, b.payload);
        return Ciphertext { payload: raw_sum, proof: generate_zk_proof(raw_sum) };
    }
}
```

### B. Module: `Verify.Strict`
Provides formal verification assertion helpers and bounded type constraints.
```aegis
module Verify.Strict;

pub type ProtectedDecimal = Decimal<Protected, 0..10_000_000>;
pub type EncryptedUUID = UUID<Encrypted, ZeroizeOnDrop>;
```

### C. Module: `Secure.Memory`
Enforces hardware memory sanitization and zeroization policies.
```aegis
module Secure.Memory;

pub trait ZeroizeOnDrop {
    pub func purge(&mut self);
}

impl ZeroizeOnDrop for SecretKey {
    pub func purge(&mut self) {
        isolate {
            hardware_overwrite_ram(&mut self.bytes, 0x00);
        }
    }
}
```

---

## 4. Project Configuration File (`aegis.toml`)
Every Aegis-S project is configured using an `aegis.toml` manifest file located in the root directory:

```toml
[package]
name = "secure_vault_system"
version = "0.1.0"
authors = ["Aegis Security Architect"]

[dependencies]
crypto-homomorphic = "1.0"
verify-strict = "1.2"

[compiler.profile.release]
entropy_level = "maximum"
enable_redteam_nn = true
smt_timeout = 10000
enforce_zkri_hardware_enclave = true
```
