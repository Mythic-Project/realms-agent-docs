# Realms Governance API Reference

Base URL: `https://v2.realms.today/api/v1`

## Endpoints

### DAO Discovery

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/daos` | List all DAOs |
| GET | `/daos/{realmPk}` | Get DAO details (config, mints, authority) |
| GET | `/leaderboard` | DAOs ranked by proposalsCount + membersCount |

### Membership

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/daos/{realmPk}/members` | List all members |
| GET | `/daos/{realmPk}/members/{walletPk}/voting-power` | Check voting power |
| POST | `/daos/{realmPk}/join` | Deposit tokens to join (fee: 0.1 SOL first time) |
| POST | `/daos/{realmPk}/leave` | Withdraw all tokens and leave |

#### Voting Power Response

```json
{
  "community": { "depositAmount": "1000000", "delegate": null } | null,
  "council": { "depositAmount": "1", "delegate": null } | null
}
```

- Both `null` → not a member
- `community` has value → can vote on community proposals
- `council` has value → can vote on council proposals

#### Join DAO

```json
POST /daos/{realmPk}/join
{
  "walletPublicKey": "...",
  "amount": "1000000",   // base units (community: 6 decimals, council: 0)
  "tokenType": "community"  // or "council"
}
```

#### Leave DAO

Cannot leave if `unrelinquishedVotesCount > 0` (must relinquish votes first).

### Delegation

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/daos/{realmPk}/delegates` | List all delegates and delegators |
| POST | `/daos/{realmPk}/delegate` | Set delegate |
| POST | `/daos/{realmPk}/undelegate` | Remove delegate |

#### Set Delegate

```json
POST /daos/{realmPk}/delegate
{
  "walletPublicKey": "...",
  "delegatePublicKey": "...",
  "tokenType": "community",
  "createTokenOwnerRecord": true
}
```

### Governance & Treasury

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/daos/{realmPk}/governances` | List governance accounts with configs |
| GET | `/daos/{realmPk}/treasury` | Get SOL balances per governance |
| GET | `/daos/{realmPk}/treasury/{governancePk}/tokens` | Get SPL token holdings |

Key governance config fields:
- `minCommunityTokensToCreateProposal` — threshold for community proposals
- `minCouncilTokensToCreateProposal` — threshold for council proposals
- `communityVoteThreshold` — % needed to pass
- `baseVotingTime` — voting duration in seconds
- `"18446744073709551615"` = DISABLED (u64::MAX)
- `{ "type": 2 }` = feature disabled

### Proposals

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/daos/{realmPk}/proposals` | List all proposals |
| GET | `/daos/{realmPk}/proposals/{proposalPk}` | Get proposal details |
| GET | `/daos/{realmPk}/proposals/{proposalPk}/votes` | Get all votes |
| POST | `/daos/{realmPk}/proposals/create` | Create proposal |
| POST | `/daos/{realmPk}/proposals/{proposalPk}/vote` | Cast vote |
| POST | `/daos/{realmPk}/proposals/{proposalPk}/sign-off` | Sign off draft |
| POST | `/daos/{realmPk}/proposals/{proposalPk}/finalize` | Finalize voting (permissionless) |
| POST | `/daos/{realmPk}/proposals/{proposalPk}/execute` | Execute passed proposal (permissionless) |
| POST | `/daos/{realmPk}/proposals/{proposalPk}/cancel` | Cancel proposal (creator only) |
| POST | `/daos/{realmPk}/proposals/{proposalPk}/relinquish` | Withdraw vote |

#### Proposal States

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

#### Create Proposal

```json
POST /daos/{realmPk}/proposals/create
{
  "walletPublicKey": "...",
  "governancePk": "...",
  "name": "Proposal Title",
  "descriptionLink": "Description or URL",
  "isDraft": false,
  "options": ["Approve"],
  "tokenType": "council",
  "instructions": [
    {
      "programId": "11111111111111111111111111111111",
      "accounts": [{ "pubkey": "...", "isSigner": true, "isWritable": true }],
      "data": "base64EncodedData"
    }
  ]
}
```

#### Vote

```json
POST /daos/{realmPk}/proposals/{proposalPk}/vote
{
  "walletPublicKey": "...",
  "voteKind": 0,
  "createTokenOwnerRecord": true
}
```

Vote kinds: 0=Approve, 1=Deny, 2=Abstain, 3=Veto

Fee: 0.1 SOL if `createTokenOwnerRecord: true` (first vote only).

#### Create DAO

```json
POST /daos/create
{
  "walletPublicKey": "...",
  "name": "My DAO",
  "councilMembers": ["wallet1", "wallet2"],
  "communityMint": "...",
  "votingPeriodDays": 3,
  "councilQuorum": 10,
  "communityQuorum": 10,
  "treasuryCount": 1,
  "isMultisig": false,
  "basicInfo": {
    "displayName": "My DAO",
    "shortDescription": "...",
    "category": "defi",
    "website": "https://...",
    "twitter": "handle"
  }
}
```

Fee: 2 SOL. Response includes `transactions`, `signers`, `realmPk`.

### User Queries

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/user/{walletPk}/daos` | All DAOs user is member of |
| GET | `/user/{walletPk}/votes` | All votes cast by user |

## Data Parsing

### Hex Vote Weights

```javascript
const weight = BigInt("0x" + voteWeight);
const humanReadable = Number(weight) / 1_000_000; // community tokens: 6 decimals
```

### Timestamps

Unix epoch seconds: `new Date(Number(timestamp) * 1000)`

### Disabled Check

```javascript
const isDisabled = value === "18446744073709551615"; // u64::MAX
const isThresholdDisabled = threshold.type === 2;
```

## Transaction Signing

All POST endpoints return unsigned transactions:

```javascript
const { transactions, signers = [] } = response;
const additionalSigners = signers.map(s => Keypair.fromSecretKey(bs58.decode(s.secretKey)));

for (const txBase64 of transactions) {
  const tx = VersionedTransaction.deserialize(Buffer.from(txBase64, 'base64'));
  tx.sign([walletKeypair, ...additionalSigners]);
  const sig = await connection.sendTransaction(tx);
  await connection.confirmTransaction(sig);
}
```

## Fees

| Action | Fee |
|--------|-----|
| Create DAO | 2 SOL |
| First join/vote | 0.1 SOL |
| Metadata proposal | 0.5 SOL |
| Proposal deposit | Varies (refunded on finalize/cancel) |

## Error Codes

| Status | Meaning | Action |
|--------|---------|--------|
| 400 | Invalid input | Check pubkey format, required fields |
| 404 | Not found | Verify realmPk/proposalPk exists |
| 409 | Conflict | Name taken or duplicate action |
| 429 | Rate limited | Wait for Retry-After header |
| 500 | Server error | Retry with exponential backoff |
