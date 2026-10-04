# WaymakerOS Host skill

The build-and-deploy playbook for Waymaker Host, as an agent skill.

Install:

```bash
npx skills add waymakerone/waymakeros --skill waymakeros-host
```

(`npx skills add waymakerone/waymakeros` installs every WaymakerOS skill, this one included.)

**Why this exists:** the MCP tool descriptions tell an agent *what* each command does. They do not
tell it the order to run them in, which flags are permanent, or which failures return success.
Every entry in `SKILL.md` is a real failure someone hit — many of them during a first client build,
by an agent following the documented path.
