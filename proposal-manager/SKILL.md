---
name: realms-proposal-manager
description: Create, vote on, and execute governance proposals on Solana DAOs through Realms. Full proposal lifecycle management — draft proposals, attach on-chain instructions (token transfers, program calls, Jupiter swaps), cast votes, finalize voting, and execute passed proposals. Use when an agent or user wants to propose an action to a DAO, vote on governance proposals, execute treasury transactions, or automate proposal monitoring. Triggers on "create proposal", "vote on proposal", "execute proposal", "governance proposal", "treasury transfer", "DAO vote", "proposal status", "finalize vote".
---

# Realms Proposal Manager

Full lifecycle management for DAO governance proposals on Solana.

## API Base

```
https://v2.realms.today/api/v1
```

## Proposal Lifecycle

```
Draft (0) → sign_off → Voting (2) → finalize → Succeeded (3) → execute → Completed (5)
                                              → Defeated (7)
```

## Create a Proposal

### Prerequisites Check

```
1. GET /daos/{realmPk}/members/{walletPk}/voting-power
   → Get depositAmount for community/council
2. GET /daos/{realmPk}/governances
   → Get minTokensToCreateProposal for target governance
3. Verify: depositAmount >= threshold
   (if threshold = "18446744073709551615" → DISABLED, cannot create)
4. Pick governancePk (matches the treasury you want to act on)
```

### Simple Proposal (No On-Chain Instructions)

For signaling, discussion, or off-chain decisions:

```json
POST /daos/{realmPk}/proposals/create
{
  "walletPublicKey": "...",
  "governancePk": "...",
  "name": "Increase marketing budget",
  "descriptionLink": "We should allocate 500 USDC/month to community growth...",
  "tokenType": "council"
}
```

### Executable Proposal (With Instructions)

For treasury transfers, program calls, parameter changes:

```json
POST /daos/{realmPk}/proposals/create
{
  "walletPublicKey": "...",
  "governancePk": "...",
  "name": "Transfer 1000 USDC to contributor",
  "descriptionLink": "Payment for Q1 development work",
  "tokenType": "council",
  "instructions": [
    {
      "programId": "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA",
      "accounts": [
        { "pubkey": "SOURCE_ATA", "isSigner": false, "isWritable": true },
        { "pubkey": "DEST_ATA", "isSigner": false, "isWritable": true },
        { "pubkey": "GOVERNANCE_AUTHORITY", "isSigner": true, "isWritable": false }
      ],
      "data": "BASE64_ENCODED_TRANSFER_IX"
    }
  ]
}
```

### Draft Proposal

Keep in draft state for review before opening to votes:

```json
{
  ...
  "isDraft": true
}
```

Then sign off when ready: `POST /daos/{realmPk}/proposals/{proposalPk}/sign-off`

## Vote on a Proposal

### Before Voting

```
1. GET /daos/{realmPk}/proposals/{proposalPk}
   → state must be 2 (Voting)
   → Read descriptionLink
   → Check instructionsCount (if > 0, review what code executes)

2. GET /daos/{realmPk}/members/{walletPk}/voting-power
   → Verify you have voting power

3. GET /daos/{realmPk}/proposals/{proposalPk}/votes
   → Analyze current sentiment (yesWeight vs noWeight)

4. If treasury funds involved:
   GET /daos/{realmPk}/treasury
   → Calculate what % of treasury the proposal affects
   → Flag if > 10% for human review
```

### Cast Vote

```json
POST /daos/{realmPk}/proposals/{proposalPk}/vote
{
  "walletPublicKey": "...",
  "voteKind": 0,
  "createTokenOwnerRecord": true
}
```

| voteKind | Action | Notes |
|----------|--------|-------|
| 0 | Approve | Vote yes |
| 1 | Deny | Vote no |
| 2 | Abstain | Counted but neutral |
| 3 | Veto | Blocks proposal (uses OPPOSITE token type) |

Fee: 0.1 SOL on first vote only (creates token owner record).

### Change Your Mind

Withdraw vote while proposal is still in Voting state:

```json
POST /daos/{realmPk}/proposals/{proposalPk}/relinquish
{ "walletPublicKey": "..." }
```

## After Voting Ends

### Finalize (Permissionless)

If voting period ended but state is still `2`:

```json
POST /daos/{realmPk}/proposals/{proposalPk}/finalize
{ "walletPublicKey": "..." }
```

Any wallet can call this. Moves to Succeeded (3) or Defeated (7).

### Execute (Permissionless)

If state is `3` (Succeeded) and `instructionsCount > 0`:

```json
POST /daos/{realmPk}/proposals/{proposalPk}/execute
{ "walletPublicKey": "..." }
```

Any wallet can call this. Executes the on-chain instructions.

### Cancel (Creator Only)

```json
POST /daos/{realmPk}/proposals/{proposalPk}/cancel
{ "walletPublicKey": "..." }
```

## Automated Proposal Monitoring

For agents that monitor and execute governance:

```
1. GET /daos/{realmPk}/proposals → list all
2. Filter by state:
   - State 2 + voting ended → finalize_vote
   - State 3 → execute_proposal
3. For new proposals (state 2, voting active):
   → Read description, analyze instructions
   → Apply voting policy (see Safety)
   → Cast vote with rationale
```

## Agent Voting Policy

When voting autonomously, always attach rationale. Recommended guardrails:

- **Auto-approve:** Routine operations, parameter tweaks < 5% change
- **Auto-deny:** Proposals with no description, anonymous proposers, > 25% treasury
- **Escalate to human:** Treasury moves 10-25%, new member additions, governance parameter changes
- **Always deny:** Proposals that would remove human oversight, change voting thresholds to exclude humans

## Transaction Signing

All POST endpoints return unsigned transactions as base64:

```javascript
const { transactions, signers = [] } = response;
const additionalSigners = (signers || []).map(s =>
  Keypair.fromSecretKey(bs58.decode(s.secretKey))
);

for (const txBase64 of transactions) {
  const tx = VersionedTransaction.deserialize(Buffer.from(txBase64, 'base64'));
  tx.sign([walletKeypair, ...additionalSigners]);
  const sig = await connection.sendTransaction(tx);
  await connection.confirmTransaction(sig);
}
```

## Proposal States Reference

| State | Value | Available Actions |
|-------|-------|-------------------|
| Draft | 0 | sign_off, cancel |
| SigningOff | 1 | sign_off |
| Voting | 2 | vote, relinquish, cancel |
| Succeeded | 3 | execute |
| Executing | 4 | (wait) |
| Completed | 5 | (none) |
| Cancelled | 6 | finalize |
| Defeated | 7 | finalize |
| Vetoed | 8 | (none) |

## References

For detailed API parameters, response formats, error codes, and data parsing: See [governance/references/api.md](../governance/references/api.md)
