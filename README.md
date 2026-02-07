# 🔐 claude-vault

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-Plugin-blueviolet)](https://docs.anthropic.com/en/docs/claude-code)
[![Platform](https://img.shields.io/badge/platform-linux%20%7C%20macOS%20%7C%20WSL-lightgrey)]()

Secure secrets handling for Claude Code. Keeps API keys out of session logs.

## The Problem

Claude Code logs everything - commands, outputs, file reads. When you run:

```bash
API_KEY=$(cat ~/.secrets/openai.env | grep KEY | cut -d= -f2)
curl -H "Authorization: Bearer $API_KEY" https://api.openai.com/v1/models
```

Both the key extraction AND the curl command (with the expanded key) get saved to `~/.claude/projects/*/sessions.jsonl` in plaintext. Forever.

## The Solution

This plugin teaches Claude to use **self-destructing scripts** that keep secrets internal:

```bash
cat > /tmp/vault-$$.sh << 'EOF'
#!/bin/bash
source ~/.secrets/openai.env
curl -s https://api.openai.com/v1/models \
    -H "Authorization: Bearer $OPENAI_API_KEY"
rm -f "$0"  # self-destruct
EOF
chmod 700 /tmp/vault-$$.sh && /tmp/vault-$$.sh
```

**Session log shows:** The script creation and API response.
**Session log does NOT show:** The actual API key.

## Quick Start

```bash
# Add marketplace and install
/plugin marketplace add ellyseum/claude-plugins
/plugin install claude-vault

# Set up a secret
mkdir -p ~/.secrets && chmod 700 ~/.secrets
echo "OPENAI_API_KEY=sk-your-key-here" > ~/.secrets/openai.env
chmod 600 ~/.secrets/openai.env
```

On session start, the plugin injects a **CRITICAL** rule into Claude's context that instructs it to always use self-destructing scripts for credential operations.

## How It Works

1. You ask Claude to do something that needs an API key
2. Claude writes a self-destructing script that:
   - Reads secrets from `~/.secrets/` (any format: `.env`, `.json`, `.ini`)
   - Makes the API call
   - Returns only the response
   - Deletes itself
3. **Secrets never appear in session logs**

Claude figures out how to parse your secrets files - you don't need to use a specific format.

## Utility Commands

```bash
/secrets list              # Show what's in ~/.secrets/
/secrets test <service>    # Check if service has secrets

/secrets scan              # Scan session logs for leaked secrets
/secrets scan -v           # Verbose - show matched patterns
/secrets redact            # Redact leaked secrets from logs
/secrets redact --dry-run  # Preview what would be redacted

/secrets audit             # Full security audit
/secrets fix               # Auto-fix permission issues
```

## Secrets Storage

Secrets live in `~/.secrets/` - any format works:

```bash
~/.secrets/
├── openai.env      # OPENAI_API_KEY=sk-...
├── anthropic.env   # ANTHROPIC_API_KEY=sk-ant-...
├── jira.json       # {"apiToken": "...", "email": "...", "domain": "..."}
├── github.env      # GH_TOKEN=ghp_...
└── aws.env         # AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=...
```

Lock it down:
```bash
chmod 700 ~/.secrets
chmod 600 ~/.secrets/*
```

## Examples

### OpenAI
```bash
cat > /tmp/vault-$$.sh << 'EOF'
#!/bin/bash
source ~/.secrets/openai.env 2>/dev/null
curl -s https://api.openai.com/v1/chat/completions \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{"model":"gpt-4","messages":[{"role":"user","content":"hello"}]}'
rm -f "$0"
EOF
chmod 700 /tmp/vault-$$.sh && /tmp/vault-$$.sh
```

### Jira (JSON config)
```bash
cat > /tmp/vault-$$.sh << 'EOF'
#!/bin/bash
DOMAIN=$(jq -r '.domain' ~/.secrets/jira.json 2>/dev/null)
EMAIL=$(jq -r '.email' ~/.secrets/jira.json 2>/dev/null)
TOKEN=$(jq -r '.apiToken' ~/.secrets/jira.json 2>/dev/null)
AUTH=$(printf "%s:%s" "$EMAIL" "$TOKEN" | base64 -w0)

curl -s "https://${DOMAIN}/rest/api/3/myself" \
    -H "Authorization: Basic $AUTH"
rm -f "$0"
EOF
chmod 700 /tmp/vault-$$.sh && /tmp/vault-$$.sh
```

### GitHub CLI
```bash
cat > /tmp/vault-$$.sh << 'EOF'
#!/bin/bash
source ~/.secrets/github.env 2>/dev/null
gh api /user
rm -f "$0"
EOF
chmod 700 /tmp/vault-$$.sh && /tmp/vault-$$.sh
```

### npm publish (config file pattern)
```bash
cat > /tmp/vault-$$.sh << 'EOF'
#!/bin/bash
source ~/.secrets/npm.env 2>/dev/null
echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > /tmp/.npmrc-vault
unset NPM_TOKEN
npm publish --access public --userconfig /tmp/.npmrc-vault
rm -f /tmp/.npmrc-vault
rm -f "$0"
EOF
chmod 700 /tmp/vault-$$.sh && /tmp/vault-$$.sh
```

## Remediation: Scan & Redact

Already have leaked secrets in your session logs? Fix them:

```bash
/secrets scan              # Find leaks
/secrets redact            # Remove them (no backup - that would preserve the leak!)
```

### Detected Patterns

| Provider | Pattern |
|----------|---------|
| OpenAI | `sk-...`, `sk-proj-...` |
| Anthropic | `sk-ant-...` |
| GitHub | `ghp_...`, `gho_...`, `ghu_...`, `ghs_...`, `ghr_...` |
| npm | `npm_...` |
| AWS | `AKIA...` (access keys) |
| Stripe | `sk_live_...`, `sk_test_...`, `pk_live_...`, `pk_test_...` |
| Slack | `xoxb-...`, `xoxp-...`, `xoxa-...` |
| Discord | Bot tokens |
| Generic | `Authorization: Bearer ...`, `api_key=...`, `password=...` |

**Missing a pattern?** [Open an issue](https://github.com/ellyseum/claude-vault/issues)

## Security

- **Temp scripts are mode 700** - only you can read/execute
- **Scripts self-destruct** - deleted after execution
- **No echo/set -x** - secrets never printed
- **Secrets dir is 700** - only you can access
- **Env files are 600** - only you can read
- **Audit command** - verifies permissions

### Stderr Safety

Bash error messages can leak expanded secrets. For example, if a `source` line has a syntax issue, bash prints the offending line with variables already expanded. All secret-reading lines should suppress stderr:

```bash
source ~/.secrets/foo.env 2>/dev/null       # safe
API_KEY=$(jq -r '.key' ~/.secrets/foo.json 2>/dev/null)  # safe
```

For tools that need tokens in config file format (`.npmrc`, `.netrc`, `.pypirc`), write a temp config file instead of trying to use env var names with special characters:

```bash
# WRONG - bash can't parse // in variable names, leaks token in error message
NPM_CONFIG_//registry.npmjs.org/:_authToken="$TOKEN"

# RIGHT - write to temp file, token never hits bash syntax parser
echo "//registry.npmjs.org/:_authToken=${TOKEN}" > /tmp/.npmrc-vault
npm publish --userconfig /tmp/.npmrc-vault
rm -f /tmp/.npmrc-vault
```

## Project Structure

```
claude-vault/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── hooks/
│   └── hooks.json          # Injects CRITICAL rule on session start
├── skills/
│   └── secrets/
│       └── SKILL.md        # Teaches the pattern
├── bin/
│   └── vault-run           # Utility commands
└── README.md
```

## License

MIT
