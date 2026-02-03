# Moltbot Configuration for Realms

Quick setup guide for configuring Moltbot to interact with Realms Governance and Launchpad.

---

## Required Environment

```bash
# Solana RPC endpoint
SOLANA_RPC_URL=https://api.mainnet-beta.solana.com

# Wallet (base58 encoded private key or path to keypair file)
SOLANA_WALLET_KEY=<your-private-key>
# OR
SOLANA_WALLET_PATH=~/.config/solana/id.json

# Cluster
SOLANA_CLUSTER=mainnet-beta  # or devnet
```

---

## Program IDs

```yaml
governance:
  program: GovER5Lthms3bLBqWub97yVrMmEogzX7xNjdXpPPCVZw
  api: https://v2.realms.today/api/v1

launchpad:
  program: ReaLM68X8dXLz35oXqofDYkNWiBFZr4FcefSJyTr9Yh
  metadata: https://v2.realms.today/launchpad/metadata.json

tokens:
  usdc: EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
```

---

## Skill Files

| Skill | File | Purpose |
|-------|------|---------|
| Governance | `docs/skill.md` | DAO management, proposals, voting |
| Launchpad | `docs/launchpad-skill.md` | Fundraising, ICO, token claims |

---

## Quick Commands

### Check Wallet Balance
```bash
solana balance <wallet-address>
```

### Check USDC Balance
```bash
spl-token balance EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
```

---

## Launchpad Actions (Short Reference)

### Invest in Pool
```
Pool ID: <quote-mint-address>
Amount: <usdc-amount>
Action: invest

Prerequisites:
- Pool status = Active
- Time between startTime and endTime
- Have USDC balance >= amount
```

### Claim Tokens/Refund
```
Receipt: <receipt-pda>
Action: claim

Prerequisites:
- Pool status != Active
- Receipt not already claimed
```

### List Active Pools
```
Filter: status = Active
Sort: endTime ascending (ending soon first)
```

---

## Governance Actions (Short Reference)

### Vote on Proposal
```
DAO: <realm-pk>
Proposal: <proposal-pk>
Vote: 0=Yes, 1=No, 2=Abstain, 3=Veto
Action: POST /daos/{realmPk}/proposals/{proposalPk}/vote
```

### Create Proposal
```
DAO: <realm-pk>
Governance: <governance-pk>
Title: <name>
Description: <text-or-url>
Action: POST /daos/{realmPk}/proposals/create
```

### Check Membership
```
DAO: <realm-pk>
Wallet: <wallet-pk>
Action: GET /daos/{realmPk}/members/{walletPk}/voting-power
```

---

## Fee Summary

| Action | Fee | Token |
|--------|-----|-------|
| Invest in pool | ~0.01 | SOL (tx fee) |
| Claim tokens | ~0.01 | SOL (tx fee) |
| Create DAO | 2.0 | SOL |
| First vote/join | 0.1 | SOL |
| Create proposal | 0-0.5 | SOL |

---

## Safety Settings

```yaml
# Recommended agent constraints
max_investment_per_pool: 1000  # USDC
require_user_confirmation: true
check_geo_restrictions: true
min_pool_progress_to_invest: 10  # % of minimum reached
max_treasury_spend_percent: 10  # Flag proposals > 10% treasury
```

---

## Dependencies

```bash
npm install @solana/web3.js @solana/spl-token @coral-xyz/anchor bn.js bs58
```

---

## Test on Devnet First

```bash
# Switch to devnet
export SOLANA_CLUSTER=devnet
export SOLANA_RPC_URL=https://api.devnet.solana.com

# Airdrop SOL for testing
solana airdrop 2

# Use devnet USDC faucet for testing investments
```
