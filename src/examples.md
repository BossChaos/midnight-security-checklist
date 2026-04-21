# Midnight Security Checklist - Code Examples

This repository contains code examples demonstrating secure patterns for Midnight dApp development.

## Examples

### 1. Secure disclose() Pattern

```zo
// ✅ Good: Minimal disclosure
action processSecret(secret: SecretData) {
    // Only disclose the proof of validity, not the secret itself
    let proof = verifySecret(secret);
    disclose(proof.hash);
}
```

### 2. Safe ownPublicKey() Usage

```zo
// ✅ Good: Single-purpose key usage
action verifySignature(message: Bytes, signature: Signature) {
    let pk = ownPublicKey();
    assert(verify(pk, message, signature));
}
```

### 3. Nonce-Based Replay Protection

```zo
contract SecureContract {
    state nonces: Map<Address, UInt64>;
    
    action transfer(from: Address, to: Address, amount: UInt64, nonce: UInt64) {
        assert(nonce == nonces[from]);
        nonces[from] = nonce + 1;
        // ... transfer logic
    }
}
```

### 4. Nullifier-Based Replay Protection

```zo
contract OneTimeClaim {
    state spentNullifiers: Set<Bytes>;
    
    action claim(nullifier: Bytes, proof: ClaimProof) {
        assert(!spentNullifiers.contains(nullifier));
        spentNullifiers.add(nullifier);
        // ... claim logic
    }
}
```

### 5. Bounded Witness Handling

```zo
const MAX_WITNESSES = 100;

action process(witnesses: List<SecretData>) {
    assert(witnesses.len() <= MAX_WITNESSES);
    for w in witnesses {
        // Process each witness
    }
}
```

## Full Tutorial

See the complete security checklist tutorial: [security-checklist.md](./security-checklist.md)

## License

MIT License - See LICENSE file for details.

---

*Created for Midnight Network Eclipse Bounty Program*
