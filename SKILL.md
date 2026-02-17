# Double Claude — Dual Subscription Manager

Manage two Claude Pro subscriptions with instant switching between accounts.

**Use this skill when:**
- You have two Claude Pro accounts and want to switch between them
- Primary account runs out mid-billing cycle
- You want to use the second account as overflow
- You need to maximize Claude usage across two subscriptions

**Triggers:** `/double`, "switch claude", "double claude"

---

## What This Skill Does

1. **Two accounts** — switch between two Claude Pro subscriptions
2. **Instant switching** — `/double` command switches immediately
3. **Manual control** — helper script for switching
4. **No auto-failover** — you control when to switch

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

# Full bigrestart (gateway restart does NOT reload profiles!)
echo "Bigrestart gateway..."
systemctl --user restart openclaw-gateway
sleep 3
if systemctl --user is-active openclaw-gateway > /dev/null 2>&1; then
  echo "✓ Done! Active profile: $PROFILE_ID"
else
  echo "❌ Gateway failed to start! Check: journalctl --user -u openclaw-gateway -n 50"
  exit 1
fi
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

### ⚠️ CRITICAL: Switching Requires Bigrestart

**`openclaw gateway restart` does NOT apply profile changes to active sessions.**

After changing `lastGood.anthropic`, you MUST run:
```bash
systemctl --user restart openclaw-gateway
```

This is a full process restart that forces profile reload. Without it, the old profile stays active.

### /double Command

When user sends `/double`:

1. Read current profile: `jq -r '.lastGood.anthropic' ~/.openclaw/agents/main/agent/auth-profiles.json`
2. Toggle to the other one (claude-cli ↔ fallback)
3. Write new lastGood
4. **Run bigrestart:** `systemctl --user restart openclaw-gateway`
5. Wait for reconnect, verify with `session_status`

```
/double         # switch to the other account
/double primary # switch to first account
/double fallback# switch to second account
```

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

**⚠️ CRITICAL:** `openclaw gateway restart` does NOT reload auth profiles for active sessions!

You MUST do a full bigrestart:

```bash
systemctl --user restart openclaw-gateway
```

This is the ONLY reliable way to force profile reload.

### How to check which subscription is currently active?

```bash
# Method 1: Check auth-profiles.json
jq -r '.lastGood.anthropic' ~/.openclaw/agents/main/agent/auth-profiles.json

# Method 2: Check OpenClaw status
openclaw models status | grep anthropic
```

### "auth-profiles.json — not found" / Профили не найдены

**Важно:** Профили лежат в правильном месте — `~/.openclaw/agents/main/agent/auth-profiles.json`

**Не** в:
- `~/.env`
- `~/.openclaw/openclaw.json`

Проверка профилей:
```bash
# Список профилей
jq '.profiles | keys[] | select(startswith("anthropic:"))' ~/.openclaw/agents/main/agent/auth-profiles.json

# Какой сейчас активен
jq '.lastGood.anthropic' ~/.openclaw/agents/main/agent/auth-profiles.json
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

## Kimi Fallback Chain

When both Claude subscriptions hit rate limits, OpenClaw falls back to alternative models. Kimi (free API keys from kimi.com) is ideal for this — you can create multiple accounts and rotate keys.

### Why Kimi?

- **Free** — no cost per token (kimi.com/coding gives free API keys)
- **200K context** — matches Claude's context window
- **128K output** — generous output limit
- **Multiple keys** — create N accounts for N fallback slots
- **OpenAI-compatible API** — works with OpenClaw's openai-completions adapter

### Setup: Ask the user

When implementing, ask:

1. **How many Kimi keys do you want in the fallback chain?** (recommended: 3-4)
2. **Provide your API keys** (one per line, format: `sk-kimi-...`)
3. **Where should Kimi sit in the chain?** (before or after minimax/other fallbacks)

### Implementation

For each key N (1-indexed), add to `openclaw.json`:

**1. Provider definition** (`models.providers`):

```json
"kimi{N}": {
  "baseUrl": "https://api.kimi.com/coding/v1",
  "apiKey": "sk-kimi-YOUR_KEY_HERE",
  "api": "openai-completions",
  "models": [{
    "id": "kimi-for-coding",
    "name": "Kimi 2.5 Coding (key{N})",
    "reasoning": false,
    "input": ["text"],
    "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
    "contextWindow": 200000,
    "maxTokens": 128000
  }]
}
```

**Note:** First key uses provider name `kimi-coding` (legacy), keys 2+ use `kimi2`, `kimi3`, `kimi4`, etc.

**2. Auth profile** (`auth.profiles`):

```json
"kimi{N}:default": {
  "provider": "kimi{N}",
  "mode": "api_key"
}
```

**3. Model alias** (`agents.defaults.models`):

```json
"kimi{N}/kimi-for-coding": {
  "alias": "kimi{N}"
}
```

**4. Fallback chain** (`agents.defaults.model.fallbacks`):

```json
"fallbacks": [
  "kimi4/kimi-for-coding",
  "kimi-coding/kimi-for-coding",
  "kimi2/kimi-for-coding",
  "kimi3/kimi-for-coding",
  "minimax/MiniMax-M2.5"
]
```

### Full Fallback Chain Example (current setup)

```
Primary:    anthropic/claude-sonnet-4-5  (Claude, oauth profile: claude-cli)
                    ↓ rate limit
Fallback 1: anthropic/claude-sonnet-4-5  (Claude, oauth profile: fallback)
                    ↓ rate limit
Fallback 2: kimi4/kimi-for-coding        (Kimi key4, free)
                    ↓ rate limit
Fallback 3: kimi-coding/kimi-for-coding   (Kimi key1, free)
                    ↓ rate limit
Fallback 4: kimi2/kimi-for-coding         (Kimi key2, free)
                    ↓ rate limit
Fallback 5: kimi3/kimi-for-coding         (Kimi key3, free)
                    ↓ rate limit
Fallback 6: minimax/MiniMax-M2.5          (MiniMax, free)
```

### Quick Add Script

To add a new Kimi key to an existing chain:

```bash
#!/bin/bash
# Usage: ./add-kimi-key.sh <key_number> <api_key>
# Example: ./add-kimi-key.sh 5 sk-kimi-abc123...

N=$1
KEY=$2
CONFIG="$HOME/.openclaw/openclaw.json"

if [ -z "$N" ] || [ -z "$KEY" ]; then
  echo "Usage: $0 <key_number> <api_key>"
  exit 1
fi

PROVIDER="kimi${N}"

# Add provider
jq ".models.providers.\"$PROVIDER\" = {
  \"baseUrl\": \"https://api.kimi.com/coding/v1\",
  \"apiKey\": \"$KEY\",
  \"api\": \"openai-completions\",
  \"models\": [{
    \"id\": \"kimi-for-coding\",
    \"name\": \"Kimi 2.5 Coding (key${N})\",
    \"reasoning\": false,
    \"input\": [\"text\"],
    \"cost\": {\"input\": 0, \"output\": 0, \"cacheRead\": 0, \"cacheWrite\": 0},
    \"contextWindow\": 200000,
    \"maxTokens\": 128000
  }]
}" "$CONFIG" > /tmp/oc-new.json && mv /tmp/oc-new.json "$CONFIG"

# Add auth profile
jq ".auth.profiles.\"${PROVIDER}:default\" = {
  \"provider\": \"$PROVIDER\",
  \"mode\": \"api_key\"
}" "$CONFIG" > /tmp/oc-new.json && mv /tmp/oc-new.json "$CONFIG"

# Add model alias
jq ".agents.defaults.models.\"${PROVIDER}/kimi-for-coding\" = {
  \"alias\": \"kimi${N}\"
}" "$CONFIG" > /tmp/oc-new.json && mv /tmp/oc-new.json "$CONFIG"

# Append to fallbacks (before minimax)
jq ".agents.defaults.model.fallbacks |= (. - [\"minimax/MiniMax-M2.5\"]) + [\"${PROVIDER}/kimi-for-coding\", \"minimax/MiniMax-M2.5\"]" "$CONFIG" > /tmp/oc-new.json && mv /tmp/oc-new.json "$CONFIG"

echo "✓ Added $PROVIDER with alias kimi${N}"
echo "Restart gateway: systemctl --user restart openclaw-gateway"
```

### Getting Kimi API Keys

1. Go to https://kimi.com
2. Create account (use different email for each key)
3. Navigate to API settings / Coding section
4. Copy the API key (`sk-kimi-...`)
5. Repeat with different accounts for more keys

---

## Related

- [OpenClaw Authentication Docs](https://docs.openclaw.ai/gateway/authentication)
- [Claude OAuth Setup](https://docs.openclaw.ai/providers/anthropic)
- [Cron Scheduler](https://docs.openclaw.ai/automation/cron)

---

**Skill created:** 2026-02-16  
**Version:** 1.2  
**Tested with:** OpenClaw 2026.2.15, Claude Code CLI v2.1.42  
**Changelog:**
- v1.2 — Added Kimi fallback chain setup, quick-add script, full chain documentation
- v1.1 — Fixed: gateway restart → bigrestart (systemctl restart) for reliable profile switching

### "auth-profiles.json — not found" / Профили не найдены

**Важно:** Профили лежат в правильном месте — `~/.openclaw/agents/main/agent/auth-profiles.json`

**Не в:**
- `~/.env`
- `~/.openclaw/openclaw.json`

**Проверка профилей:**

```bash
# Список профилей
jq '.profiles | keys[] | select(startswith("anthropic:"))' ~/.openclaw/agents/main/agent/auth-profiles.json

# Какой сейчас активен
jq '.lastGood.anthropic' ~/.openclaw/agents/main/agent/auth-profiles.json
```
