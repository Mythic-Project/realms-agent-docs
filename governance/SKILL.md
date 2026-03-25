---
name: realms-governance
description: Manage Solana DAOs through the Realms Governance REST API. Create DAOs, proposals, vote, delegate, execute passed proposals, query treasury and membership. Use when interacting with SPL Governance DAOs on Solana — creating/managing organizations, voting on proposals, checking treasury balances, or automating governance workflows. Triggers on "DAO", "governance", "proposal", "vote on", "create DAO", "treasury", "Realms".
---

# Realms Governance

Interact with Solana DAOs via the Realms Governance REST API (`https://v2.realms.today/api/v1`).

## Configuration

```yaml
base_url: https://v2.realms.today/api/v1
program: GovER5Lthms3bLBqWub97yVrMmEogzX7xNjdXpPPCVZw
cluster: mainnet-beta
rate_limit: 50/minute
```

## Critical Rules

1. **NEVER vote blindly** — analyze proposal content, current votes, and treasury impact first
2. **Check membership** before voting — call `GET /daos/{realmPk}/members/{walletPk}/voting-power`
3. **Check permissions** before creating proposals — verify token thresholds via `GET /daos/{realmPk}/governances`
4. **Submit transactions sequentially** — POST endpoints return unsigned transactions; sign and send in order
5. `"18446744073709551615"` (u64::MAX) = DISABLED; `{ "type": 2 }` = feature disabled

## Before Any Write Operation

```
1. Check wallet SOL balance (need for fees)
2. GET /daos/{realmPk}/members/{walletPk}/voting-power → verify membership
3. GET /daos/{realmPk}/governances → verify permissions/thresholds
4. Execute the operation
5. Sign and submit transactions in order
```

## Core Workflows

### Vote on a Proposal

```
1. GET /daos/{realmPk}/proposals/{proposalPk} → state must be 2 (Voting)
2. GET /daos/{realmPk}/members/{walletPk}/voting-power → verify membership
3. GET /daos/{realmPk}/proposals/{proposalPk}/votes → analyze sentiment
4. POST /daos/{realmPk}/proposals/{proposalPk}/vote
   { "walletPublicKey": "...", "voteKind": 0 }
   voteKind: 0=Approve, 1=Deny, 2=Abstain, 3=Veto
5. Sign and submit returned transaction
```

### Create a Proposal

```
1. GET /daos/{realmPk}/members/{walletPk}/voting-power → get depositAmount
2. GET /daos/{realmPk}/governances → get minTokensToCreateProposal
3. Verify: depositAmount >= threshold (if threshold = u64::MAX, cannot create)
4. POST /daos/{realmPk}/proposals/create
   { "walletPublicKey": "...", "governancePk": "...", "name": "...",
     "descriptionLink": "...", "tokenType": "council" }
5. Sign and submit returned transactions
```

### Create a DAO

```
POST /daos/create
{ "walletPublicKey": "...", "name": "...", "councilMembers": [...],
  "votingPeriodDays": 3, "treasuryCount": 1 }
Fee: 2 SOL. Response includes transactions, signers, and realmPk.
```

### Proposal Lifecycle

```
Draft (0) → sign_off → Voting (2) → finalize → Succeeded (3) → execute → Completed (5)
                                              → Defeated (7) → finalize (reclaim deposit)
```

## Safety Guardrails

- Treasury proposals > 10% of total → flag for human review
- Verify DAO name uniqueness before creating (check `GET /daos`)
- Ensure wallet has >= 2.5 SOL for DAO creation (2 SOL fee + gas)
- Re-check proposal state before executing

## Transaction Signing

All POST endpoints return unsigned transactions as base64. Sign sequentially:

```javascript
for (const txBase64 of transactions) {
  const tx = VersionedTransaction.deserialize(Buffer.from(txBase64, 'base64'));
  tx.sign([walletKeypair, ...additionalSigners]);
  const sig = await connection.sendTransaction(tx);
  await connection.confirmTransaction(sig);
}
```

## References

- **Full API reference** (all endpoints, parameters, response formats): See [references/api.md](references/api.md)
- **Decision trees and workflows** (voting logic, proposal lifecycle, data parsing): See [references/workflows.md](references/workflows.md)
