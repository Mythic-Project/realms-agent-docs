---
name: realms-sowellian
description: Create and manage Sowellian betting proposals on Solana DAOs through Realms. Sowellian is a prediction market governance plugin — proposals are bets where users stake tokens to vote Yes/No, winners get paid from losers. Use for creating betting proposals with governance instructions (e.g. Jupiter trades), settling bets, withdrawing winnings, and attaching trade instructions to proposals. Triggers on "Sowellian", "betting proposal", "prediction market", "bet on proposal", "stake to vote", "trade proposal".
---

# Sowellian Betting

Prediction market plugin for Realms governance. Proposals are bets — stake tokens to vote Yes/No, winners get paid from losers.

## Configuration

```yaml
program_id: sowEL1Rtn3p479rg34gW7mVPeCNY58Es5rkLpFsCJAW
governance_program: GovER5Lthms3bLBqWub97yVrMmEogzX7xNjdXpPPCVZw
cluster: mainnet-beta
```

## Critical Rules

1. **NEVER skip `registerVote` (TX3)** — without it, bet creator's tokens are **permanently locked** with no recovery
2. **3-transaction flow is mandatory** — do not combine or skip steps
3. **`createBet` locks tokens immediately** — ensure sufficient balance
4. **VoterWeightRecord expires per-slot** — `createBet` and `createProposal` MUST be in the same transaction
5. **Use the same `betSeed` everywhere** — derives bet PDA, proposal PDA, and vote receipt PDA

## ⚠️ Known Bug

If a proposal is cancelled BEFORE `registerVote` is called, bet tokens are **permanently locked**. No `cancel_bet` or `refund_bet` exists. Always execute all 3 transactions atomically. If TX1 succeeds but TX2/TX3 fail, retry immediately.

## Bet Lifecycle

```
createBet → registerVote → [Voting Period] → settleBet → resolveBet → withdrawPayoff
```

## 3-Transaction Flow

### TX1: Create Bet + Proposal + Signatory
- `createBet` — locks tokens, creates bet PDA, sets VoterWeightRecord
- `createProposal` — creates governance proposal using VWR (same tx!)
- `addSignatory` — adds creator as signatory

### TX2: Insert Instructions + Sign Off
- `insertTransaction` — attach governance instructions (e.g. Jupiter limit order)
- `signOffProposal` — starts voting period

### TX3: Register Vote (CRITICAL)
- `registerVote` — creates vote receipt, auto-casts Yes vote for creator
- **Without this, tokens are permanently locked**

## Post-Voting

- **settleBet** — permissionless, anyone can call after voting ends
- **resolveBet** — realm authority only, sets outcome (Yes/No)
- **withdrawPayoff** — winners claim tokens

## References

- **Full implementation** (all instructions, account derivations, discriminators, Jupiter integration): See [references/implementation.md](references/implementation.md)
