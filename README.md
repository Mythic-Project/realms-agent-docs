# Realms Agent Skills

Agent skills for interacting with the [Realms](https://v2.realms.today) platform on Solana. Compatible with [OpenClaw](https://github.com/openclaw/openclaw) and any agent framework that supports the AgentSkills spec.

## Skills

| Skill | Description |
|-------|-------------|
| [governance](governance/) | DAO management, proposals, voting, treasury, delegation via REST API |
| [launchpad](launchpad/) | Token fundraising (ICOs) that auto-convert to DAOs |
| [sowellian](sowellian/) | Prediction market governance — proposals are bets, winners get paid |

## Install

```bash
# Copy a skill into your agent's skills directory
cp -r governance/ ~/.openclaw/workspace/skills/realms-governance/
cp -r launchpad/ ~/.openclaw/workspace/skills/realms-launchpad/
cp -r sowellian/ ~/.openclaw/workspace/skills/realms-sowellian/
```

Or point your agent to the raw skill files:

```
https://raw.githubusercontent.com/Mythic-Project/realms-agent-docs/main/governance/SKILL.md
https://raw.githubusercontent.com/Mythic-Project/realms-agent-docs/main/launchpad/SKILL.md
https://raw.githubusercontent.com/Mythic-Project/realms-agent-docs/main/sowellian/SKILL.md
```

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
