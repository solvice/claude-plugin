# Solvice Claude.ai Project — Setup Guide

This guide walks you through creating a Claude.ai Project that acts as a Solvice VRP integration expert with live access to the Solvice API via MCP.

---

## Prerequisites

- A Claude.ai account with Projects access (Pro or Team plan)
- A Solvice API key — get one at https://app.solvice.io

---

## Step 1: Create a New Project

1. Go to https://claude.ai and click **Projects** in the left sidebar.
2. Click **New project**.
3. Name it something like `Solvice VRP Assistant`.

---

## Step 2: Set the System Prompt

1. Inside the project, click **Edit project details** (or the settings icon).
2. Open `system-prompt.md` from this directory and copy its entire contents.
3. Paste into the **Custom instructions** field.
4. Save.

---

## Step 3: Upload Knowledge Files

Knowledge files give Claude persistent context about Solvice constraints and patterns. Upload all three:

1. In the project, click **Add content** → **Upload files**.
2. Upload `constraint-error-glossary.md` (from `claude-plugin/docs/`).
3. Upload `integration-patterns.md` (from `claude-plugin/docs/`).
4. Upload the VRP request schema:
   - Download it from https://api.solvice.io/v2/static/vrp-request-schema.json
   - Upload the downloaded `vrp-request-schema.json` file.

---

## Step 4: Connect the MCP Server

1. In the project settings, find the **Integrations** or **MCP** section.
2. Click **Add MCP server**.
3. Enter the following:
   - **URL**: `https://api.solvice.io/mcp`
   - **Authentication**: API key
   - **Token**: your Solvice API key (JWT)
4. Save and confirm the connection is active.

---

## Step 5: Verify the Setup

Start a new conversation inside the project and send:

> Generate a demo VRP problem.

Claude should call the `vrp-demo` MCP tool automatically and return a sample `OnRouteRequest` JSON payload. If it returns a request without calling any tool, check that the MCP server connection is active in project settings.

---

## Usage Examples

Use these conversation starters to get the most out of the assistant:

**Debug a solution**

> Debug this solution: `a1b2c3d4-e5f6-7890-abcd-ef1234567890`

Claude will call `vrp-get-explanation` on the ID, interpret the score, and suggest exact field fixes for any violations.

**Build a VRP request for a use case**

> Help me build a VRP request for same-day grocery deliveries with 3 vehicles, time windows, and capacity constraints.

Claude will ask clarifying questions, generate a minimal `OnRouteRequest`, call `vrp-solve-sync` to verify feasibility, and return a corrected payload.

**Compare two solve runs**

> Compare `a1b2c3d4-e5f6-7890-abcd-ef1234567890` vs `b2c3d4e5-f6a7-8901-bcde-f12345678901`

Claude will fetch both solutions and explanations in parallel and summarise the score differences and constraint changes side by side.
