---
name: realms-dao-creator
description: Spin up a new DAO on Solana in one conversation. Creates a fully operational on-chain organization with treasury, governance rules, council members, and metadata — all via the Realms API. Use when an agent or user wants to create a DAO, start an organization, set up on-chain governance, launch a community, or build an agent-founded enterprise. Triggers on "create a DAO", "start a DAO", "spin up a DAO", "new organization", "agent DAO", "set up governance", "create multisig" (for multisig-style DAOs).
---

# Realms DAO Creator

Create a fully operational Solana DAO in one API call.

## API

```
POST https://v2.realms.today/api/v1/daos/create
```

## Quick Start

```json
{
  "walletPublicKey": "YOUR_WALLET",
  "name": "My DAO",
  "councilMembers": ["wallet1", "wallet2", "wallet3"],
  "votingPeriodDays": 3,
  "treasuryCount": 1
}
```

Fee: **2 SOL**. Response includes unsigned transactions, optional co-signers, and the new `realmPk`.

## Before Creating

1. Verify name is unique — `GET /daos` and check no collision
2. Wallet has >= 2.5 SOL (2 SOL fee + gas)
3. All council member addresses are valid Solana pubkeys
4. DAO name max 32 bytes (ASCII)

## Parameters

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| walletPublicKey | string | required | Creator wallet |
| name | string | required | DAO name (max 32 bytes) |
| councilMembers | string[] | [] | Council wallet addresses |
| communityMint | string | null | Existing token mint (creates new if omitted) |
| votingPeriodDays | number | 3 | How long proposals stay open |
| councilQuorum | number | 10 (25 if multisig) | % of council vote needed to pass |
| communityQuorum | number | 10 | % of community needed to pass |
| treasuryCount | number | 1 | Number of treasury wallets |
| isMultisig | boolean | false | Multisig-style (council-only, higher quorum) |
| coolOffPeriodDays | number | 1 | Cool-off period after voting (days) |

## Metadata (Optional)

Add `basicInfo` to make the DAO discoverable on Realms:

```json
{
  "basicInfo": {
    "displayName": "My DAO",
    "shortDescription": "A one-line pitch (max 120 chars)",
    "daoImage": "https://arweave.net/...",
    "bannerImage": "https://arweave.net/...",
    "category": "defi",
    "website": "https://example.com",
    "twitter": "handle",
    "discord": "https://discord.gg/...",
    "telegram": "handle"
  }
}
```

Categories: `defi` | `social` | `web3` | `nft` | `gaming` | `legal` | `other`

Images: 512×512 (avatar), 1200×200 (banner). Use Arweave URLs.

## DAO Types

### Community DAO
Standard DAO with token-weighted voting. Anyone holding the governance token can participate.

```json
{
  "walletPublicKey": "...",
  "name": "Community DAO",
  "communityMint": "EXISTING_TOKEN_MINT",
  "votingPeriodDays": 5,
  "communityQuorum": 5
}
```

### Council DAO
Small team governance. Council members have equal voting power.

```json
{
  "walletPublicKey": "...",
  "name": "Council DAO",
  "councilMembers": ["member1", "member2", "member3"],
  "votingPeriodDays": 3,
  "councilQuorum": 60
}
```

### Multisig DAO
M-of-N signing. Higher quorum, council-only.

```json
{
  "walletPublicKey": "...",
  "name": "Team Multisig",
  "councilMembers": ["signer1", "signer2", "signer3"],
  "isMultisig": true,
  "councilQuorum": 66
}
```

### Agent-Founded DAO
For AI agents creating autonomous organizations:

```json
{
  "walletPublicKey": "AGENT_WALLET",
  "name": "Agent Fund Alpha",
  "councilMembers": ["AGENT_WALLET", "HUMAN_OVERSIGHT_WALLET"],
  "votingPeriodDays": 1,
  "councilQuorum": 50,
  "treasuryCount": 2,
  "basicInfo": {
    "displayName": "Agent Fund Alpha",
    "shortDescription": "AI-managed treasury with human oversight",
    "category": "defi"
  }
}
```

**Best practice for agent DAOs:** Always include a human oversight wallet in the council. Set quorum so the agent can propose but not unilaterally execute.

## Transaction Flow

```javascript
const response = await fetch('https://v2.realms.today/api/v1/daos/create', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(params),
});

const { transactions, signers, realmPk } = await response.json();

// Reconstruct co-signers
const additionalSigners = (signers || []).map(s =>
  Keypair.fromSecretKey(bs58.decode(s.secretKey))
);

// Sign and submit SEQUENTIALLY
for (const txBase64 of transactions) {
  const tx = VersionedTransaction.deserialize(Buffer.from(txBase64, 'base64'));
  tx.sign([walletKeypair, ...additionalSigners]);
  const sig = await connection.sendTransaction(tx);
  await connection.confirmTransaction(sig);
}

// realmPk is the new DAO's public key — use with governance skill
```

## What You Get

After creation, the DAO has:
- **Realm** — the DAO identity on-chain (`realmPk`)
- **Governance account(s)** — rules for proposals (thresholds, voting time, quorum)
- **Treasury wallet(s)** — SOL/SPL token accounts controlled by governance
- **Council token mint** — if council members were specified
- **Community token mint** — if `communityMint` was provided or auto-created

## Next Steps

After creating a DAO, use these skills:
- **realms-governance** — create proposals, vote, manage treasury
- **realms-sowellian** — add prediction market governance
- **realms-launchpad** — raise funds with auto-DAO conversion

## Safety

- Never create a DAO without confirming the user/agent has 2.5+ SOL
- Always include human oversight for agent-founded DAOs
- Verify council addresses are real wallets (not program addresses)
- DAO names are permanent — confirm before creating
