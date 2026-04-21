# Security Checklist for Midnight dApps Before Deployment

> A comprehensive security audit checklist for developers deploying smart contracts and dApps on Midnight Network. This guide covers critical security checks including `disclose()` audits, `ownPublicKey()` vulnerability review, replay protection, and proof generation testing.

**Last Updated:** April 2026  
**Target Audience:** Midnight dApp developers preparing for mainnet deployment  
**Estimated Review Time:** 2-3 hours

---

## Table of Contents

1. [Introduction](#introduction)
2. [Pre-Audit Preparation](#pre-audit-preparation)
3. [disclose() Audit - No Secret Leaks](#disclose-audit---no-secret-leaks)
4. [ownPublicKey() Usage Review](#ownpublickey-usage-review)
5. [Replay Protection Verification](#replay-protection-verification)
6. [Exported Ledger Field Review](#exported-ledger-field-review)
7. [Witness Implementation Correctness](#witness-implementation-correctness)
8. [Version Compatibility Confirmation](#version-compatibility-confirmation)
9. [Proof Generation Testing on Testnet](#proof-generation-testing-on-testnet)
10. [Final Deployment Checklist](#final-deployment-checklist)
11. [Resources](#resources)

---

## Introduction

Deploying a dApp on Midnight Network requires rigorous security auditing. Unlike traditional blockchains, Midnight's privacy-preserving architecture introduces unique security considerations around zero-knowledge proofs, secret management, and witness data.

This checklist is designed to be **actionable** — each item includes specific commands, code patterns, and verification steps. Run through this entire checklist before deploying any contract to mainnet.

### Why This Matters

Security vulnerabilities in privacy-preserving contracts can lead to:
- **Secret leakage** through improper `disclose()` usage
- **Signature vulnerabilities** via `ownPublicKey()` misuse
- **Replay attacks** without proper nullifier management
- **Invalid proofs** causing transaction failures
- **Version mismatches** breaking contract compatibility

---

## Pre-Audit Preparation

Before starting the security audit, ensure your development environment is properly configured.

### 1.1 Environment Setup

```bash
# Verify Midnight CLI version
midnight --version

# Expected: midnight-cli 0.x.x (check docs for latest)

# Verify Node.js version (for TypeScript projects)
node --version

# Expected: v18.x or v20.x LTS
```

### 1.2 Project Structure Check

Ensure your project follows Midnight's recommended structure:

```
my-dapp/
├── contracts/
│   ├── src/
│   │   └── MyContract.zo
│   └── test/
├── frontend/
│   └── src/
├── tests/
│   └── integration/
├── package.json
└── midnight.config.json
```

### 1.3 Dependency Audit

```bash
# Check for outdated dependencies
npm outdated

# Verify Midnight-specific packages
npm list @midnight-ntwrk/*
```

**Action:** Update all dependencies to latest stable versions before audit.

---

## disclose() Audit - No Secret Leaks

The `disclose()` function reveals secret data in zero-knowledge proofs. Improper usage can leak sensitive information.

### 2.1 Identify All disclose() Calls

Search your contract code for all `disclose()` invocations:

```bash
# Search in contract files
grep -rn "disclose(" contracts/src/
```

### 2.2 Classify Each disclose() Usage

For each `disclose()` call, classify into one of these categories:

| Category | Description | Risk Level |
|----------|-------------|------------|
| **Public Data** | Data already on-chain or public | ✅ Low |
| **Computed Results** | Non-sensitive computation outputs | ✅ Low |
| **User-Provided** | Data explicitly provided by user | ⚠️ Medium |
| **Secret State** | Internal contract secrets | ❌ **CRITICAL** |

### 2.3 Audit Checklist

```
□ All disclose() calls are documented with comments
□ No secret state is disclosed without explicit user action
□ disclose() is not used in loops (potential DoS)
□ Disclosed data size is bounded (prevent large proof failures)
□ Error handling exists for disclose() failures
```

### 2.4 Code Review Pattern

**❌ Bad Pattern - Secret Leakage:**
```zo
// DO NOT DO THIS
action processSecret(secret: SecretData) {
    // Accidentally disclosing the entire secret
    disclose(secret);  // LEAKS EVERYTHING!
}
```

**✅ Good Pattern - Minimal Disclosure:**
```zo
// Only disclose what's necessary
action processSecret(secret: SecretData) {
    // Disclose only the proof of validity
    let proof = verifySecret(secret);
    disclose(proof.hash);  // Only the hash, not the secret
}
```

### 2.5 Testing

```bash
# Run contract tests with disclosure tracing
midnight test --trace-disclose

# Check test output for unexpected disclosures
```

---

## ownPublicKey() Usage Review

The `ownPublicKey()` function has known vulnerabilities if misused. This section covers safe usage patterns.

### 3.1 Known Vulnerabilities

**Vulnerability #1: Key Reuse Across Contexts**
Using the same public key for multiple purposes can link otherwise anonymous transactions.

**Vulnerability #2: Timing Attacks**
Public key operations with variable timing can leak information about secret keys.

### 3.2 Audit Checklist

```
□ ownPublicKey() is only used for intended purposes
□ No key reuse across different contract contexts
□ Signature verification uses constant-time comparison
□ Public keys are not logged or disclosed unnecessarily
□ Key derivation follows HD wallet standards (if applicable)
```

### 3.3 Safe Usage Pattern

```zo
// ✅ Correct: Single-purpose key usage
action verifySignature(message: Bytes, signature: Signature) {
    let pk = ownPublicKey();
    // Use only for this specific verification
    assert(verify(pk, message, signature));
}

// ❌ Incorrect: Key reuse across contexts
action badPattern() {
    let pk = ownPublicKey();
    // Using same key for multiple unrelated operations
    log(pk);  // Don't log keys!
    disclose(pk);  // Don't disclose unless necessary!
}
```

### 3.4 Testing

```bash
# Run security-focused tests
midnight test --security-scan

# Check for key usage patterns
midnight audit --key-usage
```

---

## Replay Protection Verification

Replay attacks occur when a valid transaction is maliciously repeated. Midnight contracts must implement replay protection.

### 4.1 Replay Protection Mechanisms

Midnight supports two primary mechanisms:

| Mechanism | Use Case | Implementation |
|-----------|----------|----------------|
| **Nonces** | Sequential transactions | Increment counter per account |
| **Nullifiers** | One-time actions | Mark spent commitments |

### 4.2 Nonce Implementation Audit

```zo
// ✅ Correct nonce pattern
contract SecureContract {
    state nonces: Map<Address, UInt64>;
    
    action transfer(from: Address, to: Address, amount: UInt64, nonce: UInt64) {
        // Verify nonce is exactly next expected value
        assert(nonce == nonces[from]);
        // Increment nonce
        nonces[from] = nonce + 1;
        // ... rest of transfer logic
    }
}
```

### 4.3 Nullifier Implementation Audit

```zo
// ✅ Correct nullifier pattern
contract OneTimeClaim {
    state spentNullifiers: Set<Bytes>;
    
    action claim(nullifier: Bytes, proof: ClaimProof) {
        // Check nullifier hasn't been used
        assert(!spentNullifiers.contains(nullifier));
        // Mark as spent
        spentNullifiers.add(nullifier);
        // ... rest of claim logic
    }
}
```

### 4.4 Audit Checklist

```
□ Every state-changing action has replay protection
□ Nonces are monotonically increasing
□ Nullifiers are checked before state modification
□ Nullifiers are added to spent set atomically
□ Replay protection cannot be bypassed via edge cases
□ Tests cover replay attack scenarios
```

### 4.5 Testing Replay Attacks

```bash
# Run replay attack simulation
midnight test --replay-simulation

# Expected: All replay attempts should fail
```

---

## Exported Ledger Field Review

Exported ledger fields determine what data is visible on-chain. Improper exports can leak sensitive information.

### 5.1 Identify All Exported Fields

```bash
# Find all exported ledger fields
grep -rn "export" contracts/src/
```

### 5.2 Field Classification

For each exported field, classify:

| Classification | Description | Action |
|----------------|-------------|--------|
| **Required** | Needed for consensus/validation | ✅ Keep |
| **Optional** | Useful but not critical | ⚠️ Review |
| **Sensitive** | Could leak information | ❌ Remove/Encrypt |

### 5.3 Audit Checklist

```
□ All exported fields are documented
□ No secret data is in exported fields
□ Exported field sizes are bounded
□ Field names follow naming conventions
□ Exports are minimal (only what's necessary)
```

### 5.4 Example Review

```zo
// ❌ Bad: Exporting too much
export struct UserData {
    address: Address,      // OK
    balance: UInt64,       // OK
    secretKey: Bytes,      // ❌ NEVER export secrets!
    transactionHistory: List<Transaction>,  // ⚠️ Consider privacy
}

// ✅ Good: Minimal exports
export struct UserData {
    address: Address,      // Required for routing
    balanceCommitment: Bytes,  // Commitment, not actual balance
}
```

---

## Witness Implementation Correctness

Witnesses provide private data to zero-knowledge proofs. Incorrect witness handling breaks privacy guarantees.

### 6.1 Witness Data Flow

```
User Input → Witness Generation → Proof Creation → Verification
     ↓              ↓                    ↓              ↓
  Validate    Compute privately    Generate ZKP    Verify on-chain
```

### 6.2 Audit Checklist

```
□ Witness data is never logged
□ Witness generation is deterministic
□ Witness size is bounded (prevent DoS)
□ Invalid witness data is rejected early
□ Witness computation doesn't leak via timing
□ Tests cover edge cases in witness handling
```

### 6.3 Common Witness Mistakes

**❌ Mistake #1: Logging Witness Data**
```zo
// NEVER DO THIS
action process(witness: SecretData) {
    log("Processing witness: {witness}");  // LEAKS TO LOGS!
}
```

**❌ Mistake #2: Unbounded Witness Size**
```zo
// Vulnerable to DoS
action process(witnesses: List<SecretData>) {
    // No size limit - attacker can send 1M witnesses
    for w in witnesses { ... }
}

// ✅ Fixed
action process(witnesses: List<SecretData>) {
    assert(witnesses.len() <= MAX_WITNESSES);  // Bounded
    for w in witnesses { ... }
}
```

### 6.4 Testing

```bash
# Run witness validation tests
midnight test --witness-validation

# Check witness size limits
midnight test --dos-simulation
```

---

## Version Compatibility Confirmation

Version mismatches between contracts, CLI, and network can cause deployment failures or unexpected behavior.

### 7.1 Version Matrix

Create a version matrix for your project:

| Component | Your Version | Required Version | Status |
|-----------|--------------|------------------|--------|
| Midnight CLI | `0.x.x` | `>=0.x.x` | ✅/❌ |
| Contract Language | `x.x` | `x.x` | ✅/❌ |
| SDK/ Libraries | `x.x.x` | `>=x.x.x` | ✅/❌ |
| Target Network | Testnet/Mainnet | - | ✅/❌ |

### 7.2 Compatibility Check

```bash
# Check CLI compatibility
midnight doctor

# Check contract compilation
midnight compile --check-versions

# Verify network compatibility
midnight network status
```

### 7.3 Audit Checklist

```
□ Contract language version is pinned
□ All dependencies use compatible versions
□ Target network is correctly configured
□ Migration scripts are version-aware
□ Rollback plan exists for version issues
```

---

## Proof Generation Testing on Testnet

Never deploy to mainnet without thorough testnet testing. This section covers comprehensive testnet validation.

### 8.1 Testnet Deployment Checklist

```
□ Contract compiles without warnings
□ All unit tests pass
□ Integration tests pass on testnet
□ Proof generation completes in <30 seconds
□ Proof verification succeeds on-chain
□ Gas/fee estimation is accurate
□ Error messages are informative
```

### 8.2 Proof Generation Testing

```bash
# Generate proof on testnet
midnight prove --network testnet --contract MyContract

# Expected output:
# ✓ Proof generated successfully
# ✓ Proof size: X bytes (within limits)
# ✓ Generation time: Y seconds (<30s)
```

### 8.3 Proof Verification Testing

```bash
# Verify proof on testnet
midnight verify --network testnet --proof proof.json

# Expected: Verification succeeds
```

### 8.4 Load Testing

```bash
# Simulate multiple concurrent proofs
midnight load-test --concurrent 10 --duration 60s

# Expected: All proofs generate and verify successfully
```

### 8.5 Edge Case Testing

Test these edge cases on testnet:

| Edge Case | Expected Behavior |
|-----------|-------------------|
| Empty witness | Rejected with clear error |
| Maximum-size witness | Succeeds within time limit |
| Invalid signature | Rejected before proof generation |
| Duplicate nullifier | Rejected as replay |
| Network timeout | Graceful retry/failure |

---

## Final Deployment Checklist

Before deploying to mainnet, run through this final checklist.

### 9.1 Pre-Deployment

```
□ All security audit items completed
□ All tests passing (unit, integration, security)
□ Code review completed by at least one other developer
□ Testnet deployment successful for 7+ days
□ No critical bugs found in testnet monitoring
□ Documentation is complete and accurate
□ Emergency pause mechanism is implemented
□ Rollback plan is documented and tested
```

### 9.2 Deployment

```bash
# Deploy to mainnet
midnight deploy --network mainnet --contract MyContract

# Verify deployment
midnight verify-deployment --address <CONTRACT_ADDRESS>
```

### 9.3 Post-Deployment

```
□ Contract address is verified on block explorer
□ Initial state is correct
□ First transaction succeeds
□ Monitoring/alerting is configured
□ Team is notified of deployment
□ Documentation is updated with deployment details
```

---

## Resources

### Official Documentation

- [Midnight Developer Docs](https://docs.midnight.network/getting-started)
- [Midnight MCP Package](https://www.npmjs.com/package/midnight-mcp)
- [Technical Style Guide](https://docs.google.com/document/d/1Srfgnn5Utp6fDXxJknwpQUd6l5TQNR9k_BJjtjAmm_s/edit)

### Community Resources

- [Developer Forum](https://forum.midnight.network/)
- [Discord Server](https://discord.com/invite/midnightnetwork)
- [GitHub Issues](https://github.com/midnightntwrk)

### Security Tools

```bash
# Midnight security scanner
midnight audit --full

# Dependency vulnerability check
npm audit

# Static analysis
midnight analyze contracts/src/
```

---

## Conclusion

Security is not a one-time checklist — it's an ongoing process. Run this audit:

- ✅ Before every mainnet deployment
- ✅ After significant code changes
- ✅ Quarterly for existing deployments
- ✅ When new vulnerabilities are disclosed

**Remember:** The cost of a security audit is always less than the cost of a security breach.

---

*This tutorial was created as part of the Midnight Network Eclipse Bounty program. For questions or feedback, please open an issue on the [contributor-hub repository](https://github.com/midnightntwrk/contributor-hub).*

**Tags:** #MidnightforDevs #Security #ZeroKnowledge #SmartContracts #Blockchain
