---
name: realms-launchpad
description: Interact with the Realms Launchpad on Solana — a fundraising platform for token sales (ICOs) that auto-convert to DAOs. Invest in pools, claim tokens/refunds, create fundraising pools, monitor progress, and set up governance for graduated pools. Use when the task involves token launches, ICOs, fundraising pools, launchpad investments, or "launch a token". Triggers on "launchpad", "fundraise", "ICO", "pool", "invest in pool", "claim tokens", "token sale".
---

# Realms Launchpad

Solana-based fundraising platform. Token sales auto-convert to DAOs on completion.

## Configuration

```yaml
program_id: ReaLM68X8dXLz35oXqofDYkNWiBFZr4FcefSJyTr9Yh
base_token: USDC (EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v)
base_decimals: 6
metadata_url: https://v2.realms.today/launchpad/metadata.json
cluster: mainnet-beta
```

## Critical Rules

1. **Verify pool status** before investing — only `Active` pools within their time window accept investments
2. **Check geo-restrictions** — some pools restrict jurisdictions
3. **Explain risks** — if minimum raise not met, investors only get refunds (no tokens)
4. **Base token is USDC** — all investments in USDC (6 decimals: `1000000` = 1.0 USDC)
5. **Get explicit user confirmation** before investing

## Pool Lifecycle

```
Created → Active → [Graduated | Failed | Cancelled] → Locked (with DAO)
```

| Status | Value | User Actions |
|--------|-------|--------------|
| Active | 0 | Invest (within time window) |
| Cancelled | 1 | Claim refund |
| Graduated | 2 | Claim tokens |
| Failed | 3 | Claim refund |
| Locked | 4 | Claim tokens (if not already) |

## Core Workflows

### Invest in a Pool

```
1. get_pool(quoteMint) → verify status=Active, now >= startTime, now < endTime
2. get_pool_metadata(quoteMint) → check geoRestrictions, show terms to user
3. Verify user has sufficient USDC
4. Get explicit confirmation
5. Call invest(amountUsdc) → sign and submit transaction
```

### Claim Tokens or Refund

```
1. get_user_receipts(wallet) → find unclaimed receipts (claimed === false)
2. Check pool status:
   - Graduated/Locked → receive pool tokens
   - Failed/Cancelled → receive USDC refund
3. Call claim() for each unclaimed receipt
```

### Create a Pool (Admin)

```
1. Gather: name, symbol, minimumRaise, startTime (future), endTime, council (1-8)
2. Upload token metadata → get URI
3. Generate quoteMint keypair
4. Call createPool → sign with admin + quoteMint keypair
5. After pool ends + minimum met: graduatePool (permissionless)
6. setupDao → creates SPL Governance realm with pool token as community mint
```

## Safety Guardrails

- Verify metadata exists and looks professional
- Pools near minimum raise have less failure risk
- Red flags: no metadata, no social links, anonymous admin, very short windows
- After `setupDao`, use the **realms-governance** skill to manage the resulting DAO

## References

- **Full on-chain API** (invest, claim, create pool, graduate, setup DAO, PDA derivations, error codes): See [references/api.md](references/api.md)
- **Decision trees** (should I invest, can I claim, pool creation flow): See [references/workflows.md](references/workflows.md)
