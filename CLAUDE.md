# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

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
- **MerkleAirdrop.sol** — Core contract. Stores an immutable Merkle root at deployment. Users call `claim(address account, uint256 amount, bytes32[] proof)` which validates the Merkle proof, marks the address as claimed (preventing double-claims), and transfers tokens. Custom errors: `InvalidProof`, `AlreadyClaimed`.

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
