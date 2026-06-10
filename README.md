# Signature Attacks Workshop — Vulnerable vs Secure

> A hands-on exploration of one of the most common (and most dangerous) signature-handling bugs in Solidity: failing to validate the return value of `ecrecover`.

[![Solidity](https://img.shields.io/badge/Solidity-0.8.24-blue)](https://soliditylang.org)
[![Foundry](https://img.shields.io/badge/Built%20with-Foundry-red)](https://book.getfoundry.sh)
[![Tests](https://img.shields.io/badge/Tests-12%2F12%20passing-brightgreen)](#tests)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 🎯 The vulnerability

In Solidity, `ecrecover(hash, v, r, s)` is the standard way of recovering the signer of an Ethereum-signed message. But when given invalid parameters (bad `v`, `r`, or `s`), it **returns `address(0)` instead of reverting**.

If you don't validate that the recovered address is non-zero before treating it as authoritative, an attacker can pass a bogus signature and your contract will happily accept it as if the zero address had signed it.

```solidity
// ❌ VULNERABLE
function recoverSigner(uint8 v, bytes32 r, bytes32 s, bytes32 hash) external {
    address signer = ecrecover(hash, v, r, s);
    // No check: signer may be address(0) → authorization can be bypassed
}
```

```solidity
// ✅ SECURE
function recoverSigner(uint8 v, bytes32 r, bytes32 s, bytes32 hash) external {
    address signer = ecrecover(hash, v, r, s);
    require(signer != address(0), "Invalid signature");
    // ... use signer
}
```

---

## 📂 Project structure

```
src/
├── SignatureAttacks.sol          # Vulnerable contract (DO NOT USE in prod)
└── SecureSignatureContract.sol   # Hardened contract with proper validation
test/
├── SignatureAttacks.t.sol        # 6 tests demonstrating the attack
└── SecureSignatureAttacks.t.sol  # 6 tests proving the fix
```

---

## 🧪 The attack in action

The `SignatureAttacks.authorizeUser` function lets the operator mark an address as authorized by submitting a signed payload. The vulnerability: when given a junk signature, `ecrecover` returns `address(0)`, and the contract treats the `address(0)` payload as if the user with id `0` had signed it — a classic authorization bypass.

### Attack steps

1. Attacker calls `recoverSigner` with arbitrary `v`, `r`, `s` values
2. `ecrecover` returns `address(0)`
3. The contract uses this zero address as the signer without validation
4. Any authorization tied to "the signer of this message" can be subverted

### The fix (in `SecureSignatureContract.sol`)

```solidity
address signer = ecrecover(hash, v, r, s);
require(signer != address(0), "Invalid signature");
require(signer == expectedSigner, "Unauthorized signer");
```

---

## 🛠 Run the tests

```bash
bash install.sh    # only first time — installs forge-std + openzeppelin
forge build
forge test -vv
```

> Don't have Foundry installed? [Install it first](https://book.getfoundry.sh/getting-started/installation).

Expected output:

```
Ran 2 tests for test/SignatureAttacks.t.sol:SignatureAttacksTest
[PASS] testAuthorizeVulnerable()      (gas: ~50k)
[PASS] testRecoverZeroAddress()       (gas: ~30k)
[PASS] testAttackScenario()           (gas: ~70k)
... 6 passing

Ran 2 tests for test/SecureSignatureAttacks.t.sol:SecureSignatureAttacksTest
[PASS] testRevertOnInvalidSignature() (gas: ~25k)
[PASS] testValidSignatureAuthorized() (gas: ~95k)
... 6 passing
```

---

## 🔍 What this project teaches

- The `ecrecover` footgun and why it matters in real audits
- The defensive pattern: `require(signer != address(0), ...)`
- How to write a Foundry test that demonstrates the exploit
- How to write a Foundry test that proves the fix holds
- Why the same `recoverSigner` function can be safe or dangerous depending on input validation

---

## 📚 Related reading

- [SWC Registry: SWC-117 Signature Malleability](https://swcregistry.io/docs/SWC-117)
- [OpenZeppelin ECDSA library](https://docs.openzeppelin.com/contracts/5.x/api/utils#ECDSA)
- [Ethernaut Level 13: Sign / ecrecover](https://ethernaut.openzeppelin.com/level/13)

---

## 📜 License

MIT — see [LICENSE](LICENSE)

---

**Author:** Santiago López Castaño · [@santilpz28](https://github.com/santilpz28)
Built as part of the **Master in Blockchain Development** (2026) — vulnerability class workshop.
