# Double Claude — Dual Subscription Manager for OpenClaw

Seamlessly manage **two different Claude Pro/Team subscriptions** with instant switching.

## Why?

- **Zero downtime** when your primary subscription hits limits
- **Automatic restoration** when your billing cycle resets
- **Proactive planning** with weekly usage reminders
- **Simple switching** via one-line commands

## Quick Start

```bash
# 1. Install Claude CLI
sudo npm install -g @anthropic-ai/claude-code

# 2. Generate tokens for both subscriptions
claude setup-token  # Primary
claude setup-token  # Fallback (log in with different account first)

# 3. Add fallback profile to OpenClaw
# (see SKILL.md for details)

# 4. Create switching script
# (copy from SKILL.md)

# 5. Switch subscriptions
~/claude-switch.sh primary
~/claude-switch.sh fallback

# OR use /double command in Telegram:
/double         # show current account
/double primary # switch to first account
/double fallback# switch to second account
```

## Commands

| Command | Description |
|---------|-------------|
| `/double` | Show current account |
| `/double primary` | Switch to first account (account1) |
| `/double fallback` | Switch to second account (account2) |

Also works with the shell script:
```bash
~/claude-switch.sh primary
~/claude-switch.sh fallback
```

## Features

✅ **Dual OAuth profiles** — two active subscriptions  
✅ **Instant switching** — one command to failover  
✅ **Scheduled restoration** — cron job for automatic switch-back  
✅ **Usage reminders** — weekly checks to plan ahead  
✅ **OpenClaw native** — no external dependencies  

## Use Cases

- **Mid-cycle exhaustion** — primary runs out before reset
- **Heavy workload weeks** — use fallback for burst work
- **Team + Personal** — switch between work and personal accounts
- **Multi-project** — isolate different projects to different subscriptions

## Requirements

- OpenClaw 2026.2.12+
- Claude Pro or Team subscription (x2)
- Node.js 18+ (for Claude CLI)

## Documentation

See [SKILL.md](./SKILL.md) for complete setup guide and troubleshooting.

## License

MIT

## Author

Created by [Chip](https://t.me/chipda) for the OpenClaw community.

## Contributing

Issues and PRs welcome! This is a community skill.

---

**OpenClaw Skill** • [Install Skills](https://docs.openclaw.ai/skills) • [More Skills](https://clawhub.com)
