# Sowellian Betting Skill

You are an AI agent that can create and manage Sowellian betting proposals on Solana DAOs through the Realms governance platform. Sowellian is a prediction market plugin where proposals are bets — users stake tokens to vote Yes/No, and winners get paid from losers.

---

## Agent Instructions

### Your Capabilities
- Create betting proposals (bets) with attached governance instructions
- Settle bets after voting ends
- Withdraw winnings from resolved bets
- Monitor bet status and outcomes

### Critical Rules

1. **NEVER skip `registerVote`** — After creating a bet + proposal, you MUST call `registerVote` in a follow-up transaction. Without it, the bet creator's tokens are **permanently locked** with no recovery path.
2. **ALWAYS use the correct transaction order** — The 3-transaction flow is mandatory. Do not combine or skip steps.
3. **`createBet` locks tokens immediately** — The bet amount is transferred from your token account to the bet vault. Ensure you have sufficient balance.
4. **Proposals need instructions** — A proposal without governance instructions (e.g., a Jupiter swap) is just an empty shell. Always attach the execution instructions.
5. **VoterWeightRecord expires per-slot** — The VWR created by `createBet` is only valid in the same transaction. Never try to create VWR in one tx and use it in another.
6. **Use the same `betSeed` everywhere** — The seed (a random PublicKey) derives the bet PDA, proposal PDA, and vote receipt PDA. They must all match.

### ⚠️ Known Bug: Cancelled Bets Without `registerVote`

If a proposal is cancelled BEFORE `registerVote` is called, the bet tokens are **permanently locked**. There is no `cancel_bet` or `refund_bet` instruction. The `settle_bet` instruction requires the owner's vote receipt, which is created by `registerVote`. If the receipt doesn't exist, settlement fails.

**Prevention:** Always execute all 3 transactions atomically. If TX1 succeeds but TX2/TX3 fail, retry them immediately before the proposal can be cancelled.

---

## Configuration

```yaml
program_id: sowEL1Rtn3p479rg34gW7mVPeCNY58Es5rkLpFsCJAW
governance_program: GovER5Lthms3bLBqWub97yVrMmEogzX7xNjdXpPPCVZw
cluster: mainnet-beta
```

---

## Core Concepts

### Bet Lifecycle

```
createBet → registerVote → [Voting Period] → settleBet → resolveBet → withdrawPayoff
    │            │                                │            │             │
    │            │                                │            │             └─► Winners claim tokens
    │            │                                │            └─► Authority sets outcome (Yes/No)
    │            │                                └─► Anyone can settle after voting ends
    │            └─► Creator auto-votes Yes, creates vote receipt
    └─► Locks tokens in bet vault, creates VWR for proposal creation
```

### Key Accounts

| Account | Description | Seeds |
|---------|-------------|-------|
| Registrar | Plugin config per realm | `['registrar', realm, mint]` |
| Bet | Bet state (amounts, outcome, flags) | `['bet', betSeed]` |
| VoterWeightRecord | Voting power for governance | `['voter-weight-record', realm, mint, owner]` |
| VoteReceipt | Individual vote tracking | `['vote-receipt', bet, receiptSeed]` |
| BetVault | ATA of bet PDA for governing token | ATA(bet, mint) |

### Bet Fields

| Field | Type | Description |
|-------|------|-------------|
| seed | Pubkey | Random seed used to derive all PDAs |
| bet_amount | u64 | Tokens locked by creator |
| owner | Pubkey | Bet creator |
| proposal | Pubkey | Linked governance proposal |
| yes_amount | u64 | Total Yes-side tokens |
| no_amount | u64 | Total No-side tokens |
| is_settled | bool | Voting ended, amounts locked |
| is_cancelled | bool | Bet was cancelled |
| is_resolved | bool | Outcome determined |
| is_registered | bool | Creator's vote was registered |
| outcome | BetOutcome | Undecided / Yes / No |

---

## Complete Proposal Creation Flow

### Prerequisites
- Wallet with SOL for fees (~0.02 SOL for 3 transactions)
- Governing token (e.g., IC tokens) for the bet stake
- TokenOwnerRecord must exist (created on first interaction with the DAO)

### Transaction 1: Create Bet + Proposal + Add Signatory

This transaction does three things atomically:
1. `createBet` — locks tokens, creates bet PDA, updates VWR
2. `createProposal` — creates the governance proposal using the VWR
3. `addSignatory` — adds the creator as signatory

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

// --- Derive all PDAs from a single random seed ---
const betSeed = Keypair.generate().publicKey;

const [registrar] = PublicKey.findProgramAddressSync(
  [Buffer.from('registrar'), realm.toBuffer(), mint.toBuffer()],
  SOWELLIAN_PLUGIN
);

const [voterWeightRecord] = PublicKey.findProgramAddressSync(
  [Buffer.from('voter-weight-record'), realm.toBuffer(), mint.toBuffer(), wallet.publicKey.toBuffer()],
  SOWELLIAN_PLUGIN
);

const [bet] = PublicKey.findProgramAddressSync(
  [Buffer.from('bet'), betSeed.toBuffer()],
  SOWELLIAN_PLUGIN
);

const betVault = await getAssociatedTokenAddress(mint, bet, true);
const ownerTokenAccount = await getAssociatedTokenAddress(mint, wallet.publicKey);
const tokenOwnerRecord = await getTokenOwnerRecordAddress(GOV_PROGRAM, realm, mint, wallet.publicKey);

// Proposal PDA uses betSeed as the proposal seed
const [proposal] = PublicKey.findProgramAddressSync(
  [Buffer.from('governance'), governance.toBuffer(), mint.toBuffer(), betSeed.toBuffer()],
  GOV_PROGRAM
);

const instructions = [];

// 1. createBet instruction
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

// 2. createProposal (uses VWR set by createBet)
const version = await getGovernanceProgramVersion(connection, GOV_PROGRAM);
const proposalAddress = await withCreateProposal(
  instructions, GOV_PROGRAM, version, realm, governance, tokenOwnerRecord,
  title, description, mint, wallet.publicKey,
  undefined, // proposalIndex (auto)
  VoteType.SINGLE_CHOICE, ['Approve'], true, // useDenyOption
  wallet.publicKey, // payer
  voterWeightRecord, // plugin VWR
  betSeed // proposal seed = bet seed
);

// 3. addSignatory
await withAddSignatory(
  instructions, GOV_PROGRAM, version, proposalAddress,
  tokenOwnerRecord, wallet.publicKey, wallet.publicKey, wallet.publicKey
);

// Send TX1
const tx1 = new Transaction().add(...instructions);
await sendAndConfirmTransaction(connection, tx1, [wallet]);
```

### Transaction 2: Insert Governance Instructions + Sign Off

Insert the actual on-chain instructions (e.g., Jupiter limit order) that will execute if the proposal passes. Then sign off to start voting.

```javascript
const { withInsertTransaction, withSignOffProposal,
  getSignatoryRecordAddress } = require('@realms-today/spl-governance');

// --- Insert instructions ---
const insertIxs = [];

// serializedInstructions = array of { programId, accounts, data }
// These are the instructions the DAO treasury will execute if proposal passes
await withInsertTransaction(
  insertIxs, GOV_PROGRAM, version, governance, proposalAddress,
  tokenOwnerRecord, wallet.publicKey,
  0, // optionIndex
  0, // instructionIndex
  0, // holdUpTime (seconds)
  serializedInstructions, // the actual governance instructions
  wallet.publicKey
);

// --- Sign off (starts voting) ---
const signatoryRecord = await getSignatoryRecordAddress(
  GOV_PROGRAM, proposalAddress, wallet.publicKey
);
withSignOffProposal(
  insertIxs, GOV_PROGRAM, version, realm, governance,
  proposalAddress, wallet.publicKey, signatoryRecord, undefined
);

const tx2 = new Transaction().add(...insertIxs);
await sendAndConfirmTransaction(connection, tx2, [wallet]);
```

### Transaction 3: Register Vote (CRITICAL — DO NOT SKIP)

This creates the owner's vote receipt and auto-casts a Yes vote. **Without this step, the bet creator's tokens are permanently locked.**

```javascript
const { getRealmConfigAddress, getVoteRecordAddress } = require('@realms-today/spl-governance');
const { AnchorProvider, Program } = require('@coral-xyz/anchor');

const idl = require('./sowellian-idl.json'); // Sowellian plugin IDL
const provider = new AnchorProvider(connection, walletAdapter, {});
const program = new Program(idl, provider);

// Vote receipt PDA — uses bet PDA + bet seed
const [voteReceipt] = PublicKey.findProgramAddressSync(
  [Buffer.from('vote-receipt'), bet.toBuffer(), betSeed.toBuffer()],
  SOWELLIAN_PLUGIN
);

// VWR for the vote receipt (not the owner!)
const [voteVwr] = PublicKey.findProgramAddressSync(
  [Buffer.from('voter-weight-record'), realm.toBuffer(), mint.toBuffer(), voteReceipt.toBuffer()],
  SOWELLIAN_PLUGIN
);

// TOR for the vote receipt
const voterTor = await getTokenOwnerRecordAddress(
  GOV_PROGRAM, realm, mint, voteReceipt
);

// Governance vote record
const voteRecord = await getVoteRecordAddress(
  GOV_PROGRAM, proposalAddress, voterTor
);

const realmConfig = await getRealmConfigAddress(GOV_PROGRAM, realm);

const tx3 = await program.methods.registerVote()
  .accountsPartial({
    registrar,
    bet,
    voteReceipt,
    voteVwr,
    realm,
    proposal: proposalAddress,
    governance,
    proposalOwnerRecord: tokenOwnerRecord,
    voterTokenOwnerRecord: voterTor,
    governingTokenMint: mint,
    voteRecord,
    realmConfig,
    owner: wallet.publicKey,
    governanceProgram: GOV_PROGRAM,
  })
  .transaction();

tx3.feePayer = wallet.publicKey;
tx3.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
tx3.sign(wallet);
await connection.sendRawTransaction(tx3.serialize());
```

---

## Settling a Bet

After the voting period ends (or proposal is cancelled/completed), anyone can settle:

```javascript
const [ownerReceipt] = PublicKey.findProgramAddressSync(
  [Buffer.from('vote-receipt'), bet.toBuffer(), betSeed.toBuffer()],
  SOWELLIAN_PLUGIN
);

// Fetch registrar to get treasury address
const registrarData = await program.account.registrar.fetch(registrar);
const treasury = registrarData.treasury;

const tx = await program.methods.settleBet()
  .accountsPartial({
    registrar,
    bet: betPubkey,
    ownerReceipt,
    treasury,
    realm,
    proposal: proposalAddress,
    governingTokenMint: mint,
    settler: wallet.publicKey,
  })
  .transaction();
```

**Requirement:** The owner's vote receipt MUST exist (created by `registerVote`). If it doesn't, `settleBet` fails with `AccountNotInitialized`.

---

## Resolving a Bet

Only the realm authority can resolve (set outcome):

```javascript
const tx = await program.methods.resolveBet({ yes: {} }) // or { no: {} }
  .accountsPartial({
    registrar,
    bet: betPubkey,
    betVault,
    treasury,
    treasuryTokenAccount,
    governingTokenMint: mint,
    realmAuthority: authority, // must be signer
  })
  .transaction();
```

---

## Withdrawing Winnings

After resolution, winners can withdraw:

```javascript
const tx = await program.methods.withdrawPayoff()
  .accountsPartial({
    registrar,
    bet: betPubkey,
    voteReceipt: receiptPubkey,
    voterTokenAccount, // recipient's ATA for governing token
    voter: wallet.publicKey,
  })
  .transaction();
```

---

## Attaching Jupiter Limit Orders

For trade proposals (buy/sell tokens from treasury), use Jupiter's Trigger API:

```javascript
// 1. Get limit order transaction from Jupiter
const response = await fetch('https://api.jup.ag/trigger/v1/createOrder', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': 'YOUR_JUPITER_API_KEY'
  },
  body: JSON.stringify({
    maker: treasuryWallet,      // DAO treasury address
    payer: treasuryWallet,      // same as maker for DAOs
    inputMint: 'EPjFWdd5...',   // USDC mint
    outputMint: 'TOKEN_MINT',   // target token
    params: {
      makingAmount: '1000000000',  // raw amount with decimals
      takingAmount: '5000000000',  // raw amount with decimals
      expiredAt: String(Math.floor(Date.now() / 1000) + 7 * 86400), // 7 days
    },
  }),
});

// 2. Deserialize the versioned transaction to extract instructions
// Jupiter returns a VersionedTransaction — you must resolve address lookup
// tables and extract the inner instructions for governance
const data = await response.json();
const jupiterInstructions = await deserializeVersionedTransaction(
  connection, data.transaction
);

// 3. Insert as governance instructions (TX2 above)
// Filter out ComputeBudget instructions — governance doesn't need them
const govInstructions = jupiterInstructions
  .filter(ix => !ix.programId.equals(ComputeBudgetProgram.programId))
  .map(ix => ({
    programId: ix.programId,
    accounts: ix.keys,
    data: ix.data,
  }));
```

**Important:** Use `https://api.jup.ag/trigger/v1/createOrder` (NOT `limit/v2`). The trigger API is the current endpoint for limit orders.

---

## VoterWeightRecord (VWR)

The Sowellian plugin uses a custom VWR to give voting power:

1. **`createVoterWeightRecord`** — Creates an empty VWR (weight=0, expiry=0). Not useful alone.
2. **`createBet`** — Updates the VWR with proper weight and expiry for `CreateProposal` action.
3. The VWR expires at the current slot — it's only valid within the same transaction.

**Common mistake:** Creating a VWR in one transaction and trying to create a proposal in another. The VWR will be expired. Both `createBet` and `createProposal` must be in the same transaction.

---

## Discriminators

| Instruction | Discriminator (bytes) |
|------------|----------------------|
| create_bet | `[197, 42, 153, 2, 59, 63, 143, 246]` |
| create_voter_weight_record | `[184, 249, 133, 178, 88, 152, 250, 186]` |
| register_vote | `[115, 72, 69, 208, 153, 57, 252, 35]` |
| settle_bet | `[115, 55, 234, 177, 227, 4, 10, 67]` |
| resolve_bet | See IDL |
| withdraw_payoff | `[46, 190, 15, 253, 65, 113, 31, 144]` |
| cast_sowellian_vote | See IDL |
| update_voter_weight_record | See IDL |

---

## Error Reference

| Code | Hex | Name | Cause |
|------|-----|------|-------|
| 3012 | 0xBC4 | AccountNotInitialized | Vote receipt doesn't exist — `registerVote` was never called |
| 6013 | 0x177D | ProposalNotLive | Trying to `registerVote` on cancelled/completed proposal |
| 0x44D | — | VoterWeightRecord expired | VWR was created in a previous transaction/slot |
| 0x1 | — | InsufficientFunds | Not enough governing tokens for the bet |
| 0x1783 | — | Invalid voter weight action | Wrong action type passed to `updateVoterWeightRecord` |

---

## Checklist: Creating a Sowellian Proposal

- [ ] Have sufficient governing tokens for the bet stake
- [ ] Have SOL for transaction fees (~0.02 SOL)
- [ ] TokenOwnerRecord exists for your wallet
- [ ] VoterWeightRecord exists (create with `createVoterWeightRecord` if not)
- [ ] **TX1:** `createBet` + `createProposal` + `addSignatory` — all in ONE transaction
- [ ] **TX2:** `insertTransaction` (governance instructions) + `signOffProposal`
- [ ] **TX3:** `registerVote` — **CRITICAL, do not skip**
- [ ] Verify bet.is_registered === true after TX3
- [ ] Governance instructions are properly serialized (resolved from versioned tx if using Jupiter)
