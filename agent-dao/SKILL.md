---
name: realms-agent-dao
description: End-to-end workflow for AI agents to found, fund, and run autonomous on-chain organizations on Solana. An agent can spin up a DAO, launch a token sale (IDO), graduate to governance, manage treasury operations, and execute trades — all programmatically, all transparent, all human-legible. The complete "agent-founded enterprise" stack. Use when an agent wants to create an autonomous organization, launch a fund, raise capital through IDO, or run ongoing treasury operations with governance. Triggers on "agent DAO", "autonomous organization", "agent-founded", "AI fund", "launch an organization", "agent enterprise", "found a DAO", "agent IDO".
---

# Agent-Founded DAOs on Realms

The complete stack for AI agents to found, fund, and run on-chain organizations.

## Why DAOs for Agents?

Traditional companies need lawyers, paperwork, bank accounts, board meetings. AI agents can't do any of that. But they can:

- Deploy a DAO in one transaction
- Raise capital through a programmatic IDO
- Execute treasury operations through governance proposals
- Explain every decision in human-readable rationale
- Open ownership to the public without an IPO

DAOs are the native organizational form for AI agents.

## The End-to-End Flow

```
1. FOUND    → Create a DAO with agent + human oversight council
2. FUND     → Launch an IDO (token sale) to raise capital
3. GRADUATE → Auto-convert to governed DAO when funding succeeds
4. OPERATE  → Manage treasury through proposals with rationale
5. GROW     → Add prediction markets, DeFi integrations, new members
```

## Phase 1: Found the DAO

Use **realms-dao-creator** to create the organization.

```json
POST https://v2.realms.today/api/v1/daos/create
{
  "walletPublicKey": "AGENT_WALLET",
  "name": "Alpha Fund DAO",
  "councilMembers": ["AGENT_WALLET", "HUMAN_OVERSIGHT_1", "HUMAN_OVERSIGHT_2"],
  "votingPeriodDays": 2,
  "councilQuorum": 50,
  "treasuryCount": 2,
  "basicInfo": {
    "displayName": "Alpha Fund DAO",
    "shortDescription": "AI-managed investment fund with human oversight",
    "category": "defi",
    "twitter": "AlphaFundDAO"
  }
}
```

**Agent DAO design principles:**
- Always include human oversight wallets in council
- Set quorum so agent can propose but not unilaterally execute
- Use 2+ treasuries (operations vs reserves)
- Fill metadata for transparency — agents should be discoverable

## Phase 2: Fund via IDO

Use **realms-launchpad** to raise capital. The launchpad is purpose-built for this: investors contribute USDC, receive governance tokens, and the pool auto-converts to a DAO on completion.

```typescript
// Agent creates a fundraising pool
await program.methods.createPool(
  new BN(50000 * 1e6),           // Minimum raise: 50,000 USDC
  new BN(startTimestamp),         // Opens in 24h
  new BN(endTimestamp),           // 7-day funding window
  [agentWallet, humanOversight],  // Council
  "Alpha Fund",                   // Token name
  "ALPHA",                        // Token symbol
  metadataUri                     // Token metadata
).accounts({...}).signers([quoteMintKeypair]).transaction();
```

What investors get:
- **ALPHA tokens** — governance power proportional to investment
- **Voting rights** — every token holder can vote on proposals
- **Refund guarantee** — if minimum raise isn't met, automatic USDC refund

This is the "opening ownership to the public without an IPO" from one API call.

## Phase 3: Graduate to Governance

After the IDO succeeds (minimum raise met + end time passed):

```typescript
// Permissionless — anyone can call this
await program.methods.graduatePool().accounts({...}).transaction();

// Then set up full DAO governance
await program.methods.setupDao(
  "Alpha Fund DAO",           // Realm name
  2 * 86400,                  // 2-day voting period
  new BN(minWeightToPropose)  // Minimum tokens to create proposals
).accounts({...}).transaction();
```

The pool token (ALPHA) becomes the community governance token. Every investor now has voting power.

## Phase 4: Operate with Transparency

This is where agent DAOs differentiate from traditional companies. Every action goes through governance with human-readable rationale.

### The Transparency Rule

**Every proposal MUST include a rationale in `descriptionLink`.** This is non-negotiable for agent DAOs. The whole point is that "proposals, votes, discussion, rationale, and execution are all human-legible."

Bad:
```json
{ "name": "Swap USDC to SOL", "descriptionLink": "" }
```

Good:
```json
{
  "name": "Swap 5,000 USDC → SOL at $142",
  "descriptionLink": "## Rationale\n\nSOL is trading at $142, down 8% this week. On-chain metrics show:\n- DEX volume up 23% (7d)\n- Staking rate stable at 65%\n- RSI at 32 (oversold)\n\nProposing to allocate 10% of treasury to SOL at current levels.\n\n## Risk\n- Max drawdown if SOL hits $120: -15%\n- Stop loss: $128 (manual proposal to exit)\n- Position size: 10% of treasury (conservative)\n\n## Expected outcome\nTarget: $165 within 2 weeks (16% upside)"
}
```

### Treasury Operations

Use **realms-proposal-execution** for all treasury actions:

```
Agent analyzes market → Creates proposal with rationale → Token holders vote
  → If approved: execute trade → Report results in next proposal
```

Available operations:
- **Jupiter limit orders** — place/cancel limit orders from treasury
- **Jupiter DCA** — set up recurring buys
- **Kamino lending** — earn yield on idle assets
- **Drift perps** — leveraged positions with governance approval
- **Token transfers** — payments, distributions
- **Staking** — validator delegation

### Automated Monitoring

Agent continuously monitors and acts:

```
1. GET /daos/{realmPk}/proposals → check for new proposals
2. For each active proposal:
   - Read rationale
   - Analyze on-chain data
   - Vote with explanation
3. For succeeded proposals:
   - Execute automatically
4. For treasury:
   - Monitor positions
   - Propose rebalances when triggers hit
   - Report performance in proposal descriptions
```

## Phase 5: Grow

### Add Prediction Markets (Sowellian)

Use **realms-sowellian** to add skin-in-the-game governance:

```
Agent proposes: "Buy 10 SOL at $142"
  → Agent stakes tokens on YES (skin in the game)
  → Community members stake YES or NO
  → If proposal passes and trade profits: YES voters win
  → If trade loses: NO voters win
```

This aligns agent incentives with fund performance. Bad trades cost the agent tokens.

### Add New Members

Governance proposal to mint tokens to new council members or adjust quorum.

### Upgrade to Multisig

For high-value operations, require multiple signatures via **realms-multisig**.

## Safety Architecture

### Agent Permissions (Recommended Defaults)

| Action | Agent Can | Requires Human |
|--------|-----------|----------------|
| Create proposals | ✅ | |
| Vote on proposals | ✅ | |
| Execute passed proposals | ✅ | |
| Treasury moves < 5% | ✅ (with rationale) | |
| Treasury moves 5-25% | ✅ propose | ✅ must co-approve |
| Treasury moves > 25% | | ✅ human-initiated only |
| Change governance params | | ✅ human-initiated only |
| Add/remove council members | | ✅ human-initiated only |

### Rationale Requirements

Every agent proposal must include:
1. **What** — clear description of the action
2. **Why** — data-driven rationale
3. **Risk** — what could go wrong, max loss
4. **Expected outcome** — target and timeframe

### Emergency Override

Human council members can always:
- **Veto** any proposal (voteKind: 3)
- **Cancel** proposals (if creator)
- **Change quorum** to require more human votes

## Skill Dependencies

This workflow uses these skills in sequence:

| Phase | Skill | Purpose |
|-------|-------|---------|
| Found | `realms-dao-creator` | Create the DAO |
| Fund | `realms-launchpad` | Run the IDO |
| Operate | `realms-proposal-manager` | Manage proposal lifecycle |
| Execute | `realms-proposal-execution` | Build treasury instructions |
| Trade | `realms-sowellian` | Prediction market governance |
| Secure | `realms-multisig` | Multi-sig for high-value ops |
| Query | `realms-governance` | Read DAO state, membership, treasury |
