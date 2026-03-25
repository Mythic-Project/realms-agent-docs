# Sowellian Implementation Reference

## Key Accounts

| Account | Description | Seeds |
|---------|-------------|-------|
| Registrar | Plugin config per realm | `['registrar', realm, mint]` |
| Bet | Bet state (amounts, outcome, flags) | `['bet', betSeed]` |
| VoterWeightRecord | Voting power for governance | `['voter-weight-record', realm, mint, owner]` |
| VoteReceipt | Individual vote tracking | `['vote-receipt', bet, receiptSeed]` |
| BetVault | ATA of bet PDA for governing token | `ATA(bet, mint)` |

## Bet Fields

| Field | Type | Description |
|-------|------|-------------|
| seed | Pubkey | Random seed for all PDA derivations |
| bet_amount | u64 | Tokens locked by creator |
| owner | Pubkey | Bet creator |
| proposal | Pubkey | Linked governance proposal |
| yes_amount / no_amount | u64 | Total staked per side |
| is_settled | bool | Voting ended, amounts locked |
| is_cancelled | bool | Bet was cancelled |
| is_resolved | bool | Outcome determined |
| is_registered | bool | Creator's vote was registered |
| outcome | BetOutcome | Undecided / Yes / No |

## Transaction 1: Create Bet + Proposal + Signatory

```javascript
const { Connection, PublicKey, Keypair, Transaction, sendAndConfirmTransaction,
  TransactionInstruction, SystemProgram } = require('@solana/web3.js');
const { withCreateProposal, withAddSignatory, VoteType,
  getGovernanceProgramVersion, getTokenOwnerRecordAddress
} = require('@realms-today/spl-governance');
const { getAssociatedTokenAddress, TOKEN_PROGRAM_ID } = require('@solana/spl-token');

const GOV_PROGRAM = new PublicKey('GovER5Lthms3bLBqWub97yVrMmEogzX7xNjdXpPPCVZw');
const SOWELLIAN_PLUGIN = new PublicKey('sowEL1Rtn3p479rg34gW7mVPeCNY58Es5rkLpFsCJAW');
const ATA_PROGRAM = new PublicKey('ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL');

// Derive PDAs from a single random seed
const betSeed = Keypair.generate().publicKey;

const [registrar] = PublicKey.findProgramAddressSync(
  [Buffer.from('registrar'), realm.toBuffer(), mint.toBuffer()], SOWELLIAN_PLUGIN);
const [voterWeightRecord] = PublicKey.findProgramAddressSync(
  [Buffer.from('voter-weight-record'), realm.toBuffer(), mint.toBuffer(), wallet.publicKey.toBuffer()], SOWELLIAN_PLUGIN);
const [bet] = PublicKey.findProgramAddressSync(
  [Buffer.from('bet'), betSeed.toBuffer()], SOWELLIAN_PLUGIN);
const betVault = await getAssociatedTokenAddress(mint, bet, true);
const ownerTokenAccount = await getAssociatedTokenAddress(mint, wallet.publicKey);
const tokenOwnerRecord = await getTokenOwnerRecordAddress(GOV_PROGRAM, realm, mint, wallet.publicKey);
const [proposal] = PublicKey.findProgramAddressSync(
  [Buffer.from('governance'), governance.toBuffer(), mint.toBuffer(), betSeed.toBuffer()], GOV_PROGRAM);

const instructions = [];

// 1. createBet
const CREATE_BET_DISCRIMINATOR = Buffer.from([197, 42, 153, 2, 59, 63, 143, 246]);
instructions.push(new TransactionInstruction({
  programId: SOWELLIAN_PLUGIN,
  keys: [
    { pubkey: registrar, isSigner: false, isWritable: false },
    { pubkey: bet, isSigner: false, isWritable: true },
    { pubkey: realm, isSigner: false, isWritable: false },
    { pubkey: mint, isSigner: false, isWritable: false },
    { pubkey: governance, isSigner: false, isWritable: false },
    { pubkey: proposal, isSigner: false, isWritable: false },
    { pubkey: ownerTokenAccount, isSigner: false, isWritable: true },
    { pubkey: betVault, isSigner: false, isWritable: true },
    { pubkey: voterWeightRecord, isSigner: false, isWritable: true },
    { pubkey: wallet.publicKey, isSigner: true, isWritable: true },
    { pubkey: TOKEN_PROGRAM_ID, isSigner: false, isWritable: false },
    { pubkey: ATA_PROGRAM, isSigner: false, isWritable: false },
    { pubkey: SystemProgram.programId, isSigner: false, isWritable: false },
  ],
  data: Buffer.concat([CREATE_BET_DISCRIMINATOR, betSeed.toBuffer()]),
}));

// 2. createProposal (uses VWR set by createBet — MUST be same tx)
const version = await getGovernanceProgramVersion(connection, GOV_PROGRAM);
await withCreateProposal(
  instructions, GOV_PROGRAM, version, realm, governance, tokenOwnerRecord,
  title, description, mint, wallet.publicKey,
  undefined, VoteType.SINGLE_CHOICE, ['Approve'], true,
  wallet.publicKey, voterWeightRecord, betSeed
);

// 3. addSignatory
await withAddSignatory(
  instructions, GOV_PROGRAM, version, proposal,
  tokenOwnerRecord, wallet.publicKey, wallet.publicKey, wallet.publicKey
);

const tx1 = new Transaction().add(...instructions);
await sendAndConfirmTransaction(connection, tx1, [wallet]);
```

## Transaction 2: Insert Instructions + Sign Off

```javascript
const { withInsertTransaction, withSignOffProposal,
  getSignatoryRecordAddress } = require('@realms-today/spl-governance');

const insertIxs = [];

await withInsertTransaction(
  insertIxs, GOV_PROGRAM, version, governance, proposal,
  tokenOwnerRecord, wallet.publicKey,
  0, 0, 0, // optionIndex, instructionIndex, holdUpTime
  serializedInstructions, // the governance instructions to execute if passed
  wallet.publicKey
);

const signatoryRecord = await getSignatoryRecordAddress(GOV_PROGRAM, proposal, wallet.publicKey);
withSignOffProposal(
  insertIxs, GOV_PROGRAM, version, realm, governance,
  proposal, wallet.publicKey, signatoryRecord, undefined
);

const tx2 = new Transaction().add(...insertIxs);
await sendAndConfirmTransaction(connection, tx2, [wallet]);
```

## Transaction 3: Register Vote (CRITICAL)

```javascript
const { getRealmConfigAddress, getVoteRecordAddress } = require('@realms-today/spl-governance');

const [voteReceipt] = PublicKey.findProgramAddressSync(
  [Buffer.from('vote-receipt'), bet.toBuffer(), betSeed.toBuffer()], SOWELLIAN_PLUGIN);
const [voteVwr] = PublicKey.findProgramAddressSync(
  [Buffer.from('voter-weight-record'), realm.toBuffer(), mint.toBuffer(), voteReceipt.toBuffer()], SOWELLIAN_PLUGIN);
const voterTor = await getTokenOwnerRecordAddress(GOV_PROGRAM, realm, mint, voteReceipt);
const voteRecord = await getVoteRecordAddress(GOV_PROGRAM, proposal, voterTor);
const realmConfig = await getRealmConfigAddress(GOV_PROGRAM, realm);

const tx3 = await program.methods.registerVote()
  .accountsPartial({
    registrar, bet, voteReceipt, voteVwr, realm,
    proposal, governance, proposalOwnerRecord: tokenOwnerRecord,
    voterTokenOwnerRecord: voterTor, governingTokenMint: mint,
    voteRecord, realmConfig, owner: wallet.publicKey,
    governanceProgram: GOV_PROGRAM,
  })
  .transaction();

await sendAndConfirmTransaction(connection, tx3, [wallet]);
```

## Settle, Resolve, Withdraw

### settleBet (permissionless, after voting ends)

```javascript
const [ownerReceipt] = PublicKey.findProgramAddressSync(
  [Buffer.from('vote-receipt'), bet.toBuffer(), betSeed.toBuffer()], SOWELLIAN_PLUGIN);
const registrarData = await program.account.registrar.fetch(registrar);

await program.methods.settleBet()
  .accountsPartial({
    registrar, bet: betPubkey, ownerReceipt,
    treasury: registrarData.treasury, realm,
    proposal, governingTokenMint: mint, settler: wallet.publicKey,
  })
  .transaction();
```

Requirement: owner's vote receipt MUST exist (`registerVote` must have been called).

### resolveBet (realm authority only)

```javascript
await program.methods.resolveBet({ yes: {} }) // or { no: {} }
  .accountsPartial({
    registrar, bet: betPubkey, betVault, treasury,
    treasuryTokenAccount, governingTokenMint: mint,
    realmAuthority: authority,
  })
  .transaction();
```

### withdrawPayoff (winners)

```javascript
await program.methods.withdrawPayoff()
  .accountsPartial({
    registrar, bet: betPubkey, voteReceipt: receiptPubkey,
    voterTokenAccount, voter: wallet.publicKey,
  })
  .transaction();
```

## Jupiter Limit Orders as Governance Instructions

For trade proposals, use Jupiter's Trigger API:

```javascript
const response = await fetch('https://api.jup.ag/trigger/v1/createOrder', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'x-api-key': 'YOUR_KEY' },
  body: JSON.stringify({
    maker: treasuryWallet, payer: treasuryWallet,
    inputMint: 'EPjFWdd5...', outputMint: 'TOKEN_MINT',
    params: {
      makingAmount: '1000000000', takingAmount: '5000000000',
      expiredAt: String(Math.floor(Date.now() / 1000) + 7 * 86400),
    },
  }),
});

// Deserialize VersionedTransaction, resolve lookup tables, extract instructions
// Filter out ComputeBudget instructions — governance doesn't need them
const govInstructions = jupiterInstructions
  .filter(ix => !ix.programId.equals(ComputeBudgetProgram.programId))
  .map(ix => ({ programId: ix.programId, accounts: ix.keys, data: ix.data }));
```

Use `https://api.jup.ag/trigger/v1/createOrder` (NOT `limit/v2`).

## VoterWeightRecord (VWR) Notes

- `createVoterWeightRecord` — creates empty VWR (weight=0, expiry=0)
- `createBet` — updates VWR with proper weight/expiry for `CreateProposal` action
- VWR expires at current slot — only valid within same transaction
- Common mistake: creating VWR in one tx, proposal in another → VWR expired

## Discriminators

| Instruction | Discriminator |
|------------|---------------|
| create_bet | `[197, 42, 153, 2, 59, 63, 143, 246]` |
| create_voter_weight_record | `[184, 249, 133, 178, 88, 152, 250, 186]` |
| register_vote | `[115, 72, 69, 208, 153, 57, 252, 35]` |
| settle_bet | `[115, 55, 234, 177, 227, 4, 10, 67]` |
| withdraw_payoff | `[46, 190, 15, 253, 65, 113, 31, 144]` |

## Error Reference

| Code | Name | Cause |
|------|------|-------|
| 3012 (0xBC4) | AccountNotInitialized | Vote receipt missing — `registerVote` never called |
| 6013 (0x177D) | ProposalNotLive | `registerVote` on cancelled/completed proposal |
| 0x44D | VWR expired | VWR created in previous tx/slot |
| 0x1 | InsufficientFunds | Not enough governing tokens |

## Checklist

- [ ] Sufficient governing tokens + SOL (~0.02)
- [ ] TokenOwnerRecord exists
- [ ] VoterWeightRecord exists (create if not)
- [ ] **TX1:** createBet + createProposal + addSignatory (ONE transaction)
- [ ] **TX2:** insertTransaction + signOffProposal
- [ ] **TX3:** registerVote (**CRITICAL**)
- [ ] Verify bet.is_registered === true after TX3
