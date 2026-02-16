# Double Claude — Dual Subscription Manager

Manage two Claude subscriptions with automatic failover and scheduled switching.

**Use this skill when:**
- Your primary Claude subscription runs out mid-billing cycle
- You want automatic fallback to a secondary subscription
- You need scheduled switching back to primary when it resets
- You want to maximize Claude usage across multiple Pro subscriptions

**Triggers:** "double claude", "claude failover", "switch claude subscription", "dual claude setup"

---

## What This Skill Does

1. **Dual OAuth profiles** — maintains two separate Claude subscription tokens
2. **Automatic failover** — switches to backup when primary hits limits
3. **Scheduled restoration** — cron job to switch back when primary resets
4. **Manual control** — helper script for instant switching
5. **Weekly reminders** — check usage and plan ahead

---

## Prerequisites

- OpenClaw installed and configured
- Two separate Claude Pro/Team subscriptions
- Access to both Claude accounts (different emails)

---

## Setup Process

### Step 1: Install Claude CLI

```bash
sudo npm install -g @anthropic-ai/claude-code
```

### Step 2: Generate OAuth Token for Primary Subscription

```bash
# This will open browser for OAuth flow
claude setup-token
```

Save the token — this is your **primary** profile (`anthropic:claude-cli` or `anthropic:primary`).

### Step 3: Generate OAuth Token for Fallback Subscription

**Important:** Log out of Claude in your browser first, then log in with your SECOND subscription account.

```bash
# Run again for fallback subscription
claude setup-token
```

You'll get a URL like:
```
https://claude.ai/oauth/authorize?code=true&client_id=...
```

Open it in browser (where you're logged into the SECOND account), authorize, copy the code.

Paste the code when prompted. Save this token — this is your **fallback** profile.

### Step 4: Add Both Profiles to OpenClaw

Location: `~/.openclaw/agents/main/agent/auth-profiles.json`

Add fallback profile:

```bash
jq '.profiles."anthropic:fallback" = {
  "type": "oauth",
  "provider": "anthropic",
  "access": "YOUR_FALLBACK_ACCESS_TOKEN",
  "refresh": "YOUR_FALLBACK_REFRESH_TOKEN",
  "expires": EXPIRY_TIMESTAMP_MS
}' ~/.openclaw/agents/main/agent/auth-profiles.json > /tmp/auth-new.json && \
mv /tmp/auth-new.json ~/.openclaw/agents/main/agent/auth-profiles.json
```

**Note:** Setup-token from Claude CLI contains both access and refresh tokens in the single OAuth token string. OpenClaw will extract them automatically.

### Step 5: Create Switching Script

```bash
cat > ~/claude-switch.sh << 'EOF'
#!/bin/bash
# Claude subscription switcher
# Usage: ./claude-switch.sh {primary|fallback}

set -e

TARGET=$1
AUTH_FILE="$HOME/.openclaw/agents/main/agent/auth-profiles.json"

if [ -z "$TARGET" ]; then
  echo "Usage: $0 {primary|fallback}"
  echo ""
  echo "Current active profile:"
  jq -r '.lastGood.anthropic' "$AUTH_FILE"
  exit 1
fi

case "$TARGET" in
  primary)
    PROFILE_ID="anthropic:claude-cli"
    NOTE="Switched to primary Claude subscription"
    ;;
  fallback)
    PROFILE_ID="anthropic:fallback"
    NOTE="Switched to fallback Claude subscription"
    ;;
  *)
    echo "Error: Target must be 'primary' or 'fallback'"
    exit 1
    ;;
esac

echo "Switching to: $PROFILE_ID"

# Update lastGood in auth-profiles.json
jq ".lastGood.anthropic = \"$PROFILE_ID\"" "$AUTH_FILE" > /tmp/auth-profiles-new.json
mv /tmp/auth-profiles-new.json "$AUTH_FILE"

echo "✓ Updated auth-profiles.json"

# Restart gateway
echo "Restarting gateway..."
openclaw gateway restart --note "$NOTE" --delay 2000

echo "✓ Done! Active profile: $PROFILE_ID"
EOF

chmod +x ~/claude-switch.sh
```

### Step 6: Set Up Automatic Switching

**Switch to fallback NOW:**

```bash
~/claude-switch.sh fallback
```

**Schedule switch back to primary** (example: next Thursday 1:00 PM local time):

```bash
# Calculate UTC time for your local reset time
# Example: Thursday 1:00 PM MSK = Thursday 10:00 AM UTC

openclaw cron add --json '{
  "name": "claude-switch-to-primary",
  "schedule": {"kind": "at", "at": "2026-02-20T10:00:00.000Z"},
  "sessionTarget": "main",
  "wakeMode": "now",
  "payload": {
    "kind": "systemEvent",
    "text": "Switch back to primary Claude subscription. Run: ~/claude-switch.sh primary"
  },
  "enabled": true
}'
```

**Optional: Weekly usage reminder:**

```bash
openclaw cron add --json '{
  "name": "claude-usage-check",
  "schedule": {"kind": "cron", "expr": "0 10 * * 4", "tz": "UTC"},
  "sessionTarget": "isolated",
  "wakeMode": "now",
  "payload": {
    "kind": "agentTurn",
    "message": "Weekly Claude usage check. Remind user to check https://claude.ai/settings/usage and switch to fallback if primary is running low."
  },
  "delivery": {"mode": "announce"},
  "enabled": true
}'
```

---

## Usage

### Check Current Profile

```bash
jq -r '.lastGood.anthropic' ~/.openclaw/agents/main/agent/auth-profiles.json
```

### Switch Manually

```bash
# Switch to primary
~/claude-switch.sh primary

# Switch to fallback
~/claude-switch.sh fallback
```

### Check OAuth Status

```bash
openclaw models status --json | jq '.auth.oauth.providers[] | select(.provider == "anthropic")'
```

### List All Profiles

```bash
jq '.profiles | keys[] | select(startswith("anthropic:"))' ~/.openclaw/agents/main/agent/auth-profiles.json
```

---

## How It Works

### OAuth Profile Selection

OpenClaw uses `lastGood.anthropic` in `auth-profiles.json` to determine which profile to use:

```json
{
  "profiles": {
    "anthropic:claude-cli": { "type": "oauth", "access": "...", "refresh": "...", "expires": ... },
    "anthropic:fallback": { "type": "oauth", "access": "...", "refresh": "...", "expires": ... }
  },
  "lastGood": {
    "anthropic": "anthropic:fallback"  // ← Active profile
  }
}
```

### Token Refresh

OpenClaw automatically refreshes OAuth tokens before expiry. Both profiles maintain their own refresh tokens.

### Failover Logic

1. Primary subscription runs out → switch to fallback manually or via cron
2. OpenClaw continues using fallback profile
3. Scheduled cron job switches back when primary resets
4. Weekly reminder helps you plan ahead

---

## Troubleshooting

### "No credentials found for profile anthropic:fallback"

Re-run `claude setup-token` while logged into your second Claude account and update the profile.

### "OAuth token refresh failed"

Token expired or was revoked. Generate new setup-token:

```bash
claude setup-token
# Update auth-profiles.json with new tokens
```

### Switch didn't take effect

Restart gateway manually:

```bash
openclaw gateway restart
```

### How to check which subscription is currently active?

```bash
# Method 1: Check auth-profiles.json
jq -r '.lastGood.anthropic' ~/.openclaw/agents/main/agent/auth-profiles.json

# Method 2: Check OpenClaw status
openclaw models status | grep anthropic
```

---

## Tips

**Naming convention:** Use descriptive profile names:
- `anthropic:personal` / `anthropic:work`
- `anthropic:primary` / `anthropic:backup`
- `anthropic:main` / `anthropic:overflow`

**Reset timing:** Claude Pro resets at a specific time each billing cycle. Find yours at https://claude.ai/settings/billing and schedule accordingly.

**Multiple fallbacks:** You can add more than 2 profiles. Just follow the same pattern:

```json
"anthropic:backup2": { "type": "oauth", ... }
```

**Monitoring:** Set up usage tracking to auto-switch before hitting limits:

```bash
# Check current usage via Claude API
# Switch proactively at 80% usage
```

---

## Advanced: Auto-Failover on Rate Limit

Add this to your OpenClaw config to automatically switch when hitting rate limits:

```json5
{
  "hooks": {
    "internal": {
      "entries": {
        "claude-failover": {
          "enabled": true,
          "on": "rate_limit_error",
          "action": "switch_profile",
          "provider": "anthropic",
          "fallback": "anthropic:fallback"
        }
      }
    }
  }
}
```

*(Note: This is a conceptual example — implement via custom hook script)*

---

## Security Notes

- **Never commit auth-profiles.json** to version control
- **Use environment variables** for sensitive tokens if scripting
- **Rotate tokens regularly** (Claude OAuth tokens are valid for 1 year)
- **Separate accounts** — use different email addresses for primary/fallback

---

## Files Created

- `~/claude-switch.sh` — Manual switching script
- `~/.openclaw/agents/main/agent/auth-profiles.json` — OAuth profiles storage
- Cron jobs in OpenClaw scheduler

---

## Examples

### Scenario 1: Mid-Month Exhaustion

```bash
# Primary runs out on Feb 10th
~/claude-switch.sh fallback

# Schedule switch back for March 1st (reset day)
openclaw cron add --json '{
  "name": "restore-primary",
  "schedule": {"kind": "at", "at": "2026-03-01T00:00:00.000Z"},
  "sessionTarget": "main",
  "payload": {"kind": "systemEvent", "text": "~/claude-switch.sh primary"}
}'
```

### Scenario 2: Heavy Week Ahead

```bash
# Proactively switch to fallback for heavy work
~/claude-switch.sh fallback

# Keep primary for regular use next week
# (manual switch back when needed)
```

### Scenario 3: Team + Personal

```bash
# Work hours: use team subscription
~/claude-switch.sh primary  # (team account)

# After hours: use personal subscription
~/claude-switch.sh fallback  # (personal account)
```

---

## Related

- [OpenClaw Authentication Docs](https://docs.openclaw.ai/gateway/authentication)
- [Claude OAuth Setup](https://docs.openclaw.ai/providers/anthropic)
- [Cron Scheduler](https://docs.openclaw.ai/automation/cron)

---

**Skill created:** 2026-02-16  
**Version:** 1.0  
**Tested with:** OpenClaw 2026.2.15, Claude Code CLI v2.1.42
