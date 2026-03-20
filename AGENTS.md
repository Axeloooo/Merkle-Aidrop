# AGENTS.md

This file provides guidance to AI Agents when working with code in this repository.

## Commands

```bash
# Build
forge build

# Test
forge test          # Run all tests
forge test -vvv     # Verbose output
forge test --match-test <testName>  # Run a single test

# Format
forge fmt           # Format code
forge fmt --check   # Check formatting (used in CI)

# Gas snapshots
forge snapshot

# Generate Merkle input data (whitelist → input.json)
forge script script/GenerateInput.s.sol

# Generate Merkle tree and proofs (input.json → output.json)
forge script script/MakeMerkle.s.sol
```

## Architecture

This is a Merkle tree-based ERC20 airdrop system built with Foundry.

### Contracts (`src/`)

- **BagelToken.sol** — Owner-mintable/burnable ERC20 token (name: "BagelToken", symbol: "BAGEL") used as the airdrop token.
- **MerkleAirdrop.sol** — Core contract. Inherits from OpenZeppelin `EIP712` (domain: `"MerkleAirdrop"`, version: `"1"`). Stores an immutable Merkle root at deployment. Users call `claim(address account, uint256 amount, bytes32[] proof, uint8 v, bytes32 r, bytes32 s)` which validates both the Merkle proof and an EIP712 ECDSA signature, marks the address as claimed (preventing double-claims), and transfers tokens. Custom errors: `InvalidProof`, `AlreadyClaimed`, `MerkleAirdrop__InvalidSignature`.

#### EIP712 / ECDSA Signature Flow

The contract requires claimants to sign an EIP712-structured message before anyone can submit their claim. This enables meta-transactions: a third party (`gasPayer`) can call `claim()` on behalf of `user`, as long as `user` has signed the message.

**Type hash & struct:**
```solidity
bytes32 private constant MESSAGE_TYPEHASH = keccak256("AirdropClaim(address account,uint256 amount)");

struct AirdropClaim {
    address account;
    uint256 amount;
}
```

**Generating the digest (off-chain or in tests):**
```solidity
bytes32 digest = merkleAirdrop.getMessageHash(account, amount);
(uint8 v, bytes32 r, bytes32 s) = vm.sign(userPrivKey, digest); // Foundry cheatcode
```

**Signature verification (inside `claim()`):**
```solidity
// Recovers signer via ECDSA.tryRecover and asserts signer == account
if (!_isValidSignature(account, getMessageHash(account, amount), v, r, s)) {
    revert MerkleAirdrop__InvalidSignature();
}
```

`getMessageHash()` is a public `view` function that wraps `_hashTypedDataV4()`, so front-ends and tests can easily produce the correct EIP712 digest.

### Scripts (`script/`)

- **GenerateInput.s.sol** — Hardcodes 4 whitelisted addresses (each eligible for 25 BAGEL) and writes `script/target/input.json`.
- **MakeMerkle.s.sol** — Reads `input.json`, uses the Murky library to construct the Merkle tree, and writes the root + per-address proofs to `script/target/output.json`.

### Data Flow

1. `GenerateInput.s.sol` → `script/target/input.json` (whitelist)
2. `MakeMerkle.s.sol` → `script/target/output.json` (Merkle root + proofs)
3. Deploy `MerkleAirdrop` with the Merkle root from step 2
4. Users call `claim()` with their proof from the output JSON

### Dependencies (git submodules in `lib/`)

- `forge-std` — Foundry testing framework
- `openzeppelin-contracts` — ERC20 base and utilities
- `murky` — Merkle tree construction library

The Merkle leaf encoding (in `MerkleAirdrop.sol`) must stay consistent with how leaves are built in `MakeMerkle.s.sol` — both hash `abi.encodePacked(address, uint256)`.
