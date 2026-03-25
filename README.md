# Realms Agent Skills

Agent skills for interacting with the [Realms](https://v2.realms.today) platform on Solana. Compatible with [OpenClaw](https://github.com/openclaw/openclaw) and any agent framework that supports the AgentSkills spec.

## Skills

| Skill | Description |
|-------|-------------|
| [dao-creator](dao-creator/) | Spin up a new DAO on Solana in one conversation |
| [proposal-manager](proposal-manager/) | Create, vote on, and execute governance proposals |
| [multisig](multisig/) | M-of-N multisig wallets — team treasuries, shared wallets, protocol admin |
| [governance](governance/) | Full DAO management — members, delegation, treasury, governances |
| [launchpad](launchpad/) | Token fundraising (ICOs) that auto-convert to DAOs |
| [sowellian](sowellian/) | Prediction market governance — proposals are bets, winners get paid |

## Install

```bash
# Copy any skill into your agent's skills directory
cp -r dao-creator/ ~/.openclaw/workspace/skills/realms-dao-creator/
cp -r proposal-manager/ ~/.openclaw/workspace/skills/realms-proposal-manager/
cp -r multisig/ ~/.openclaw/workspace/skills/realms-multisig/
cp -r governance/ ~/.openclaw/workspace/skills/realms-governance/
cp -r launchpad/ ~/.openclaw/workspace/skills/realms-launchpad/
cp -r sowellian/ ~/.openclaw/workspace/skills/realms-sowellian/
```

Or point your agent to the raw SKILL.md files on GitHub.

## Programs

| Program | Address |
|---------|---------|
| SPL Governance | `GovER5Lthms3bLBqWub97yVrMmEogzX7xNjdXpPPCVZw` |
| Launchpad | `ReaLM68X8dXLz35oXqofDYkNWiBFZr4FcefSJyTr9Yh` |
| Sowellian | `sowEL1Rtn3p479rg34gW7mVPeCNY58Es5rkLpFsCJAW` |

## API

```
https://v2.realms.today/api/v1
```

## Links

- [Realms App](https://v2.realms.today)
- [Realms Discord](https://discord.gg/realms)
