# WaymakerOS Commander skill

`status: draft`

The working playbook for Waymaker Commander over MCP and the `waymaker` CLI, as an agent skill.

**Why this exists:** the MCP tool descriptions say *what* each call does. They don't say which
organisation a call lands in, that a new workspace is private by default, that a vault note "missing"
locally means pull (never re-create), that opening an email marks it read in a real person's
mailbox, or which deletes can't be undone. Every entry in `SKILL.md` was checked against the
current server and tool code; most come from real failures.

This directory is the canonical source. It is published to the public `waymakeros` skills package.
Load it via the `waymakeros` entry-point skill.
