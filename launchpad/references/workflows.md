# Launchpad Decision Trees

## Should I Invest in This Pool?

```
START
  │
  ├─► get_pool(quoteMint) → status === "Active"?
  │     NO → Cannot invest
  │     YES ↓
  │
  ├─► now >= startTime?
  │     NO → Pool not started yet
  │     YES ↓
  │
  ├─► now < endTime?
  │     NO → Pool ended
  │     YES ↓
  │
  ├─► get_pool_metadata → geoRestricted?
  │     YES & user in restricted region → Cannot invest
  │     NO ↓
  │
  ├─► User has sufficient USDC?
  │     NO → Need to acquire USDC first
  │     YES ↓
  │
  ├─► Explain terms to user:
  │     - Minimum raise, current progress %
  │     - Exchange rate, time remaining
  │     - Risk: if minimum not met → refund only
  │
  ├─► Get explicit user confirmation
  │     NO → Do not proceed
  │     YES → Execute invest transaction
```

## Can I Claim from This Pool?

```
START
  │
  ├─► get_user_receipts(wallet) → filter this pool
  │
  ├─► Any unclaimed (claimed === false)?
  │     NO → Nothing to claim
  │     YES ↓
  │
  ├─► Pool status === Active?
  │     YES → Cannot claim yet
  │     NO ↓
  │
  ├─► Graduated/Locked → receive pool tokens
  │   Failed/Cancelled → receive USDC refund
  │
  └─► Execute claim for each unclaimed receipt
```

## Creating a Fundraising Pool

```
1. Gather: name, symbol, minimumRaise, startTime (future), endTime, council (1-8)
2. Upload token metadata → get URI
3. Generate quoteMint keypair
4. createPool → sign with admin + quoteMint keypair
5. After endTime + minimum met → graduatePool (permissionless)
6. setupDao → creates SPL Governance DAO
```

## Safety Red Flags

- No metadata / no social links / anonymous admin
- Extremely high minimum raise relative to progress
- Very short time windows
- Pool with 0% progress near end time

## Integration with Governance

After `setup_dao`:
- Realm created with pool token as community mint
- Council mint for privileged governance
- Treasury holds raised USDC
- Use **realms-governance** skill for ongoing DAO management
