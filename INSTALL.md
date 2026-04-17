# Installation Guide — Brevo Sales Engineer Plugin

Step-by-step setup to use the CRM and Loyalty demo agents in Claude Code.

---

## Prerequisites

- macOS, Linux, or Windows (WSL recommended on Windows)
- A terminal (Terminal on Mac, bash/zsh on Linux, WSL on Windows)
- Node.js 18+ installed → check with: `node --version`
  - If missing: https://nodejs.org/en/download

---

## Step 1 — Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

Verify the installation:

```bash
claude --version
```

---

## Step 2 — Authenticate with your Anthropic account

```bash
claude
```

On first launch, Claude Code opens a browser to log in with your Anthropic account. Follow the instructions, then come back to the terminal.

---

## Step 3 — Configure the Brevo plugin

### 3a — Open the settings file

```bash
# Create the directory if it doesn't exist
mkdir -p ~/.claude

# Open the settings file (creates it if missing)
code ~/.claude/settings.json
# or: nano ~/.claude/settings.json
# or: open ~/.claude/settings.json  (Mac, opens in default editor)
```

### 3b — Add the plugin configuration

Copy-paste the following content. If the file already exists with content, **merge** the keys below with your existing content (do not replace the whole file).

```json
{
  "extraKnownMarketplaces": {
    "brevo": {
      "source": {
        "source": "git",
        "url": "https://github.com/antoineleonbrevo/sales_engineer_repo.git"
      }
    }
  },
  "enabledPlugins": {
    "sales-engineer@brevo": true
  }
}
```

Save the file.

---

## Step 4 — Verify the plugin is loaded

Open a new terminal window and run:

```bash
claude
```

Type the following in Claude Code:

```
/plugins
```

You should see **sales-engineer** listed as an active plugin. If not, restart Claude Code and try again.

---

## Step 5 — You're ready

### Launch a CRM demo

```
I want to set up a demo for [Prospect Name]
```

Claude Code will:
1. Research the prospect automatically
2. Propose a dataset (contacts, products, orders, events, custom objects)
3. Ask for your Brevo API key
4. Execute everything after your validation

### Launch a Loyalty demo

```
loyalty demo for [Prospect Name]
```

Claude Code will:
1. Propose a loyalty program (tiers, points, rewards)
2. Ask for your Brevo API key
3. Execute everything after your validation

### Combined CRM + Loyalty demo

```
I want to do a custom demo for [Prospect Name] (CRM + Loyalty)
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `claude: command not found` | Run `npm install -g @anthropic-ai/claude-code` again, then restart your terminal |
| Plugin not showing in `/plugins` | Check that `settings.json` is saved at `~/.claude/settings.json` and restart Claude Code |
| `Permission denied` on settings.json | Run `chmod 644 ~/.claude/settings.json` |
| Plugin outdated | Claude Code pulls the latest version from GitHub automatically at each session start |

---

## Getting updates

The plugin updates automatically from GitHub every time you start Claude Code. No action needed.

If you want to force a refresh:

```bash
rm -rf ~/.claude/plugins/marketplaces/brevo
# Then restart Claude Code — it will re-download the plugin
```

---

## Support

Contact: antoine.leon@brevo.com
