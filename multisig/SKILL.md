---
name: realms-multisig
description: Create and operate M-of-N multisig wallets on Solana using Realms governance. Safer alternative to Squads — fully on-chain, open source, battle-tested by 4,000+ DAOs. Use for team treasuries, protocol admin keys, shared wallets, or any scenario requiring multiple signers to approve transactions. Triggers on "multisig", "multi-sig", "shared wallet", "team treasury", "M-of-N", "multiple signers", "cosign", "Squads alternative".
---

# Realms Multisig

M-of-N multisig wallets powered by SPL Governance. No separate program — uses the same battle-tested governance infrastructure as all Realms DAOs.

## API

```
https://v2.realms.today/api/v1
```

## Quick Start: Create a Multisig

```json
POST /daos/create
{
  "walletPublicKey": "CREATOR_WALLET",
  "name": "Team Treasury",
  "councilMembers": ["signer1", "signer2", "signer3"],
  "isMultisig": true,
  "councilQuorum": 66,
  "votingPeriodDays": 2,
  "treasuryCount": 1
}
```

Fee: **2 SOL**. This creates a 2-of-3 multisig (66% of 3 = 2 signers needed).

## How It Works

Realms multisig = a council-only DAO where:
- Each council member = 1 signer (equal weight)
- Quorum % determines M-of-N threshold
- "Signing" = voting Approve on a proposal
- "Executing" = running the proposal's on-chain instructions after quorum is met

### M-of-N Examples

| Signers | Quorum | Required | Setup |
|---------|--------|----------|-------|
| 3 | 66% | 2-of-3 | Most common for small teams |
| 5 | 60% | 3-of-5 | Protocol admin |
| 7 | 43% | 3-of-7 | Large committee, fast execution |
| 3 | 100% | 3-of-3 | Maximum security, all must sign |
| 2 | 50% | 1-of-2 | Low security, any signer can execute |

## Workflow: Send Tokens from Multisig

### 1. Create Transfer Proposal

```json
POST /daos/{realmPk}/proposals/create
{
  "walletPublicKey": "PROPOSER_WALLET",
  "governancePk": "TREASURY_GOVERNANCE",
  "name": "Pay contractor 500 USDC",
  "descriptionLink": "Monthly payment to @dev for frontend work. Invoice: ...",
  "tokenType": "council",
  "instructions": [
    {
      "programId": "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA",
      "accounts": [
        { "pubkey": "TREASURY_USDC_ATA", "isSigner": false, "isWritable": true },
        { "pubkey": "RECIPIENT_USDC_ATA", "isSigner": false, "isWritable": true },
        { "pubkey": "GOVERNANCE_AUTHORITY", "isSigner": true, "isWritable": false }
      ],
      "data": "BASE64_TRANSFER_DATA"
    }
  ]
}
```

### 2. Other Signers Approve

Each co-signer votes Approve:

```json
POST /daos/{realmPk}/proposals/{proposalPk}/vote
{
  "walletPublicKey": "COSIGNER_WALLET",
  "voteKind": 0
}
```

### 3. Execute After Quorum

Once enough signers approve:

```json
POST /daos/{realmPk}/proposals/{proposalPk}/execute
{ "walletPublicKey": "ANY_WALLET" }
```

Permissionless — anyone can trigger execution once quorum is met and voting period ends.

## Common Operations

### Add a Signer
Create a proposal to mint 1 council token to the new signer's wallet.

### Remove a Signer
Create a proposal to burn the council token from the removed signer's wallet. Adjust quorum if needed.

### Change Threshold
Create a proposal to update the governance config's `councilQuorum` parameter.

### Multiple Treasuries
Create with `treasuryCount: 2+` for separate wallets (e.g., operating funds vs reserves). Each treasury gets its own governance account with independent proposal thresholds.

## Agent-Operated Multisig

For AI agents participating in multisig operations:

```
1. Monitor: GET /daos/{realmPk}/proposals → filter state === 2 (Voting)
2. Analyze: Read proposal description and instructions
3. Decide: Apply policy (see below)
4. Sign: POST vote with voteKind 0 (approve) or 1 (deny)
5. Execute: POST execute for state === 3 proposals
```

### Agent Signing Policy

- **Auto-approve:** Recurring payments to known addresses, routine operations
- **Require human co-sign:** New recipients, amounts > threshold, governance changes
- **Auto-deny:** Unknown programs, proposals without descriptions, self-payments

## Transaction Signing

All responses are unsigned transactions (base64). Sign sequentially:

```javascript
for (const txBase64 of transactions) {
  const tx = VersionedTransaction.deserialize(Buffer.from(txBase64, 'base64'));
  tx.sign([walletKeypair]);
  const sig = await connection.sendTransaction(tx);
  await connection.confirmTransaction(sig);
}
```

## vs Squads

| Feature | Realms Multisig | Squads |
|---------|----------------|--------|
| Program | SPL Governance (audited, open source) | Squads MPL |
| DAOs using it | 4,000+ | ~2,000 |
| Upgradeable to full DAO | ✅ Same program | ❌ Migration needed |
| Prediction markets | ✅ Sowellian plugin | ❌ |
| Token launches | ✅ Launchpad integration | ❌ |
| API | ✅ REST API | SDK only |
| Agent-ready | ✅ AgentSkills | ❌ |

## Safety

- Always verify recipient addresses before proposing transfers
- Use descriptive proposal names and descriptions (audit trail)
- Set `votingPeriodDays` >= 1 for non-urgent operations (gives time to review)
- For high-value treasuries, use 100% quorum (all signers must approve)
- Never set quorum below 50% unless you trust any single signer

## References

For detailed API docs: See [governance/references/api.md](../governance/references/api.md)
