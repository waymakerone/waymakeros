---
name: waymakeros
description: >-
  Entry point for working in WaymakerOS through its MCP server or the `waymaker` CLI. Says which
  playbook to load for the job: Commander (tasks, documents, sheets, goals, people, calendar, mail,
  MyVault), Commander Desktop (agents in the Mac app's terminals), or Host (apps, databases,
  releases). Load it whenever a Waymaker MCP server is connected, or the person mentions
  WaymakerOS, Waymaker, Commander, MyVault or Host. Trigger phrases: "Waymaker", "WaymakerOS",
  "in Commander", "my vault", "deploy on Waymaker", "waymaker CLI".
---

# WaymakerOS

WaymakerOS has three layers:

- **Commander**: the business tools a team works in every day. Tasks, documents, sheets, goals,
  the org chart, calendar, mail and each person's MyVault.
- **Host**: where you build and run the organisation's own apps, databases and serverless
  functions.
- **One**: the intelligence layer that reads across both.

The MCP tool descriptions tell you **what** each call does. The playbooks below tell you what the
tools leave out: the order of operations, the prerequisites that dead-end you, and the calls that
report success without doing what you meant. **Load the one for your job before you start.**

## Which playbook

| You're about to… | Load |
|---|---|
| Create or change tasks, boards, documents, sheets, goals, roles, events or workspaces | `waymakeros-commander` |
| Read, write or move MyVault notes or files, or work in a vault clone | `waymakeros-commander` (§3) |
| Read or triage someone's mail or a shared inbox | `waymakeros-commander` (§4) |
| Delete or archive anything in Commander | `waymakeros-commander` (§5) |
| Work in a terminal inside the Commander Desktop Mac app (`TERM_PROGRAM=CommanderDesktop`) | `waymakeros-commander-desktop`, then `waymakeros-commander` |
| Create a Host app, provision a database, run migrations, plan or apply a Solution release | `waymakeros-host` |

A job that spans two layers needs both playbooks. For example, building an app in a repository
and tracking it on a Commander taskboard means loading both `waymakeros-host` and
`waymakeros-commander`.

## Three things that apply everywhere

1. **You act as one person, in one organisation.** Every call runs with that person's
   permissions, and anything you create or change is theirs. The connection decides the
   organisation. Call `get_my_info` before your first write, and say "in your organisation" when
   you describe what you can do.
2. **Find before you create.** Search or list first (`commander_search`, the `*_list` tools,
   `waymaker <group> list`). A duplicate board, document or note costs the person more to clean up
   than the lookup would have cost you.
3. **What's missing from your tool list isn't available to you.** The tool list is filtered by
   the connection's permissions. If a tool isn't there, say so. Don't go looking for another
   route.

## Getting connected

- MCP server: `https://mcp.waymakerone.com/mcp`. See
  [Connecting an MCP client](https://docs.waymakerone.com/mcp/connecting).
- CLI: `npm i -g @waymakeros/cli`, then set `WAYMAKER_API_KEY` and run `waymaker auth status`.
- Every tool, by category: [MCP tool catalogue](https://docs.waymakerone.com/mcp/tools).
