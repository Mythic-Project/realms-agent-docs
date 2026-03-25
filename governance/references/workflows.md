# Governance Workflows & Decision Trees

## Should I Vote on This Proposal?

```
START
  │
  ├─► Is proposal state === 2 (Voting)?
  │     NO → Cannot vote
  │     YES ↓
  │
  ├─► GET /daos/{realmPk}/members/{walletPk}/voting-power
  │     Both null? → Cannot vote (not a member)
  │     Has tokens? ↓
  │
  ├─► Read proposal descriptionLink — understand what it does
  │
  ├─► instructionsCount > 0? → Review what code will execute
  │
  ├─► GET /daos/{realmPk}/proposals/{proposalPk}/votes
  │     Calculate: yesWeight / (yesWeight + noWeight)
  │
  ├─► Check treasury impact if funds involved
  │     GET /daos/{realmPk}/treasury
  │     What % of treasury does this affect?
  │
  └─► Make informed decision
        voteKind: 0 (yes), 1 (no), 2 (abstain), 3 (veto)
```

## Can I Create a Proposal?

```
START
  │
  ├─► GET /daos/{realmPk}/members/{walletPk}/voting-power
  │     Get depositAmount for community/council
  │
  ├─► GET /daos/{realmPk}/governances
  │     Get minCommunityTokensToCreateProposal
  │     Get minCouncilTokensToCreateProposal
  │
  ├─► Is threshold === "18446744073709551615"?
  │     YES → That token type CANNOT create proposals
  │     NO ↓
  │
  ├─► depositAmount >= threshold?
  │     NO → Insufficient tokens
  │     YES → Can create proposal
  │
  └─► Pick governancePk (use treasury you want to act on)
```

## Proposal Lifecycle

```
State 0 (Draft)
  └─► sign_off_proposal → State 2 (Voting)

State 2 (Voting)
  ├─► vote (while active)
  ├─► relinquish_vote (change mind)
  └─► (wait for voting period to end)

State 2 (Voting ended)
  └─► finalize_vote → State 3 (Succeeded) or 7 (Defeated)

State 3 (Succeeded)
  └─► execute_proposal → State 5 (Completed)

State 7 (Defeated)
  └─► finalize_vote (reclaim deposit)
```

## Safety Checks

### Before Voting YES on Treasury Proposals
1. Calculate `proposalAmount / totalTreasury`
2. If > 10% → flag for human review
3. Check if similar proposals failed recently
4. Verify proposer has history of successful proposals

### Before Creating DAOs
1. Verify unique name via `GET /daos`
2. Ensure wallet has >= 2.5 SOL
3. Confirm council members are valid Solana addresses

### Before Executing Proposals
1. Re-check state === 3 (Succeeded)
2. Verify `instructionsCount > 0`
3. Understand what the instructions will do

## Example: Monitor and Execute Passed Proposals

```
1. GET /daos/{realmPk}/proposals → filter state === 3
2. For each: POST /daos/{realmPk}/proposals/{proposalPk}/execute
   { "walletPublicKey": "..." }
3. Sign and submit (permissionless — any wallet can execute)
```
