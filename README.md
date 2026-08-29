# WaymakerOS skills

The build playbook for [WaymakerOS](https://waymakeros.com), as an agent skill.

```bash
npx skills add waymakerone/waymakeros
```

This is what the Waymaker MCP server asks connected agents to load before building
on Host. `SKILL.md` covers the Host build-and-deploy path: the order the steps go
in, the flags that are permanent, and the failures that return success.

**Why it exists.** The MCP tool descriptions tell an agent *what* each command does.
They do not tell it the order to press them in, which choices cannot be undone, or
which failures report success anyway. Every entry here is a real failure someone
hit — most of them during a first client build, by an agent following the
documented path.

Commander and One playbooks follow.
