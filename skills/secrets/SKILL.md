---
name: secrets
description: Securely execute commands requiring API keys without exposing secrets in session logs
---

## The Rule

**NEVER** let secrets appear in session logs. This means:
- No `cat ~/.secrets/*`
- No `echo $API_KEY`
- No `curl -H "Authorization: Bearer sk-actual-key"`
- No reading secret files with the Read tool

**ALWAYS** use self-destructing scripts that keep secrets internal.

---

## Where Secrets Live

```
~/.secrets/
├── openai.env      # OPENAI_API_KEY=sk-...
├── jira.json       # {"apiToken": "...", "email": "...", "domain": "..."}
├── cloudflare.ini  # [cloudflare]\napi_token=...
├── github.env      # GH_TOKEN=ghp_...
└── aws.env         # AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=...
```

Files can be any format (`.env`, `.json`, `.ini`, `.yaml`). You figure out how to parse them.

---

## The Pattern

When you need to use secrets, write a **self-destructing script**:

```bash
cat > /tmp/vault-XXXXXX.sh << 'EOF'
#!/bin/bash
set -e

# 1. Read secrets from file (parse whatever format)
API_KEY=$(jq -r '.apiToken' ~/.secrets/jira.json)
EMAIL=$(jq -r '.email' ~/.secrets/jira.json)

# 2. Use them in your command
curl -s -X GET \
    -H "Authorization: Basic $(echo -n "$EMAIL:$API_KEY" | base64)" \
    "https://mycompany.atlassian.net/rest/api/3/search?jql=..."

# 3. Self-destruct
rm -f "$0"
EOF
chmod 700 /tmp/vault-XXXXXX.sh
/tmp/vault-XXXXXX.sh
```

**What gets logged:** The script creation and output.
**What stays secret:** The actual API key values (only exist inside the running script).

---

## Helper Commands

These are optional - you can always write scripts directly.

```bash
# List what's in ~/.secrets/
/secrets list

# Check if a service has secrets configured
/secrets test <service>

# Scan session logs for leaked secrets
/secrets scan
/secrets scan -v              # Verbose

# Redact leaked secrets from session logs
/secrets redact
/secrets redact --dry-run     # Preview

# Security audit (permissions, leaks, etc.)
/secrets audit

# Auto-fix permission issues
/secrets fix
```

---

## Examples

### OpenAI API call
```bash
cat > /tmp/vault-$$.sh << 'EOF'
#!/bin/bash
source ~/.secrets/openai.env
curl -s https://api.openai.com/v1/chat/completions \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{"model":"gpt-4","messages":[{"role":"user","content":"hello"}]}'
rm -f "$0"
EOF
chmod 700 /tmp/vault-$$.sh && /tmp/vault-$$.sh
```

### Jira with JSON config
```bash
cat > /tmp/vault-$$.sh << 'EOF'
#!/bin/bash
DOMAIN=$(jq -r '.domain' ~/.secrets/jira.json)
EMAIL=$(jq -r '.email' ~/.secrets/jira.json)
TOKEN=$(jq -r '.apiToken' ~/.secrets/jira.json)
AUTH=$(printf "%s:%s" "$EMAIL" "$TOKEN" | base64 -w0)

curl -s "https://${DOMAIN}/rest/api/3/myself" \
    -H "Authorization: Basic $AUTH" \
    -H "Content-Type: application/json"
rm -f "$0"
EOF
chmod 700 /tmp/vault-$$.sh && /tmp/vault-$$.sh
```

### AWS CLI
```bash
cat > /tmp/vault-$$.sh << 'EOF'
#!/bin/bash
source ~/.secrets/aws.env
aws s3 ls
rm -f "$0"
EOF
chmod 700 /tmp/vault-$$.sh && /tmp/vault-$$.sh
```

### GitHub CLI
```bash
cat > /tmp/vault-$$.sh << 'EOF'
#!/bin/bash
source ~/.secrets/github.env
gh api /user
rm -f "$0"
EOF
chmod 700 /tmp/vault-$$.sh && /tmp/vault-$$.sh
```

---

## Key Points

1. **Secrets stay inside the script** - never echoed, never returned to Claude
2. **Script self-destructs** - `rm -f "$0"` at the end
3. **Only output is the API response** - that's what Claude sees
4. **Any file format works** - parse with jq, source, grep, whatever fits
5. **chmod 700** - script is only readable/executable by user

---

## Remediation

Already have leaked secrets in session logs?

```bash
/secrets scan          # Find them
/secrets redact        # Remove them (no backup - that would preserve the leak!)
```

Detected patterns: OpenAI (`sk-`), Anthropic (`sk-ant-`), GitHub (`ghp_`), AWS (`AKIA`), Stripe, Slack, Discord, and generic Bearer/password patterns.
