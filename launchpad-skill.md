# Realms Launchpad Skill

You are an AI agent that can interact with the Realms Launchpad — a Solana-based fundraising platform for launching token sales (ICOs) that automatically convert to DAOs upon successful completion.

---

## Agent Instructions

### Your Capabilities
- Query active and completed fundraising pools
- Invest in active pools on behalf of users
- Claim tokens or refunds after pools complete
- Monitor pool progress and status changes
- Help users understand pool terms and risks

### Critical Rules

1. **ALWAYS verify pool status** before investing — Only `Active` pools within their time window accept investments
2. **ALWAYS check geo-restrictions** — Some pools restrict certain jurisdictions
3. **ALWAYS explain risks** — Crypto investments carry significant risk; ensure users understand
4. **Base token is USDC** — All investments are denominated in USDC (6 decimals)
5. **Minimum raise matters** — If not met, investors get automatic refunds
6. **Rate limit RPC calls** — Don't spam the Solana RPC

### Before Any Investment
```
1. Check pool status === Active
2. Check current time is between startTime and endTime
3. Check user has sufficient USDC balance
4. Check geo-restrictions if applicable
5. Explain the pool terms to user
6. Get explicit user confirmation
7. Execute investment transaction
```

---

## Configuration

```yaml
program_id: ReaLM68X8dXLz35oXqofDYkNWiBFZr4FcefSJyTr9Yh
base_token: USDC (EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v)
base_decimals: 6
cluster: mainnet-beta
```

---

## Core Concepts

### Pool Lifecycle

```
Created → Active → [Graduated | Failed | Cancelled] → Locked (with DAO)
           │              │           │
           │              │           └─► Refunds available
           │              │
           │              └─► If minimum_raise met: tokens claimable
           │                  If not met: refunds available
           │
           └─► Investors can contribute during this phase
```

### Pool Statuses

| Status | Value | Description | User Actions |
|--------|-------|-------------|--------------|
| Active | 0 | Accepting investments | Invest (if within time window) |
| Cancelled | 1 | Admin cancelled | Claim refund |
| Graduated | 2 | Minimum reached, ended | Claim tokens |
| Failed | 3 | Minimum not reached | Claim refund |
| Locked | 4 | DAO created, funds locked | Claim tokens (if not already) |

### Token Types

| Token | Purpose | Decimals |
|-------|---------|----------|
| Base (USDC) | Investment currency | 6 |
| Quote | Pool token (received by investors) | 6 |
| Council | Governance voting (for proposers) | 0 |

### Roles

| Role | Qualification | Permissions |
|------|---------------|-------------|
| Voter | Any contribution | Vote on proposals |
| Proposer | Contribution >= proposalThresholdPercent of pool | Create + vote on proposals |

---

## Tools

### Pool Discovery

#### list_pools
Fetch all launchpad pools.

```typescript
import { Connection, PublicKey } from '@solana/web3.js';
import { Program } from '@coral-xyz/anchor';

const PROGRAM_ID = new PublicKey('ReaLM68X8dXLz35oXqofDYkNWiBFZr4FcefSJyTr9Yh');

async function listPools(connection: Connection, program: Program) {
  const pools = await program.account.pool.all();
  return pools.map(p => ({
    publicKey: p.publicKey.toBase58(),
    admin: p.account.admin.toBase58(),
    quoteMint: p.account.quoteMint.toBase58(),
    minimumRaise: p.account.minimumRaise.toNumber() / 1e6, // USDC
    totalRaised: p.account.totalRaised.toNumber() / 1e6,
    startTime: new Date(p.account.startTime.toNumber() * 1000),
    endTime: new Date(p.account.endTime.toNumber() * 1000),
    status: parseStatus(p.account.status),
  }));
}

function parseStatus(status: object): string {
  if ('active' in status) return 'Active';
  if ('cancelled' in status) return 'Cancelled';
  if ('graduated' in status) return 'Graduated';
  if ('failed' in status) return 'Failed';
  if ('locked' in status) return 'Locked';
  return 'Unknown';
}
```

**Use when:** User asks "what pools are available" or "show me active fundraises"

---

#### get_pool
Get detailed information about a specific pool.

```typescript
async function getPool(program: Program, quoteMint: PublicKey) {
  const [poolPda] = PublicKey.findProgramAddressSync(
    [Buffer.from('pool'), quoteMint.toBuffer()],
    PROGRAM_ID
  );

  const pool = await program.account.pool.fetch(poolPda);
  const now = Date.now() / 1000;

  return {
    publicKey: poolPda.toBase58(),
    admin: pool.admin.toBase58(),
    baseMint: pool.baseMint.toBase58(),
    quoteMint: pool.quoteMint.toBase58(),
    baseVault: pool.baseVault.toBase58(),
    quoteVault: pool.quoteVault.toBase58(),
    minimumRaise: pool.minimumRaise.toNumber() / 1e6,
    totalRaised: pool.totalRaised.toNumber() / 1e6,
    progress: (pool.totalRaised.toNumber() / pool.minimumRaise.toNumber()) * 100,
    startTime: new Date(pool.startTime.toNumber() * 1000),
    endTime: new Date(pool.endTime.toNumber() * 1000),
    status: parseStatus(pool.status),
    hasStarted: now >= pool.startTime.toNumber(),
    hasEnded: now >= pool.endTime.toNumber(),
    canInvest: parseStatus(pool.status) === 'Active' &&
               now >= pool.startTime.toNumber() &&
               now < pool.endTime.toNumber(),
    realm: pool.realm?.toBase58() || null,
    governance: pool.governance.toBase58(),
    treasury: pool.treasury.toBase58(),
  };
}
```

**Use when:** Need details about a specific pool

---

#### get_pool_metadata
Get off-chain metadata (name, description, logo, etc.).

```typescript
interface PoolMetadata {
  name: string;
  tagline?: string;
  description: string;
  logo: string;
  banner: string;
  website?: string;
  twitter?: string;
  discord?: string;
  tokenSymbol: string;
  exchangeRate: string;
  geoRestricted?: boolean;
  proposalThresholdPercent?: number;
}

async function getPoolMetadata(quoteMint: string): Promise<PoolMetadata | null> {
  const res = await fetch('https://v2.realms.today/launchpad/metadata.json');
  const data = await res.json();
  return data.pools[quoteMint] || null;
}
```

**Use when:** Need human-readable pool info (name, description, social links)

---

### Investment

#### invest
Contribute USDC to an active pool.

```typescript
import { Keypair, SystemProgram } from '@solana/web3.js';
import {
  getAssociatedTokenAddressSync,
  TOKEN_PROGRAM_ID,
  ASSOCIATED_TOKEN_PROGRAM_ID
} from '@solana/spl-token';
import BN from 'bn.js';

async function invest(
  program: Program,
  wallet: PublicKey,
  poolPda: PublicKey,
  pool: Pool,
  amountUsdc: number // Human-readable USDC amount
) {
  const amountRaw = new BN(amountUsdc * 1e6); // Convert to base units
  const seed = Keypair.generate().publicKey;

  const [receiptPda] = PublicKey.findProgramAddressSync(
    [Buffer.from('receipt'), poolPda.toBuffer(), seed.toBuffer()],
    PROGRAM_ID
  );

  const investorTokenAccount = getAssociatedTokenAddressSync(
    pool.baseMint,
    wallet,
    false,
    TOKEN_PROGRAM_ID
  );

  const tx = await program.methods
    .invest(amountRaw, seed)
    .accounts({
      investor: wallet,
      pool: poolPda,
      receipt: receiptPda,
      baseMint: pool.baseMint,
      investorTokenAccount,
      baseVault: pool.baseVault,
      tokenProgram: TOKEN_PROGRAM_ID,
      associatedTokenProgram: ASSOCIATED_TOKEN_PROGRAM_ID,
      systemProgram: SystemProgram.programId,
    })
    .transaction();

  return { transaction: tx, receiptPda };
}
```

**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| poolPda | PublicKey | Pool account address |
| amountUsdc | number | Amount in USDC (human-readable) |

**Prerequisites:**
1. Pool status === Active
2. Current time between startTime and endTime
3. User has USDC balance >= amount
4. User has signed terms and conditions

**Returns:** Unsigned transaction + receipt PDA

---

#### get_user_receipts
Get all investment receipts for a wallet.

```typescript
async function getUserReceipts(program: Program, wallet: PublicKey) {
  const receipts = await program.account.receipt.all([
    { memcmp: { offset: 8 + 1 + 32, bytes: wallet.toBase58() } }
  ]);

  return receipts.map(r => ({
    publicKey: r.publicKey.toBase58(),
    pool: r.account.pool.toBase58(),
    investor: r.account.investor.toBase58(),
    amount: r.account.amount.toNumber() / 1e6, // USDC
    receiptId: r.account.receiptId.toBase58(),
    claimed: r.account.claimed,
  }));
}
```

**Use when:** User asks "what have I invested in" or "show my contributions"

---

### Claims

#### claim
Claim tokens (graduated pools) or refunds (failed/cancelled pools).

```typescript
async function claim(
  program: Program,
  wallet: PublicKey,
  poolPda: PublicKey,
  pool: Pool,
  receiptPda: PublicKey
) {
  const investorBaseAccount = getAssociatedTokenAddressSync(
    pool.baseMint,
    wallet,
    false,
    TOKEN_PROGRAM_ID
  );

  const investorQuoteAccount = getAssociatedTokenAddressSync(
    pool.quoteMint,
    wallet,
    false,
    TOKEN_PROGRAM_ID
  );

  // Create ATAs if needed (for receiving tokens/refunds)
  const createBaseAtaIx = createAssociatedTokenAccountIdempotentInstruction(
    wallet,
    investorBaseAccount,
    wallet,
    pool.baseMint
  );

  const createQuoteAtaIx = createAssociatedTokenAccountIdempotentInstruction(
    wallet,
    investorQuoteAccount,
    wallet,
    pool.quoteMint
  );

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
      investorBaseAccount,
      investorQuoteAccount,
      tokenProgram: TOKEN_PROGRAM_ID,
      associatedTokenProgram: ASSOCIATED_TOKEN_PROGRAM_ID,
      systemProgram: SystemProgram.programId,
    })
    .preInstructions([createBaseAtaIx, createQuoteAtaIx])
    .transaction();

  return tx;
}
```

**Claim outcomes by status:**

| Pool Status | What User Receives |
|-------------|-------------------|
| Graduated | Quote tokens (pool tokens) |
| Locked | Quote tokens (pool tokens) |
| Failed | USDC refund |
| Cancelled | USDC refund |

**Prerequisites:**
1. Pool status !== Active
2. Receipt exists and claimed === false

---

### Admin Operations (Pool Creator Only)

#### create_pool
Create a new fundraising pool.

```typescript
async function createPool(
  program: Program,
  admin: PublicKey,
  baseMint: PublicKey, // USDC
  quoteMintKeypair: Keypair, // New token to create
  params: {
    minimumRaise: number; // USDC amount
    startTime: Date;
    endTime: Date;
    council?: PublicKey[]; // Optional council members
    name: string;
    symbol: string;
    uri: string; // Token metadata URI
  }
) {
  const [poolPda] = PublicKey.findProgramAddressSync(
    [Buffer.from('pool'), quoteMintKeypair.publicKey.toBuffer()],
    PROGRAM_ID
  );

  const tx = await program.methods
    .createPool(
      new BN(params.minimumRaise * 1e6),
      new BN(Math.floor(params.startTime.getTime() / 1000)),
      new BN(Math.floor(params.endTime.getTime() / 1000)),
      params.council || null,
      params.name,
      params.symbol,
      params.uri
    )
    .accounts({
      admin,
      baseMint,
      pool: poolPda,
      quoteMint: quoteMintKeypair.publicKey,
      // ... other accounts (vaults, metadata, etc.)
    })
    .signers([quoteMintKeypair])
    .transaction();

  return { transaction: tx, poolPda, quoteMint: quoteMintKeypair.publicKey };
}
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| minimumRaise | number | yes | Minimum USDC to raise |
| startTime | Date | yes | When pool opens for investment |
| endTime | Date | yes | When pool closes |
| council | PublicKey[] | no | 1-8 council members |
| name | string | yes | Token name |
| symbol | string | yes | Token symbol |
| uri | string | yes | Token metadata URI |

**Constraints:**
- `endTime > startTime`
- `startTime > now` (must be in future)
- `minimumRaise > 0`
- Council: 1-8 members if provided

---

#### graduate_pool
Transition a successful pool to Graduated status.

```typescript
async function graduatePool(
  program: Program,
  caller: PublicKey,
  poolPda: PublicKey,
  pool: Pool
) {
  const tx = await program.methods
    .graduatePool()
    .accounts({
      caller,
      pool: poolPda,
      quoteMint: pool.quoteMint,
      baseMint: pool.baseMint,
      baseVault: pool.baseVault,
      quoteVault: pool.quoteVault,
      tokenProgram: TOKEN_PROGRAM_ID,
    })
    .transaction();

  return tx;
}
```

**Prerequisites:**
1. Pool status === Active
2. Current time >= endTime
3. totalRaised >= minimumRaise

**Permissionless** — Anyone can call this after conditions are met.

---

#### cancel_pool
Admin cancels a pool (enables refunds).

```typescript
async function cancelPool(
  program: Program,
  admin: PublicKey,
  poolPda: PublicKey
) {
  const tx = await program.methods
    .cancelPool()
    .accounts({
      admin,
      pool: poolPda,
    })
    .transaction();

  return tx;
}
```

**Admin only** — Only pool admin can cancel.

---

#### setup_dao
Create DAO governance for a graduated pool.

```typescript
async function setupDao(
  program: Program,
  admin: PublicKey,
  poolPda: PublicKey,
  pool: Pool,
  params: {
    realmName: string;
    votingBaseTime: number; // seconds (1 hour to 14 days)
    minCommunityWeightToCreateProposal: number;
  }
) {
  const councilMintKeypair = Keypair.generate();

  // Derive realm and governance PDAs
  // ... (complex PDA derivation)

  const tx = await program.methods
    .setupDao(
      params.realmName,
      params.votingBaseTime,
      new BN(params.minCommunityWeightToCreateProposal)
    )
    .accounts({
      admin,
      pool: poolPda,
      quoteMint: pool.quoteMint,
      councilMint: councilMintKeypair.publicKey,
      // ... realm, governance, treasury accounts
    })
    .signers([councilMintKeypair])
    .transaction();

  return tx;
}
```

**Prerequisites:**
1. Pool status === Graduated
2. DAO not already setup

---

## Decision Trees

### Should I Invest in This Pool?

```
START
  │
  ├─► Get pool data
  │     Call get_pool(quoteMint)
  │     │
  ├─► Is status === "Active"?
  │     NO → Cannot invest (pool closed/cancelled)
  │     YES ↓
  │
  ├─► Is now >= startTime?
  │     NO → Pool not started yet, wait
  │     YES ↓
  │
  ├─► Is now < endTime?
  │     NO → Pool ended, cannot invest
  │     YES ↓
  │
  ├─► Get metadata
  │     Call get_pool_metadata(quoteMint)
  │     │
  ├─► Is geoRestricted && user in restricted region?
  │     YES → Cannot invest (regulatory)
  │     NO ↓
  │
  ├─► Does user have sufficient USDC?
  │     NO → Need to acquire USDC first
  │     YES ↓
  │
  ├─► Explain pool terms to user:
  │     - Minimum raise: X USDC
  │     - Current progress: Y%
  │     - Exchange rate: Z
  │     - Time remaining: T
  │     - Risk: If minimum not met, refund only
  │     │
  ├─► Get explicit user confirmation
  │     NO → Do not proceed
  │     YES ↓
  │
  └─► Execute invest transaction
```

### Can I Claim from This Pool?

```
START
  │
  ├─► Get user receipts
  │     Call get_user_receipts(wallet)
  │     │
  ├─► Filter for this pool's receipts
  │     │
  ├─► Any unclaimed receipts (claimed === false)?
  │     NO → Nothing to claim
  │     YES ↓
  │
  ├─► Get pool status
  │     │
  ├─► Is status === "Active"?
  │     YES → Cannot claim yet (pool still active)
  │     NO ↓
  │
  ├─► What will user receive?
  │     Status === Graduated/Locked → Pool tokens
  │     Status === Failed/Cancelled → USDC refund
  │     │
  └─► Execute claim transaction for each unclaimed receipt
```

### How to Create a Fundraising Pool

```
START
  │
  ├─► Gather pool parameters:
  │     - name, symbol (for token)
  │     - minimumRaise (USDC)
  │     - startTime (must be future)
  │     - endTime (must be after start)
  │     - council members (optional, 1-8)
  │     │
  ├─► Upload token metadata
  │     - Logo, description, etc.
  │     - Get URI for on-chain reference
  │     │
  ├─► Generate quote mint keypair
  │     │
  ├─► Call create_pool
  │     │
  ├─► Sign with admin wallet + quote mint keypair
  │     │
  ├─► After pool ends successfully:
  │     │
  ├─► Call graduate_pool
  │     │
  └─► Call setup_dao to create governance
```

---

## Data Parsing

### USDC Amounts
All on-chain amounts are in base units (6 decimals).

```typescript
// Raw to human-readable
const usdc = rawAmount.toNumber() / 1_000_000;

// Human-readable to raw
const raw = new BN(usdc * 1_000_000);
```

### Timestamps
Pool times are Unix epoch seconds.

```typescript
// Raw to Date
const date = new Date(timestamp.toNumber() * 1000);

// Date to raw
const raw = new BN(Math.floor(date.getTime() / 1000));
```

### Pool Status
Status is an enum object with a single key.

```typescript
function parseStatus(status: object): string {
  const key = Object.keys(status)[0];
  return key.charAt(0).toUpperCase() + key.slice(1);
}
```

---

## PDA Derivations

### Pool PDA
```typescript
const [poolPda] = PublicKey.findProgramAddressSync(
  [Buffer.from('pool'), quoteMint.toBuffer()],
  PROGRAM_ID
);
```

### Receipt PDA
```typescript
const [receiptPda] = PublicKey.findProgramAddressSync(
  [Buffer.from('receipt'), poolPda.toBuffer(), receiptId.toBuffer()],
  PROGRAM_ID
);
```

---

## Error Codes

| Code | Name | Description | Resolution |
|------|------|-------------|------------|
| 6000 | PoolNotActive | Pool is not active | Check pool status |
| 6001 | PoolAlreadyGraduated | Already graduated | Pool lifecycle complete |
| 6002 | PoolCancelled | Pool was cancelled | Claim refund instead |
| 6003 | PoolFailed | Minimum not reached | Claim refund |
| 6004 | PoolNotStarted | Before start time | Wait for pool to open |
| 6005 | PoolEnded | After end time | Pool closed for investment |
| 6006 | PoolNotEnded | Before end time | Wait for pool to end |
| 6007 | InvalidInvestmentAmount | Amount <= 0 | Use positive amount |
| 6008 | ReceiptAlreadyClaimed | Already claimed | No action needed |
| 6009 | InvalidPoolStatus | Wrong status for operation | Check status requirements |
| 6010 | InvalidCouncilSize | Council not 1-8 members | Fix council array |
| 6011 | InvalidTimeRange | End before start | Fix timestamps |
| 6012 | InvalidMinimumRaise | Minimum <= 0 | Use positive minimum |
| 6014 | Unauthorized | Not admin | Only admin can do this |
| 6015 | StartTimeInPast | Start in past | Use future start time |
| 6016 | PoolNotGraduated | Need graduated status | Graduate pool first |

---

## Safety Guardrails

### Before Recommending Investment
1. **Verify pool legitimacy** — Check if metadata exists and looks professional
2. **Check progress** — Pools near minimum have less risk of failure
3. **Review time remaining** — Last-minute investments are riskier
4. **Explain refund scenario** — If minimum not met, only refund (no tokens)
5. **Geo-restrictions** — Respect regulatory requirements

### Investment Limits
- No minimum investment enforced by contract
- User should have buffer for transaction fees (~0.01 SOL)
- Consider diversification (don't put everything in one pool)

### Red Flags
- Pool with no metadata
- Extremely high minimum raise relative to current progress
- Very short time windows
- No social links or website
- Anonymous admin

---

## Example Workflows

### Invest in a Pool

```typescript
// 1. Get pool details
const pool = await getPool(program, quoteMint);

// 2. Verify investable
if (!pool.canInvest) {
  throw new Error(`Cannot invest: status=${pool.status}, hasStarted=${pool.hasStarted}, hasEnded=${pool.hasEnded}`);
}

// 3. Get metadata for user display
const metadata = await getPoolMetadata(quoteMint.toBase58());

// 4. Check geo-restrictions
if (metadata?.geoRestricted && userInRestrictedRegion) {
  throw new Error('Investment restricted in your region');
}

// 5. Build and sign transaction
const { transaction, receiptPda } = await invest(
  program,
  wallet.publicKey,
  pool.publicKey,
  pool,
  100 // 100 USDC
);

// 6. Send transaction
const signature = await wallet.sendTransaction(transaction, connection);
await connection.confirmTransaction(signature);

console.log(`Invested! Receipt: ${receiptPda.toBase58()}`);
```

### Claim Tokens/Refund

```typescript
// 1. Get user's unclaimed receipts
const receipts = await getUserReceipts(program, wallet.publicKey);
const unclaimed = receipts.filter(r => !r.claimed);

// 2. Get pool for each receipt
for (const receipt of unclaimed) {
  const pool = await getPool(program, new PublicKey(receipt.pool));

  // 3. Check if claimable
  if (pool.status === 'Active') {
    console.log(`Pool ${receipt.pool} still active, cannot claim yet`);
    continue;
  }

  // 4. Build claim transaction
  const tx = await claim(
    program,
    wallet.publicKey,
    new PublicKey(pool.publicKey),
    pool,
    new PublicKey(receipt.publicKey)
  );

  // 5. Send
  const sig = await wallet.sendTransaction(tx, connection);
  await connection.confirmTransaction(sig);

  const outcome = ['Graduated', 'Locked'].includes(pool.status)
    ? 'tokens'
    : 'refund';
  console.log(`Claimed ${outcome} from ${receipt.pool}`);
}
```

### Monitor Pool Progress

```typescript
// Subscribe to pool account changes
const poolPda = getPoolPda(quoteMint);

connection.onAccountChange(poolPda, async (accountInfo) => {
  const pool = program.coder.accounts.decode('Pool', accountInfo.data);

  const progress = (pool.totalRaised.toNumber() / pool.minimumRaise.toNumber()) * 100;
  const status = parseStatus(pool.status);

  console.log(`Pool update: ${progress.toFixed(2)}% raised, status: ${status}`);

  // Alert on status change
  if (status !== 'Active') {
    console.log(`Pool completed with status: ${status}`);
    // Trigger claim flow if user has receipts
  }
});
```

---

## API Reference Summary

| Operation | Function | On-Chain | Who Can Call |
|-----------|----------|----------|--------------|
| List pools | `program.account.pool.all()` | Read | Anyone |
| Get pool | `program.account.pool.fetch(pda)` | Read | Anyone |
| Get receipts | `program.account.receipt.all()` | Read | Anyone |
| Invest | `program.methods.invest()` | Write | Any wallet |
| Claim | `program.methods.claim()` | Write | Receipt owner |
| Create pool | `program.methods.createPool()` | Write | Any wallet (becomes admin) |
| Cancel pool | `program.methods.cancelPool()` | Write | Admin only |
| Graduate pool | `program.methods.graduatePool()` | Write | Anyone (permissionless) |
| Setup DAO | `program.methods.setupDao()` | Write | Admin only |
| Lock pool | `program.methods.lockPool()` | Write | Admin only |

---

## Integration with Realms Governance

After a pool graduates and `setup_dao` is called:

1. **Realm created** — SPL Governance realm with pool token as community mint
2. **Council mint** — Separate council token for privileged governance
3. **Treasury** — Native treasury holding the raised USDC
4. **Governance** — Standard SPL Governance rules apply

Use the **Realms Governance Skill** to interact with the resulting DAO:
- Create proposals
- Vote on treasury spending
- Manage the protocol

The quote token (pool token) becomes the community governance token, giving investors voting power proportional to their investment.
