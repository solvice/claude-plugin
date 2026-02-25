---
name: setup
description: Connect Cowork to the Solvice API. Run once to set your API key, then verify the MCP connection is live.
---

# Solvice Setup

One-time setup to connect Cowork to the Solvice routing API.

---

## Step 1: Get Your API Key

1. Go to [https://app.solvice.io](https://app.solvice.io)
2. Navigate to **Settings** → **API Keys**
3. Create a new key or copy an existing one

---

## Step 2: Set the Environment Variable

The plugin's `.mcp.json` reads `SOLVICE_API_KEY` from your environment. Add it to your shell profile so it persists across restarts.

**macOS / Linux:**
```bash
echo 'export SOLVICE_API_KEY=your-key-here' >> ~/.zshrc
source ~/.zshrc
```

Or for bash:
```bash
echo 'export SOLVICE_API_KEY=your-key-here' >> ~/.bashrc
source ~/.bashrc
```

**Windows:**
Open System Properties → Advanced → Environment Variables → New:
- Variable name: `SOLVICE_API_KEY`
- Variable value: `your-key-here`

---

## Step 3: Restart Cowork

Cowork reads environment variables at startup. After setting `SOLVICE_API_KEY`, restart the Claude desktop app fully — quit and reopen, not just close the window.

---

## Step 4: Verify the Connection

Ask Claude: **"Call the vrp-demo tool"**

**Success** — returns a sample `OnRouteRequest` JSON with vehicles, jobs, and settings. The Solvice MCP is live.

**`401 Unauthorized`** — the API key is missing or invalid. Double-check the value and confirm the environment variable is set in the current shell (`echo $SOLVICE_API_KEY`).

**Tool not found** — the MCP server did not start. Confirm you restarted Cowork after setting the env var.

---

## Step 5: Next Steps

- `/solvice:scaffold` — build your first routing request through a short interview
- `/solvice:debug <solve-id>` — diagnose an infeasible or partially-served solution

---

## What You Get in Cowork

Once connected, these tools render interactive apps inline:

- **`vrp-get-solution`** — opens a live route map showing each vehicle's path and stops
- **`vrp-get-explanation`** — opens a score dashboard with constraint violation breakdown
- **`vrp-what-if`** — opens a drag-and-drop editor to move jobs between routes and see the score update instantly
