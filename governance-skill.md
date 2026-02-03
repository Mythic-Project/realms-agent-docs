# Realms Governance Skill

You are an AI agent that can interact with Solana DAOs through the Realms Governance API. This skill enables you to create DAOs, manage proposals, cast votes, and participate in on-chain governance.

---

## Agent Instructions

### Your Capabilities
- Query DAO information, proposals, members, and treasuries
- Create new DAOs with metadata and council members
- Create, vote on, and manage governance proposals
- Join/leave DAOs and manage delegation
- Execute passed proposals and finalize votes

### Critical Rules

1. **NEVER vote blindly** — Always analyze proposal content, current votes, and treasury impact before casting a vote
2. **ALWAYS check membership** before voting — Call `get_voting_power` first
3. **ALWAYS check permissions** before creating proposals — Verify token thresholds
4. **Transactions must be signed** — All POST endpoints return unsigned transactions that require wallet signing
5. **Submit transactions sequentially** — Later transactions depend on earlier ones
6. **Rate limit: 50 requests/minute** — Pace your requests accordingly

### Before Any Write Operation
```
1. Check wallet balance (need SOL for fees)
2. Check membership status (get_voting_power)
3. Verify permissions (governance config)
4. Execute the operation
5. Sign and submit transactions in order
```

---

## Configuration

```yaml
base_url: https://v2.realms.today/api/v1
cluster: mainnet-beta  # or devnet
rate_limit: 50/minute
```

---

## Tools

### DAO Discovery

#### list_daos
List all DAOs on Realms.

```
GET /daos
```

**Parameters:** None

**Returns:** Array of DAOs with `publicKey`, `name`, `mint`, `program`, `council`

**Use when:** User asks "what DAOs exist" or "show me DAOs"

---

#### get_dao
Get detailed information about a specific DAO.

```
GET /daos/{realmPk}
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| realmPk | string | yes | DAO public key |

**Returns:** Full realm data including config, mints, authority

**Use when:** User asks about a specific DAO or you need governance config

---

#### get_leaderboard
Get DAOs ranked by activity.

```
GET /leaderboard
```

**Returns:** DAOs sorted by `proposalsCount` + `membersCount`

**Use when:** User asks "most active DAOs" or "top DAOs"

---

### Membership

#### get_voting_power
Check a wallet's membership and voting power in a DAO.

```
GET /daos/{realmPk}/members/{walletPk}/voting-power
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| realmPk | string | yes | DAO public key |
| walletPk | string | yes | Wallet to check |

**Returns:**
```json
{
  "community": { "depositAmount": "1000000", "delegate": null } | null,
  "council": { "depositAmount": "1", "delegate": null } | null
}
```

**Decision logic:**
- If both `null` → Wallet is NOT a member
- If `community` has value → Can vote on community proposals
- If `council` has value → Can vote on council proposals

**ALWAYS call this before:** `vote`, `create_proposal`, `delegate`

---

#### join_dao
Deposit tokens to become a DAO member.

```
POST /daos/{realmPk}/join
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| walletPublicKey | string | yes | Your wallet |
| amount | string | yes | Token amount in BASE UNITS |
| tokenType | string | no | `community` or `council` |

**Token decimals:**
- Community: 6 decimals → `"1000000"` = 1.0 token
- Council: 0 decimals → `"1"` = 1 token

**Fee:** 0.1 SOL (first time only)

**Prerequisites:**
1. Wallet must hold the tokens in ATA
2. Wallet must have 0.1 SOL for account creation

---

#### leave_dao
Withdraw all tokens and leave a DAO.

```
POST /daos/{realmPk}/leave
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| walletPublicKey | string | yes | Your wallet |
| tokenType | string | no | `community` or `council` |

**Cannot leave if:** `unrelinquishedVotesCount > 0` (must relinquish votes first)

---

#### get_members
List all members of a DAO.

```
GET /daos/{realmPk}/members
```

**Returns:** Array of token owner records with deposit amounts and delegates

---

### Delegation

#### set_delegate
Delegate your voting power to another wallet.

```
POST /daos/{realmPk}/delegate
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| walletPublicKey | string | yes | Your wallet (delegator) |
| delegatePublicKey | string | yes | Delegate's wallet |
| tokenType | string | no | `community` or `council` |
| createTokenOwnerRecord | boolean | no | Create record if none exists |

---

#### remove_delegate
Remove your delegate.

```
POST /daos/{realmPk}/undelegate
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| walletPublicKey | string | yes | Your wallet |
| tokenType | string | no | `community` or `council` |

---

#### list_delegates
Get all delegates and their delegators.

```
GET /daos/{realmPk}/delegates
```

**Use when:** Analyzing voting power distribution or finding influential delegates

---

### Governance & Treasury

#### list_governances
Get all governance accounts (treasuries) for a DAO.

```
GET /daos/{realmPk}/governances
```

**Returns:** Array with governance configs including:
- `minCommunityTokensToCreateProposal` — threshold for community proposals
- `minCouncilTokensToCreateProposal` — threshold for council proposals
- `communityVoteThreshold` — % needed to pass
- `baseVotingTime` — voting duration in seconds

**Key values to check:**
- `"18446744073709551615"` = DISABLED (u64::MAX)
- `{ "type": 2 }` = feature disabled

---

#### get_treasury
Get treasury balances.

```
GET /daos/{realmPk}/treasury
```

**Returns:** Governance addresses with native SOL balances

---

#### get_treasury_tokens
Get SPL token holdings for a specific treasury.

```
GET /daos/{realmPk}/treasury/{governancePk}/tokens
```

**Returns:** Token accounts with mint, amount, decimals

---

### Proposals

#### list_proposals
Get all proposals for a DAO.

```
GET /daos/{realmPk}/proposals
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| source | query | no | `rpc` (default) or `backend` |

**Proposal states:**
| State | Value | Meaning | Available Actions |
|-------|-------|---------|-------------------|
| Draft | 0 | Not yet open | sign_off, cancel |
| SigningOff | 1 | Awaiting signatures | sign_off |
| Voting | 2 | Open for votes | vote, relinquish, cancel |
| Succeeded | 3 | Passed, ready to execute | execute |
| Executing | 4 | Being executed | (wait) |
| Completed | 5 | Done | (none) |
| Cancelled | 6 | Cancelled | finalize |
| Defeated | 7 | Failed | finalize |
| Vetoed | 8 | Blocked by veto | (none) |

---

#### get_proposal
Get a specific proposal with full details.

```
GET /daos/{realmPk}/proposals/{proposalPk}
```

**Key fields to analyze:**
- `descriptionLink` — Read this to understand the proposal
- `state` — Current lifecycle state
- `options[].voteWeight` — Hex-encoded approve votes
- `denyVoteWeight` — Hex-encoded deny votes
- `maxVoteWeight` — Total possible votes
- `instructionsCount` — If > 0, proposal executes on-chain code

---

#### create_proposal
Create a new governance proposal.

```
POST /daos/{realmPk}/proposals/create
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| walletPublicKey | string | yes | Proposer wallet |
| governancePk | string | yes | Target governance |
| name | string | yes | Proposal title |
| descriptionLink | string | no | Description or URL |
| isDraft | boolean | no | Keep in draft state |
| options | string[] | no | Vote options (default: `["Approve"]`) |
| instructions | object[] | no | On-chain instructions to execute |
| tokenType | string | no | `community` or `council` |

**Instruction format:**
```json
{
  "programId": "11111111111111111111111111111111",
  "accounts": [
    { "pubkey": "...", "isSigner": true, "isWritable": true }
  ],
  "data": "base64EncodedData"
}
```

**Before creating:**
1. Call `get_voting_power` — verify membership
2. Call `list_governances` — get `minTokensToCreateProposal`
3. Compare: `depositAmount >= minTokensToCreateProposal`
4. If threshold is `u64::MAX`, that token type CANNOT create proposals

---

#### get_votes
Get all votes cast on a proposal.

```
GET /daos/{realmPk}/proposals/{proposalPk}/votes
```

**Returns:** Array of votes with `voter`, `weight`, `voteType` ("yes"/"no")

**Use for sentiment analysis:**
- Count yes vs no votes
- Sum weights to see vote distribution
- Identify whale voters (high weight)

---

#### vote
Cast a vote on a proposal.

```
POST /daos/{realmPk}/proposals/{proposalPk}/vote
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| walletPublicKey | string | yes | Voter wallet |
| voteKind | number | yes | 0=Approve, 1=Deny, 2=Abstain, 3=Veto |
| createTokenOwnerRecord | boolean | no | Auto-join if not member |

**Vote kinds:**
| Value | Name | Effect |
|-------|------|--------|
| 0 | Approve | Vote yes |
| 1 | Deny | Vote no |
| 2 | Abstain | Counted but neutral |
| 3 | Veto | Block proposal (uses OPPOSITE token type) |

**Fee:** 0.1 SOL if `createTokenOwnerRecord: true` (first vote only)

---

#### sign_off_proposal
Move a draft proposal to voting.

```
POST /daos/{realmPk}/proposals/{proposalPk}/sign-off
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| walletPublicKey | string | yes | Creator or signatory wallet |

**Use when:** Proposal `state === 0` (Draft)

---

#### finalize_vote
Finalize voting after period ends.

```
POST /daos/{realmPk}/proposals/{proposalPk}/finalize
```

**Permissionless** — Any wallet can call this.

**Use when:** Voting period ended but state is still `2`

---

#### execute_proposal
Execute a passed proposal's instructions.

```
POST /daos/{realmPk}/proposals/{proposalPk}/execute
```

**Permissionless** — Any wallet can call this.

**Use when:** Proposal `state === 3` (Succeeded)

---

#### cancel_proposal
Cancel a proposal (creator only).

```
POST /daos/{realmPk}/proposals/{proposalPk}/cancel
```

**Use when:** Proposal in Draft or Voting state, and you are the creator

---

#### relinquish_vote
Withdraw your vote from an active proposal.

```
POST /daos/{realmPk}/proposals/{proposalPk}/relinquish
```

**Use when:** Proposal `state === 2` (Voting) and you want to change/remove vote

---

### DAO Creation

#### create_dao
Create a new DAO with governance and treasury.

```
POST /daos/create
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| walletPublicKey | string | yes | Creator wallet |
| name | string | yes | DAO name (max 32 bytes) |
| councilMembers | string[] | no | Council wallet addresses |
| communityMint | string | no | Existing token mint |
| votingPeriodDays | number | no | Default: 3 |
| councilQuorum | number | no | Default: 10 (or 25 for multisig) |
| communityQuorum | number | no | Default: 10 |
| treasuryCount | number | no | Default: 1 |
| isMultisig | boolean | no | Multisig-style DAO |
| basicInfo | object | no | Metadata (see below) |

**basicInfo schema:**
```json
{
  "displayName": "My DAO",           // Required, max 30 chars
  "shortDescription": "...",         // Required, max 120 chars
  "daoImage": "https://...",         // Arweave URL, 512x512px
  "bannerImage": "https://...",      // Arweave URL, 1200x200px
  "category": "defi",                // defi|social|web3|nft|gaming|legal|other
  "website": "https://...",
  "twitter": "handle",               // Without @
  "discord": "https://discord.gg/...",
  "telegram": "handle"               // Without @
}
```

**Fee:** 2 SOL

**Response includes:**
- `transactions` — Sign and submit in order
- `signers` — Additional keypairs to co-sign
- `realmPk` — New DAO address

---

### User Queries

#### get_user_daos
Get all DAOs a wallet is a member of.

```
GET /user/{walletPk}/daos
```

---

#### get_user_votes
Get all votes cast by a wallet.

```
GET /user/{walletPk}/votes
```

---

## Decision Trees

### Should I Vote on This Proposal?

```
START
  │
  ├─► Is proposal state === 2 (Voting)?
  │     NO → Cannot vote (not in voting period)
  │     YES ↓
  │
  ├─► Call get_voting_power for my wallet
  │     Both null? → Cannot vote (not a member)
  │     Has tokens? ↓
  │
  ├─► Read proposal descriptionLink
  │     Understand what it does
  │     │
  ├─► Does it have instructionsCount > 0?
  │     YES → Review what code will execute
  │     │
  ├─► Call get_votes to see current sentiment
  │     Calculate: yesWeight / (yesWeight + noWeight)
  │     │
  ├─► Check treasury impact (if funds involved)
  │     GET /daos/{realmPk}/treasury
  │     What % of treasury does this affect?
  │     │
  └─► Make informed decision
        voteKind: 0 (yes), 1 (no), 2 (abstain), 3 (veto)
```

### Can I Create a Proposal?

```
START
  │
  ├─► Call get_voting_power
  │     Get depositAmount for community/council
  │     │
  ├─► Call list_governances
  │     Get minCommunityTokensToCreateProposal
  │     Get minCouncilTokensToCreateProposal
  │     │
  ├─► Is threshold === "18446744073709551615"?
  │     YES → That token type CANNOT create proposals
  │     NO ↓
  │
  ├─► Compare: depositAmount >= threshold?
  │     NO → Insufficient tokens
  │     YES → Can create proposal
  │
  └─► Pick governancePk from list_governances
        (Use treasury you want to act on)
```

### How to Handle Proposal Lifecycle

```
State 0 (Draft)
  └─► sign_off_proposal → State 2

State 2 (Voting)
  ├─► vote (while active)
  ├─► relinquish_vote (change mind)
  └─► (wait for voting period to end)

State 2 (Voting ended)
  └─► finalize_vote → State 3 or 7

State 3 (Succeeded)
  └─► execute_proposal → State 5

State 7 (Defeated)
  └─► finalize_vote (reclaim deposit)
```

---

## Data Parsing

### Hex Vote Weights
Vote weights like `"94b9bc86c4d5af"` are hex-encoded big numbers.

```javascript
const weight = BigInt("0x" + voteWeight);
// For community tokens (6 decimals):
const humanReadable = Number(weight) / 1_000_000;
```

### Timestamps
Timestamps like `"1713373926"` are Unix epoch seconds.

```javascript
const date = new Date(Number(timestamp) * 1000);
```

### Checking if Feature is Disabled
```javascript
const isDisabled = value === "18446744073709551615"; // u64::MAX
const isThresholdDisabled = threshold.type === 2;
```

---

## Transaction Signing

All POST endpoints return unsigned transactions. Your agent must:

```javascript
import { VersionedTransaction, Keypair, Connection } from '@solana/web3.js';
import bs58 from 'bs58';

const connection = new Connection('https://api.mainnet-beta.solana.com');

// 1. Parse response
const { transactions, signers = [] } = response;

// 2. Reconstruct additional signers
const additionalSigners = signers.map(s =>
  Keypair.fromSecretKey(bs58.decode(s.secretKey))
);

// 3. Sign and submit SEQUENTIALLY
for (const txBase64 of transactions) {
  const tx = VersionedTransaction.deserialize(
    Buffer.from(txBase64, 'base64')
  );
  tx.sign([walletKeypair, ...additionalSigners]);

  const sig = await connection.sendTransaction(tx);
  await connection.confirmTransaction(sig);
}
```

---

## Fee Summary

| Action | Fee | Notes |
|--------|-----|-------|
| Create DAO | 2 SOL | Always required |
| First join/vote | 0.1 SOL | Token owner record creation |
| Metadata proposal | 0.5 SOL | Only if `includesMetadata: true` |
| Proposal deposit | Varies | Refunded on finalize/cancel |

---

## Error Handling

| Status | Meaning | Action |
|--------|---------|--------|
| 400 | Invalid input | Check public key format, required fields |
| 404 | Not found | Verify realmPk/proposalPk exists |
| 409 | Conflict | Name taken, or duplicate action |
| 429 | Rate limited | Wait for `Retry-After` header value |
| 500 | Server error | Retry with exponential backoff |

---

## Safety Guardrails

### Before Voting YES on Treasury Proposals
1. Calculate `proposalAmount / totalTreasury`
2. If > 10% of treasury, flag for human review
3. Check if similar proposals failed recently
4. Verify proposer has history of successful proposals

### Before Creating DAOs
1. Verify unique name (check `list_daos`)
2. Ensure wallet has >= 2.5 SOL (2 SOL fee + gas)
3. Confirm council members are valid Solana addresses

### Before Executing Proposals
1. Re-check proposal state is `3` (Succeeded)
2. Verify `instructionsCount > 0` (has something to execute)
3. Understand what the instructions will do

---

## Example Workflows

### Vote on an Active Proposal

```
1. GET /daos/{realmPk}/proposals/{proposalPk}
   → Check state === 2
   → Read descriptionLink
   → Note instructionsCount

2. GET /daos/{realmPk}/members/{walletPk}/voting-power
   → Verify community or council has depositAmount

3. GET /daos/{realmPk}/proposals/{proposalPk}/votes
   → Analyze current vote distribution

4. POST /daos/{realmPk}/proposals/{proposalPk}/vote
   {
     "walletPublicKey": "...",
     "voteKind": 0,
     "createTokenOwnerRecord": true
   }

5. Sign and submit returned transaction
```

### Create a Simple Proposal

```
1. GET /daos/{realmPk}/members/{walletPk}/voting-power
   → Get depositAmount

2. GET /daos/{realmPk}/governances
   → Get minCouncilTokensToCreateProposal
   → Get first governance pubkey

3. Verify: depositAmount >= minTokens

4. POST /daos/{realmPk}/proposals/create
   {
     "walletPublicKey": "...",
     "governancePk": "...",
     "name": "Proposal Title",
     "descriptionLink": "Description text or URL",
     "tokenType": "council"
   }

5. Sign and submit returned transactions
```

### Monitor and Execute Passed Proposals

```
1. GET /daos/{realmPk}/proposals
   → Filter for state === 3 (Succeeded)

2. For each succeeded proposal:
   POST /daos/{realmPk}/proposals/{proposalPk}/execute
   {
     "walletPublicKey": "..."
   }

3. Sign and submit (permissionless - any wallet can execute)
```

---

## API Reference Summary

| Category | Endpoint | Method | Purpose |
|----------|----------|--------|---------|
| DAOs | `/daos` | GET | List all DAOs |
| DAOs | `/daos/{realmPk}` | GET | Get DAO details |
| DAOs | `/daos/create` | POST | Create new DAO |
| DAOs | `/leaderboard` | GET | DAOs by activity |
| Members | `/daos/{realmPk}/members` | GET | List members |
| Members | `/daos/{realmPk}/members/{walletPk}/voting-power` | GET | Check voting power |
| Members | `/daos/{realmPk}/join` | POST | Join DAO |
| Members | `/daos/{realmPk}/leave` | POST | Leave DAO |
| Delegation | `/daos/{realmPk}/delegates` | GET | List delegates |
| Delegation | `/daos/{realmPk}/delegate` | POST | Set delegate |
| Delegation | `/daos/{realmPk}/undelegate` | POST | Remove delegate |
| Governance | `/daos/{realmPk}/governances` | GET | List governances |
| Treasury | `/daos/{realmPk}/treasury` | GET | Get treasury balances |
| Treasury | `/daos/{realmPk}/treasury/{governancePk}/tokens` | GET | Get treasury tokens |
| Proposals | `/daos/{realmPk}/proposals` | GET | List proposals |
| Proposals | `/daos/{realmPk}/proposals/{proposalPk}` | GET | Get proposal |
| Proposals | `/daos/{realmPk}/proposals/create` | POST | Create proposal |
| Proposals | `/daos/{realmPk}/proposals/{proposalPk}/votes` | GET | Get votes |
| Proposals | `/daos/{realmPk}/proposals/{proposalPk}/vote` | POST | Cast vote |
| Proposals | `/daos/{realmPk}/proposals/{proposalPk}/sign-off` | POST | Sign off draft |
| Proposals | `/daos/{realmPk}/proposals/{proposalPk}/finalize` | POST | Finalize voting |
| Proposals | `/daos/{realmPk}/proposals/{proposalPk}/execute` | POST | Execute proposal |
| Proposals | `/daos/{realmPk}/proposals/{proposalPk}/cancel` | POST | Cancel proposal |
| Proposals | `/daos/{realmPk}/proposals/{proposalPk}/relinquish` | POST | Withdraw vote |
| User | `/user/{walletPk}/daos` | GET | User's DAOs |
| User | `/user/{walletPk}/votes` | GET | User's votes |
