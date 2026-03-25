# Proposal Instruction Serialization & Insertion

How instructions get from `TransactionInstruction` objects into governance proposals.

## The Flow

```
TransactionInstruction
  → serializeInstructionToBase64()
  → getInstructionDataFromBase64()
  → InstructionData { programId, accounts[], data }
  → withInsertTransaction()
  → Stored on-chain in ProposalTransaction account
  → Executed when proposal passes
```

## Serialization

```typescript
import {
  serializeInstructionToBase64,
  getInstructionDataFromBase64,
  InstructionData,
  withInsertTransaction,
} from '@realms-today/spl-governance';

// 1. Build a TransactionInstruction (any Solana ix)
const ix: TransactionInstruction = createTransferCheckedInstruction(...);

// 2. Serialize to base64
const base64 = serializeInstructionToBase64(ix);

// 3. Parse to InstructionData (governance format)
const instructionData: InstructionData = getInstructionDataFromBase64(base64);
```

## Inserting into a Proposal

```typescript
const insertIxs: TransactionInstruction[] = [];

// Each instruction becomes a separate proposal transaction
for (const [index, instructionData] of governanceInstructions.entries()) {
  await withInsertTransaction(
    insertIxs,
    governanceProgramId,
    programVersion,
    governance,          // governance account pubkey
    proposalAddress,     // proposal pubkey
    tokenOwnerRecord,    // proposer's TOR
    walletPublicKey,     // proposer (governanceAuthority)
    index,               // transaction index (0-based)
    0,                   // option index (0 for single-choice proposals)
    holdUpTime,          // seconds before execution (0 = immediate after vote)
    [instructionData],   // array of InstructionData (usually 1 per transaction)
    walletPublicKey,     // payer
  );
}
```

## Chunking

Multiple insert instructions may exceed transaction size limits. Chunk them:

```typescript
import { chunks } from '@/lib/data';

// Default chunk size based on instruction type
const chunkSize = hasComplexInstructions ? 1 : 2;  // 1 for DeFi, 2-3 for simple transfers
const insertChunks = chunks(insertIxs, chunkSize);

// Send each chunk as a separate Solana transaction
for (const chunk of insertChunks) {
  const tx = new Transaction().add(...chunk);
  await sendAndConfirmTransaction(connection, tx, [wallet]);
}
```

## Prerequisite Instructions

Some instructions need setup that runs at **execution time** (not proposal creation):

```typescript
// Prerequisite instructions are NOT part of the governance proposal.
// They run as top-level Solana instructions in the same tx as withExecuteTransaction.

// Example: Kamino deposit needs refreshReserve + refreshObligation
// in the same slot as the actual deposit instruction.

// During execution:
const prereqIxs = await getExecutionPrerequisites(innerIxs, wallet, connection);
const execIxs = [];
await withExecuteTransaction(execIxs, ...);

// Both in same Solana tx:
const tx = new Transaction().add(
  ComputeBudgetProgram.setComputeUnitLimit({ units: 1_000_000 }),
  ...prereqIxs,  // refresh reserve, refresh obligation, etc.
  ...execIxs,    // the governance execute instruction
);
```

## Hold-Up Time

`holdUpTime` is the delay (in seconds) between a proposal passing and when the instruction can be executed.

- `0` = executable immediately after vote passes
- `300` = 5 minute delay (used for Manifest settle instructions)
- Governance default: `governance.account.config.minInstructionHoldUpTime`

## Multi-Transaction Execution

When a proposal has multiple instructions, each becomes a separate `ProposalTransaction` on-chain. At execution time:

```typescript
// multiTransactionMode: each ProposalTransaction = separate Solana tx
// This is needed when instructions are too large for one tx

for (const proposalInstruction of proposalInstructions) {
  const execIxs = [];
  await withExecuteTransaction(execIxs, ...proposalInstruction);

  // Add prereqs for this specific instruction
  const prereqIxs = await getExecutionPrerequisites(...);

  const tx = new Transaction().add(
    ComputeBudgetProgram.setComputeUnitLimit({ units: 1_000_000 }),
    ...prereqIxs,
    ...execIxs,
  );
  await sendAndConfirmTransaction(connection, tx, [wallet]);
}
```

## Complete Example: Transfer Proposal

```typescript
import {
  withCreateProposal,
  withInsertTransaction,
  withSignOffProposal,
  withAddSignatory,
  getSignatoryRecordAddress,
  serializeInstructionToBase64,
  getInstructionDataFromBase64,
  VoteType,
} from '@realms-today/spl-governance';

// 1. Build transfer instruction
const transferIx = SystemProgram.transfer({
  fromPubkey: treasuryAddress,
  toPubkey: recipientAddress,
  lamports: 1_000_000_000, // 1 SOL
});

// 2. Create proposal
const proposalIxs = [];
// Fetch governance to get proposalCount (used as proposal index)
const governanceAccount = await getGovernance(connection, governance);

const proposalAddress = await withCreateProposal(
  proposalIxs, programId, version, realm, governance,
  tokenOwnerRecord, "Transfer 1 SOL", "Payment for services",
  governingTokenMint, wallet, governanceAccount.account.proposalCount,
  VoteType.SINGLE_CHOICE, ['Approve'], true, wallet
);

await withAddSignatory(
  proposalIxs, programId, version, proposalAddress,
  tokenOwnerRecord, wallet, wallet, wallet
);

// Send TX1: create proposal + add signatory
await sendAndConfirm(proposalIxs);

// 3. Insert instruction
const insertIxs = [];
const instructionData = getInstructionDataFromBase64(
  serializeInstructionToBase64(transferIx)
);

await withInsertTransaction(
  insertIxs, programId, version, governance, proposalAddress,
  tokenOwnerRecord, wallet, 0, 0, 0, [instructionData], wallet
);

// 4. Sign off (starts voting)
const signatoryRecord = await getSignatoryRecordAddress(programId, proposalAddress, wallet);
withSignOffProposal(
  insertIxs, programId, version, realm, governance,
  proposalAddress, wallet, signatoryRecord, undefined
);

// Send TX2: insert instruction + sign off
await sendAndConfirm(insertIxs);
```
