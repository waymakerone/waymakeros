# WaymakerOS skill (entry point)

`status: draft`

The entry-point skill the Waymaker MCP server tells connected agents to load. It is deliberately
short: it routes to the playbook for the job.

| Skill | For |
|---|---|
| `waymakeros-commander` | Commander through MCP / CLI — tasks, documents, sheets, goals, people, calendar, mail, MyVault |
| `waymakeros-commander-desktop` | Agents running in Commander Desktop's terminals on a Mac |
| `waymakeros-host` | Building and releasing on Host |

This directory is the canonical source. It is published to the public `waymakeros` skills
package alongside the other `waymakeros-*` skills. The publishing layout (one package with several
skills, or one package per skill) is being settled separately; this file assumes the sibling
layout that `waymakeros-host` already uses.
