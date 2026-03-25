# Realms Launchpad API Reference

Program: `ReaLM68X8dXLz35oXqofDYkNWiBFZr4FcefSJyTr9Yh`

## Pool Discovery

### list_pools

```typescript
const pools = await program.account.pool.all();
return pools.map(p => ({
  publicKey: p.publicKey.toBase58(),
  admin: p.account.admin.toBase58(),
  quoteMint: p.account.quoteMint.toBase58(),
  minimumRaise: p.account.minimumRaise.toNumber() / 1e6,
  totalRaised: p.account.totalRaised.toNumber() / 1e6,
  startTime: new Date(p.account.startTime.toNumber() * 1000),
  endTime: new Date(p.account.endTime.toNumber() * 1000),
  status: parseStatus(p.account.status),
}));

function parseStatus(status: object): string {
  const key = Object.keys(status)[0];
  return key.charAt(0).toUpperCase() + key.slice(1);
}
```

### get_pool

```typescript
const [poolPda] = PublicKey.findProgramAddressSync(
  [Buffer.from('pool'), quoteMint.toBuffer()],
  PROGRAM_ID
);
const pool = await program.account.pool.fetch(poolPda);
```

Key derived fields:
- `canInvest`: status === Active && now >= startTime && now < endTime
- `progress`: (totalRaised / minimumRaise) * 100

### get_pool_metadata

```typescript
const res = await fetch('https://v2.realms.today/launchpad/metadata.json');
const data = await res.json();
return data.pools[quoteMint]; // { name, tagline, description, logo, banner, website, twitter, discord, tokenSymbol, exchangeRate, geoRestricted, proposalThresholdPercent }
```

## Investment

### invest

```typescript
const amountRaw = new BN(amountUsdc * 1e6);
const seed = Keypair.generate().publicKey;

const [receiptPda] = PublicKey.findProgramAddressSync(
  [Buffer.from('receipt'), poolPda.toBuffer(), seed.toBuffer()],
  PROGRAM_ID
);

const tx = await program.methods
  .invest(amountRaw, seed)
  .accounts({
    investor: wallet,
    pool: poolPda,
    receipt: receiptPda,
    baseMint: pool.baseMint,
    investorTokenAccount: getAssociatedTokenAddressSync(pool.baseMint, wallet),
    baseVault: pool.baseVault,
    tokenProgram: TOKEN_PROGRAM_ID,
    associatedTokenProgram: ASSOCIATED_TOKEN_PROGRAM_ID,
    systemProgram: SystemProgram.programId,
  })
  .transaction();
```

### get_user_receipts

```typescript
const receipts = await program.account.receipt.all([
  { memcmp: { offset: 8 + 1 + 32, bytes: wallet.toBase58() } }
]);
// Returns: publicKey, pool, investor, amount (/ 1e6 for USDC), receiptId, claimed
```

## Claims

### claim

```typescript
const tx = await program.methods
  .claim()
  .accounts({
    investor: wallet,
    pool: poolPda,
    receipt: receiptPda,
    baseMint: pool.baseMint,
    quoteMint: pool.quoteMint,
    baseVault: pool.baseVault,
    quoteVault: pool.quoteVault,
    investorBaseAccount: getAssociatedTokenAddressSync(pool.baseMint, wallet),
    investorQuoteAccount: getAssociatedTokenAddressSync(pool.quoteMint, wallet),
    tokenProgram: TOKEN_PROGRAM_ID,
    associatedTokenProgram: ASSOCIATED_TOKEN_PROGRAM_ID,
    systemProgram: SystemProgram.programId,
  })
  .preInstructions([createBaseAtaIx, createQuoteAtaIx]) // create ATAs idempotent
  .transaction();
```

Outcome by status:
- Graduated/Locked → quote tokens (pool tokens)
- Failed/Cancelled → USDC refund

## Admin Operations

### create_pool

```typescript
const tx = await program.methods
  .createPool(
    new BN(minimumRaise * 1e6),
    new BN(Math.floor(startTime.getTime() / 1000)),
    new BN(Math.floor(endTime.getTime() / 1000)),
    council || null, // 1-8 PublicKeys or null
    name, symbol, uri
  )
  .accounts({ admin, baseMint, pool: poolPda, quoteMint: quoteMintKeypair.publicKey, ... })
  .signers([quoteMintKeypair])
  .transaction();
```

Constraints: endTime > startTime, startTime > now, minimumRaise > 0, council 1-8 members.

### graduate_pool

Permissionless after: status === Active, now >= endTime, totalRaised >= minimumRaise.

```typescript
await program.methods.graduatePool()
  .accounts({ caller, pool: poolPda, quoteMint, baseMint, baseVault, quoteVault, tokenProgram })
  .transaction();
```

### cancel_pool

Admin only.

```typescript
await program.methods.cancelPool().accounts({ admin, pool: poolPda }).transaction();
```

### setup_dao

After graduation. Creates SPL Governance realm with pool token as community mint.

```typescript
await program.methods.setupDao(realmName, votingBaseTime, new BN(minWeightToCreateProposal))
  .accounts({ admin, pool: poolPda, quoteMint, councilMint: councilMintKeypair.publicKey, ... })
  .signers([councilMintKeypair])
  .transaction();
```

## PDA Derivations

```typescript
// Pool
[Buffer.from('pool'), quoteMint.toBuffer()]

// Receipt
[Buffer.from('receipt'), poolPda.toBuffer(), receiptId.toBuffer()]
```

## Roles

| Role | Qualification | Permissions |
|------|---------------|-------------|
| Voter | Any contribution | Vote on proposals |
| Proposer | Contribution >= proposalThresholdPercent | Create + vote |

## Token Types

| Token | Purpose | Decimals |
|-------|---------|----------|
| Base (USDC) | Investment currency | 6 |
| Quote | Pool token (investors receive) | 6 |
| Council | Governance voting (proposers) | 0 |

## Error Codes

| Code | Name | Resolution |
|------|------|------------|
| 6000 | PoolNotActive | Check pool status |
| 6004 | PoolNotStarted | Wait for start time |
| 6005 | PoolEnded | Pool closed |
| 6007 | InvalidInvestmentAmount | Use positive amount |
| 6008 | ReceiptAlreadyClaimed | Already claimed |
| 6010 | InvalidCouncilSize | 1-8 members required |
| 6011 | InvalidTimeRange | endTime must be after startTime |
| 6014 | Unauthorized | Admin only |
| 6015 | StartTimeInPast | Use future start time |
| 6016 | PoolNotGraduated | Graduate pool first |

## Data Parsing

```typescript
// USDC: rawAmount / 1_000_000
// Timestamps: new Date(timestamp.toNumber() * 1000)
// Status: Object.keys(status)[0] → capitalize
```
