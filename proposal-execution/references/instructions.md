# Proposal Instruction Reference

All instructions execute from the **governance treasury** address. Derive it with `getNativeTreasuryAddress(governancePk, governanceProgramId)` — this is the PDA that acts as signer/authority.

```typescript
import { PublicKey } from '@solana/web3.js';

const GOV_PROGRAM = new PublicKey('GovER5Lthms3bLBqWub97yVrMmEogzX7xNjdXpPPCVZw');

const [nativeTreasury] = PublicKey.findProgramAddressSync(
  [Buffer.from('native-treasury'), governancePk.toBuffer()],
  GOV_PROGRAM
);
// nativeTreasury is the treasury wallet address
```

## Token Transfer (SOL)

```typescript
import { SystemProgram, PublicKey } from '@solana/web3.js';

// Native SOL transfer from treasury
// Use BigNumber for safe decimal conversion (avoid floating-point errors)
import BigNumber from 'bignumber.js';

const ix = SystemProgram.transfer({
  fromPubkey: new PublicKey(treasuryWallet),
  toPubkey: new PublicKey(recipientAddress),
  lamports: BigInt(new BigNumber(amount).multipliedBy(1e9).toFixed(0)), // SOL has 9 decimals
});
```

## Token Transfer (SPL)

```typescript
import {
  createTransferCheckedInstruction,
  createAssociatedTokenAccountInstruction,
  getAssociatedTokenAddressSync,
} from '@solana/spl-token';

const mint = new PublicKey(tokenMint);
const treasury = new PublicKey(treasuryWallet);
const recipient = new PublicKey(recipientAddress);
const programId = new PublicKey(tokenProgramId); // TOKEN_PROGRAM_ID or TOKEN_2022_PROGRAM_ID

// Source: treasury's token account. For standard ATAs:
const sourceAta = getAssociatedTokenAddressSync(mint, treasury, true, programId);
// Note: for tokens where treasury holds in a non-ATA account, use the specific
// token account address from the treasury's on-chain state instead.

// Destination: recipient's ATA
const destAta = getAssociatedTokenAddressSync(mint, recipient, true, programId);

const ixs = [];

// Create destination ATA if it doesn't exist (treasury pays)
const destInfo = await connection.getAccountInfo(destAta);
if (!destInfo) {
  ixs.push(createAssociatedTokenAccountInstruction(
    treasury, destAta, recipient, mint, programId
  ));
}

// Transfer with decimal check — use BigNumber for safe amount conversion
ixs.push(createTransferCheckedInstruction(
  sourceAta, mint, destAta, treasury,
  BigInt(new BigNumber(amount).multipliedBy(new BigNumber(10).pow(decimals)).toFixed(0)),
  decimals, undefined, programId
));
```

## Burn Tokens

```typescript
import { createBurnCheckedInstruction, getAssociatedTokenAddressSync } from '@solana/spl-token';

const tokenAccount = getAssociatedTokenAddressSync(mint, treasury, true, programId);

const ix = createBurnCheckedInstruction(
  tokenAccount, mint, treasury,
  BigInt(new BigNumber(amount).multipliedBy(new BigNumber(10).pow(decimals)).toFixed(0)),
  decimals, undefined, programId
);
```

## Jupiter Limit Order

Uses Jupiter Trigger API v1. Returns a VersionedTransaction that must be deserialized to extract inner instructions.

```typescript
// 1. Get limit order tx from Jupiter
const response = await fetch('https://api.jup.ag/trigger/v1/createOrder', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': JUPITER_API_KEY, // Required
  },
  body: JSON.stringify({
    maker: treasuryWallet,
    payer: treasuryWallet,
    inputMint: inputMintAddress,
    outputMint: outputMintAddress,
    params: {
      makingAmount: String(inputAmount), // raw amount with decimals
      takingAmount: String(outputAmount),
      expiredAt: expiry ? String(Math.floor(expiry / 1000)) : undefined,
    },
  }),
});

const { transaction } = await response.json();

// 2. Deserialize VersionedTransaction → extract instructions
const vtx = VersionedTransaction.deserialize(Buffer.from(transaction, 'base64'));
const message = TransactionMessage.decompile(vtx.message, {
  addressLookupTableAccounts: await resolveLookupTables(connection, vtx),
});

// 3. Filter out ComputeBudget instructions
const govInstructions = message.instructions.filter(
  ix => !ix.programId.equals(ComputeBudgetProgram.programId)
);
```

**Important:** Use `https://api.jup.ag/trigger/v1/createOrder` (NOT `limit/v2`). Always include `x-api-key` header.

## Jupiter DCA (Recurring Order)

```typescript
// Uses Jupiter DCA program
// See createJupiterRecurringInstruction in realms-ui for full implementation
// Key params: inputMint, outputMint, inAmount, numberOfOrders, interval (seconds)
```

## Kamino Lending

Requires multi-step setup: init obligation → refresh reserve → refresh obligation → deposit.

```typescript
// 1. Check if obligation exists, create if needed
const initSets = await createKaminoInitInstructionSets(treasuryWallet, connection);

// 2. Create deposit instructions (includes refresh prereqs)
const lendIxs = await createKaminoLendInstructions(treasuryWallet, {
  kaminoAction: 'lend',
  tokenType: 'USDC', // or 'SOL', etc.
  amount: '1000',
});

// Each becomes a separate governance proposal transaction
// Prereqs (refresh) must execute in same Solana tx as deposit
```

**Key lesson:** Reserve AND obligation must be refreshed in the same slot as the deposit. These go as prerequisite instructions in the execution transaction, not as separate governance transactions.

## Drift Perp Trading

Requires account initialization first, then deposit, then trade.

```typescript
// 1. Init Drift account (one-time)
const initIx = await createDriftInitAccountInstruction(connection, treasuryWallet);

// 2. Deposit collateral
const depositIx = await createDriftDepositInstruction(connection, treasuryWallet, {
  amount: '1000', // USDC
});

// 3. Open perp position
const tradeIx = await createDriftPerpTradeInstruction(connection, treasuryWallet, {
  marketIndex: 0,       // SOL-PERP
  marketLabel: 'SOL-PERP',
  direction: 'long',
  size: '10',           // SOL amount
  orderType: 'market',  // or 'limit'
  entryPrice: null,     // for market orders
  stopLossPrice: null,
  takeProfitPrice: null,
  leverage: '5',
  collateral: '1000',
});
```

## Validator Staking

```typescript
import { StakeProgram, Keypair } from '@solana/web3.js';

// Create stake account + delegate to validator
const stakeKeypair = Keypair.generate();
const ixs = StakeProgram.createAccount({
  fromPubkey: treasury,
  stakePubkey: stakeKeypair.publicKey,
  authorized: {
    staker: treasury,
    withdrawer: treasury,
  },
  lamports: amount * 1e9,
});

const delegateIx = StakeProgram.delegate({
  stakePubkey: stakeKeypair.publicKey,
  authorizedPubkey: treasury,
  votePubkey: new PublicKey(validatorVoteAccount),
});
```

## Update Council Members

### Add Member

```typescript
import { createMintToInstruction, getAssociatedTokenAddressSync } from '@solana/spl-token';

// Mint 1 council token to new member
const councilMint = realm.account.config.councilMint;
const memberAta = getAssociatedTokenAddressSync(councilMint, newMemberWallet, true);

const ixs = [
  createAssociatedTokenAccountInstruction(treasury, memberAta, newMemberWallet, councilMint),
  createMintToInstruction(councilMint, memberAta, treasury, 1),
];
```

### Remove Member

```typescript
// Burn council token from member (requires governance authority as mint authority)
```

## Edit Wallet Rules (Governance Config)

Changes governance parameters: voting thresholds, quorum, hold-up time, tipping.

```typescript
import { withSetGovernanceConfig } from '@realms-today/spl-governance';

const ixs = [];
await withSetGovernanceConfig(
  ixs, programId, programVersion, governance,
  {
    communityVoteThreshold: { type: 0, value: 60 }, // 60% to pass
    councilVoteThreshold: { type: 0, value: 50 },
    minCommunityTokensToCreateProposal: new BN(1000000),
    minCouncilTokensToCreateProposal: new BN(1),
    baseVotingTime: 3 * 86400, // 3 days in seconds
    communityVoteTipping: { type: 0 }, // disabled=0, strict=1, early=2
    councilVoteTipping: { type: 2 }, // early tipping
    minInstructionHoldUpTime: 0,
    councilVetoVoteThreshold: { type: 0, value: 50 },
    communityVetoVoteThreshold: { type: 2 }, // disabled
    votingCoolOffTime: 0,
    depositExemptProposalCount: 10,
  }
);
```

## Custom Instruction (Arbitrary)

For any Solana instruction not covered by built-in types:

```typescript
import { serializeInstructionToBase64, getInstructionDataFromBase64 } from '@realms-today/spl-governance';

// Build any TransactionInstruction
const customIx = new TransactionInstruction({
  programId: new PublicKey('PROGRAM_ID'),
  keys: [
    { pubkey: account1, isSigner: false, isWritable: true },
    { pubkey: treasury, isSigner: true, isWritable: true }, // treasury is signer
  ],
  data: Buffer.from([...]),
});

// Serialize to base64 for governance
const serialized = serializeInstructionToBase64(customIx);
// Then: getInstructionDataFromBase64(serialized) → InstructionData for withInsertTransaction
```

## Token Streaming (Streamflow)

```typescript
// Uses Streamflow SDK
// Creates vesting/payment streams from treasury
// Key params: recipient, mint, totalAmount, startTime, schedule, cancelable
// Schedule options: per-second, per-minute, hourly, daily, weekly, monthly, yearly, cliff
```

## Add DAO Metadata

Fee: 0.5 SOL (first time only). Creates on-chain metadata for the DAO.

```typescript
// Fields: displayName (max 30), shortDescription (max 120),
//         daoImage (512x512 arweave URL), bannerImage (1200x200),
//         category, website, twitter, discord
```
