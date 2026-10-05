# WaymakerOS skills

Playbooks for coding agents building on WaymakerOS: the order things go in, the decisions that are
permanent, and the failures that report success. One package, one skill per product.

```bash
npx skills add waymakerone/waymakeros                          # every WaymakerOS skill
npx skills add waymakerone/waymakeros --list                   # see what is in the package
npx skills add waymakerone/waymakeros --skill waymakeros-host  # just one
```

| Skill | Use it |
|---|---|
| [`waymakeros`](skills/waymakeros/SKILL.md) | first, whenever a Waymaker MCP server or the `waymaker` CLI is in use. It says which playbook below fits the job |
| [`waymakeros-commander`](skills/waymakeros-commander/SKILL.md) | before creating or changing work in Commander: tasks, documents, sheets, goals, MyVault, mail |
| [`waymakeros-commander-desktop`](skills/waymakeros-commander-desktop/SKILL.md) | when working in a terminal inside the Commander Desktop Mac app |
| [`waymakeros-host`](skills/waymakeros-host/SKILL.md) | before creating an app, provisioning a database, writing migrations, or planning and applying a Solution release on Waymaker Host; and before adding end-user sign-in, app email, storage, Ambassadors or a custom domain (reference pages in `skills/waymakeros-host/references/`) |

**Why these exist:** the WaymakerOS MCP and CLI describe *what* each command does. They do not say
which order to run them in, which flags cannot be changed later, or which failures look like
success. Every entry in a skill is a real failure someone hit.

## Layout

```
README.md                    this file
skills/<name>/SKILL.md       one directory per skill; the directory name is the skill's `name`
skills/<name>/...            anything else that skill needs
```

There is deliberately **no `SKILL.md` at the root**. The `skills` installer stops at a root
`SKILL.md` and installs only that one, so a root skill would hide every product skill below it.
With skills under `skills/<name>/`, one `npx skills add waymakerone/waymakeros` finds all of them,
and `--skill <name>` picks one.

## Where this comes from

This package is published automatically from Waymaker's source repository every time its
`skills/` directory changes. It is a mirror: changes made directly here are overwritten by the next
publish. To report a problem with a skill, open an issue.

<!-- Maintainers: the publisher and its rules are documented in scripts/publish-skills.mjs in the source repository. -->
