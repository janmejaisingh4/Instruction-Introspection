# Dice Game (Solana Anchor)

## Project Overview

This repository contains a Solana Anchor program that implements a deterministic dice betting game. The on-chain program manages a house vault and individual player bets through PDAs, resolves outcomes using cryptographic entropy or Ed25519-signed messages, and enforces payout rules with a house edge.

The project includes:
- Anchor program in `programs/dice-game`
- TypeScript integration tests in `tests/src/dice-game.ts`
- Localnet deployment settings in `Anchor.toml`

## Key Concepts

- `initialize`: Funds a house vault PDA and prepares the program state.
- `place_bet`: Creates a bet account PDA and deposits the player stake into the house vault.
- `resolve_bet`: Resolves a bet using an Ed25519 signature from the house key.
- `commit_entropy` / `resolve_from_entropy`: Supports entropy-based resolution for deterministic, verifiable outcomes.
- `Bet` account: Stores player, seed, slot, amount, roll target, and bump seed values.

## Architecture

1. House Vault PDA
   - Seed: `["vault", house.key()]`
   - Stores deposited SOL required to pay winning bets.

2. Bet PDA
   - Seed: `["bet", vault.key(), player.key(), seed]`
   - Stores the individual bet parameters and is closed after resolution.

3. Outcome Resolution
   - `resolve_bet`: Uses Ed25519 instruction verification from the instruction sysvar.
   - `resolve_from_entropy`: Uses on-chain committed entropy and bet metadata to derive a random roll.

4. Payout Calculation
   - Winning probability is derived from the target roll.
   - House edge is fixed at `150 bps` (1.50%).
   - Payout formula matches the on-chain logic to ensure consistency with client-side expectations.

## Prerequisites

- Rust toolchain
- Solana CLI
- Anchor CLI
- Node.js and npm
- Local Solana cluster (`solana-test-validator` or Anchor localnet)

## Setup

1. Install Anchor and required dependencies:

```bash
cargo install --git https://github.com/coral-xyz/anchor anchor-cli --locked
npm install
```

2. Ensure the Solana CLI is configured for localnet and a funded wallet:

```bash
solana config set --url localhost
solana airdrop 2
```

3. Verify `Anchor.toml` points to `Localnet` and the correct wallet path.

## Build and Deploy

1. Build the Anchor program:

```bash
anchor build
```

2. Deploy the program locally:

```bash
anchor deploy
```

3. Confirm the generated IDL and client types in `target/types/dice_game.ts`.

## Testing

The repository provides automated integration tests that exercise both signature-based and entropy-based bet resolution.

Run tests with:

```bash
anchor test
```

The test suite performs the following scenarios:
- House and player airdrops
- Vault initialization
- Placing a bet
- Resolving a bet with Ed25519 signature verification
- Resolving a bet from committed entropy

## Instruction Flow

### Initialization

- `initialize(amount)`
- Transfers `amount` from the house signer into the vault PDA.
- Creates the house vault PDA using a static seed.

### Place Bet

- `place_bet(seed, amount, roll)`
- Validates minimum bet size and roll bounds.
- Creates the bet PDA and transfers the bet amount into the vault.

### Resolve Bet

- `resolve_bet(sig)`
- Reads the previous instruction from `SYSVAR_INSTRUCTIONS_PUBKEY`.
- Verifies the Ed25519 signature by comparing public key, signature, and bet message.
- Hashes the signature with SHA-256 to derive a pseudo-random roll.
- Pays out winners from the vault via CPI.

### Resolve From Entropy

- `commit_entropy(entropy)`
- `resolve_from_entropy()`
- Commits a 32-byte entropy value in a separate instruction.
- Uses the committed entropy plus bet PDA metadata to derive a deterministic roll.
- Enables verifiable entropy-based resolution for games not relying on the house key.

## Code Structure

- `programs/dice-game/src/lib.rs`
  - Entrypoint and program instruction definitions.

- `programs/dice-game/src/instructions/`
  - `intialize.rs`
  - `place_bet.rs`
  - `resolve_bet.rs`
  - `commit_entropy.rs`
  - `resolve_from_entropy.rs`

- `programs/dice-game/src/state/mod.rs`
  - `Bet` account layout and serialization logic.

- `programs/dice-game/src/error.rs`
  - Program error definitions and validation failure messages.

- `tests/src/dice-game.ts`
  - TypeScript test scenarios, transaction flows, and payout validation.

## Professional Notes

- The bet resolution flow uses deterministic hash-based randomness derived from the submitted signature or entropy.
- The `Bet` account is closed after resolution and returns remaining rent-exempt SOL to the player.
- The contract enforces `MIN_BET_LAMPORTS` and roll boundaries to avoid invalid state.
- The explicit verification of instruction sysvar contents ensures the contract only resolves legitimate on-chain proofs.

## Recommended Next Steps

- Add client-side UI integration for submitting bets and displaying outcomes.
- Implement additional safety checks for vault balance and replay protection.
- Extend entropy commitment to support external randomness or verifiable randomness oracles.

---

For a modern Solana project, maintain the separation of account logic, instruction validation, and client test flows shown in this repository.