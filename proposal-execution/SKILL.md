---
name: realms-proposal-execution
description: Build and attach executable instructions to Realms governance proposals. Supports 40+ instruction types — token transfers, burns, Jupiter swaps/DCA/limit orders, Kamino lending, Drift perp trading, validator staking, token streaming/locking, council member management, DAO config changes, treasury operations, and custom arbitrary instructions. Use when building executable proposals that perform on-chain actions through DAO governance. Triggers on "transfer tokens", "send from treasury", "swap from DAO", "Jupiter limit order", "DCA proposal", "Kamino deposit", "Drift trade", "stake to validator", "stream tokens", "burn tokens", "custom instruction", "governance instruction", "executable proposal".
---

# Realms Proposal Execution

Attach on-chain instructions to governance proposals. When a proposal passes and is executed, these instructions run from the DAO treasury.

## How It Works

1. Build instructions as `TransactionInstruction` objects
2. Serialize with `serializeInstructionToBase64()` from `@realms-today/spl-governance`
3. Pass to `withInsertTransaction()` during proposal creation
4. Each instruction is inserted as a separate proposal transaction
5. When executed, the governance authority (treasury) signs as the authority

## Key Concept: Governance Authority

The treasury wallet (`getNativeTreasuryAddress(governancePk)`) acts as the signer/authority for all instructions. When building instructions, use the **treasury address** as the authority — NOT the proposer's wallet.

## Instruction Types

### Treasury Operations

| Type | Description |
|------|-------------|
| **Send Tokens** | Transfer SPL tokens or SOL from treasury |
| **Bulk Send** | Transfer multiple token types in one proposal |
| **Burn Tokens** | Burn SPL tokens from treasury |
| **Close Token Account** | Close empty token accounts, reclaim SOL |
| **Wrap SOL** | Wrap native SOL to wSOL in treasury |

### DeFi Integrations

| Type | Description |
|------|-------------|
| **Jupiter Limit Order** | Place limit orders (Trigger API) |
| **Cancel Jupiter Limit Order** | Cancel existing limit orders |
| **Jupiter DCA** | Set up recurring/DCA orders |
| **Cancel Jupiter DCA** | Cancel recurring orders |
| **Withdraw Jupiter DCA** | Withdraw accumulated tokens |
| **Kamino Lend** | Deposit into Kamino lending reserves |
| **Kamino Withdraw** | Withdraw from Kamino lending |
| **Drift Init Account** | Initialize Drift trading account |
| **Drift Deposit** | Deposit collateral to Drift |
| **Drift Perp Trade** | Open perp positions on Drift |
| **Drift Withdraw** | Withdraw from Drift |
| **Drift Close Position** | Close Drift perp positions |
| **Drift Cancel Orders** | Cancel Drift orders |
| **Manifest Limit Order** | Place orders on Manifest DEX |
| **DFlow Prediction Market** | Buy prediction market positions |

### Staking

| Type | Description |
|------|-------------|
| **Stake to Validator** | Delegate SOL to a validator |
| **Deactivate Validator Stake** | Begin stake deactivation |
| **Withdraw Validator Stake** | Withdraw deactivated stake |
| **Stake with Save** | Deposit/withdraw via Save Finance |

### Token Streaming & Vesting

| Type | Description |
|------|-------------|
| **Stream Tokens** | Create token payment streams |
| **Lock Tokens** | Lock tokens with cliff vesting |
| **Cancel Stream** | Cancel active token streams |
| **Cancel Lock** | Cancel token locks |

### Governance Management

| Type | Description |
|------|-------------|
| **Update Council Members** | Add/remove council members |
| **DAO Config** | Change governance parameters |
| **Edit Wallet Rules** | Modify governance thresholds, voting time, quorum |
| **Add Metadata** | Set DAO name, description, image, links |
| **Set Treasury Name** | Assign .sol domain to treasury |
| **VSR Enable** | Enable Voter Stake Registry plugin |
| **VSR Grant** | Grant locked tokens via VSR |
| **VSR Clawback** | Clawback unvested VSR grants |

### Distribution

| Type | Description |
|------|-------------|
| **Init Distribution** | Create token distribution/dissolution |
| **Add Distribution Tokens** | Fund a distribution |
| **Withdraw from Collection** | Withdraw from distribution vault |

### Custom

| Type | Description |
|------|-------------|
| **Custom Instruction** | Any arbitrary Solana instruction (base64 encoded) |

## Common Instruction Patterns

For detailed implementation of each instruction type with code examples: See [references/instructions.md](references/instructions.md)

For the serialization and proposal insertion flow: See [references/serialization.md](references/serialization.md)
