---
name: waymakeros-commander
description: >-
  Work in Waymaker Commander through the MCP server or the `waymaker` CLI: the order to do things
  in, the prerequisites that dead-end you, and the calls that report success without doing what you
  meant. Use before creating tasks, documents, sheets, goals, roles or calendar events, before
  reading or triaging someone's mail or a shared inbox, and before touching MyVault notes or files.
  Trigger phrases: "add a task in Commander", "create a Commander document", "write to my vault",
  "save this to MyVault", "my vault note is missing", "triage my inbox", "check the shared inbox",
  "set up a workspace", "book a meeting in Waymaker", "update the org chart".
---

# Working in Waymaker Commander

The tool descriptions say what each call does. This page says what they don't: which organisation
you're in, where things live, what each write touches, and which calls can't be undone.

**Read the top five sections before your first write.** They cover the mistakes an agent makes
most often. The tool-by-tool notes after them are reference.

## 1. Know which organisation you're in

Every call runs as **one signed-in person, in one organisation**. Nothing in a call chooses the
organisation; the connection does.

- **An API key belongs to one organisation.** A person in two organisations has a key, and a
  separate MCP config, for each. With OAuth, the connection is bound to the organisation that was
  active when the person connected.
- **Call `get_my_info` first.** It returns the user and organisation IDs for this connection. If
  the person names a workspace and you can't find it, check you're in the right
  organisation before you create anything. A missing workspace usually means a different
  organisation, not a missing workspace.
- **To work in another organisation, reconnect with that organisation's key.** An
  `organization_id` argument can't reach another organisation. Anything other than the
  connection's own organisation is refused.
- **The CLI remembers a current workspace and project.** `waymaker workspaces use` and
  `waymaker projects use` save a default that later commands reuse. A `.commander/config.json`
  in the current directory, written by `waymaker init`, overrides it. Run `waymaker auth whoami`
  and `waymaker workspaces current` before a write from a directory you didn't set up.

## 2. Workspaces, projects, and where things go

```
Organisation
├── Roles (org chart), Teams, Goals, People   ← organisation-wide
└── Workspaces                                 ← a department, client or business area
    └── Projects
        └── Taskboards (→ status columns, layers → tasks), Documents, Sheets, Folders, Sparks
```

- **Find before you create.** `commander_workspace_list` → `commander_project_list(workspace_id)`
  → `commander_taskboard_list(project_id)`. `commander_search` finds tasks, documents and layers by
  name, and is usually faster than walking the tree.
- **A task needs a board, and a board needs a column.** `commander_status_list(taskboard_id)`
  gives the columns. Without `status_section_id` a task lands in the first column.
- **Every person has a personal workspace called "My Workspace".** Work for one person goes there.
  Work for a team goes in a shared workspace.
- **A new workspace is private by default.** `commander_workspace_create` defaults to
  `access_type: explicit`, which means only its creator can see it. For a workspace the whole
  organisation should see, pass `access_type: organization`. If you create it wrong, change it
  with `commander_workspace_update`. Team-only access needs a team chosen, so set that up in
  Commander.
- **Desktop calls a local folder a repository, never a workspace.** In Commander, "workspace"
  always means the shared, cloud container above. See `waymakeros-commander-desktop`.

## 3. MyVault is a git repository

Each person's MyVault is a **private git repository of markdown notes**. The web app, the MCP
tools, an IDE clone, Commander Desktop and email capture all commit to that same repository. Every
save is a commit.

**"The note is missing" means pull. Never re-create it.** If a note shows up through the MCP but
not in your local clone, your clone is behind. Run `git pull`. If you re-author the note from an
MCP read, you fork the history into duplicate, divergent commits, and someone has to untangle them
by hand. **Pull at the start of every session** that works in a clone.

- **Read the vault's own rules first.** `AGENTS.md` at the vault root is the contract: the
  frontmatter spec, folder layout, wiki-link syntax and commit-message prefixes. The person may
  have edited it. Subfolders can add their own `AGENTS.md`.
- **Updates replace the whole note.** `commander_notes_update` has no patch mode, so send the full
  new body. Pass the `base_ref` from your last `commander_notes_read`, so the update fails
  instead of overwriting an edit made since you read it. Keep `id` and `created`, and set
  `updated`.
- **`commander_notes_create` fails if the path exists.** That failure protects the existing note,
  so don't work around it by choosing a new path. Read the note, then update it.
- **Bulk imports go through `commander_notes_create_many`**, up to 200 notes per call. One call
  per note is slow and strains the vault.
- **Ask before you touch many notes.** "I'm about to edit 47 notes in `clients/`. Confirm?"

### Large files are not stored in the vault's git history

A note can be at most **1 MB**. A non-text file over **100 KB**, or any file over **1 MB**, is
stored by the platform outside git. The vault keeps a small `<name>.workspace-ref` pointer in its
place.

- **To add a file, use the file tools, not git.** For files up to 10 MB (base64) or 25 MB (from a
  URL), use `commander_vault_attach`. It can also write the note that references the file, in the
  same call. For any size, use `commander_vault_upload_init`, PUT the bytes to the signed URL, then
  call `commander_vault_upload_complete`. From a terminal, use `waymaker vault add <file>`.
- **To read a file, use `commander_vault_attach_get(path)`.** It returns a short-lived download
  link. A pointer file opened as text is not the file.
- **In a clone, never `git add -f` a large binary.** If one is committed, the platform moves it out
  of git and replaces it with a pointer. In a fresh clone it then appears as a `.workspace-ref`.
- **`commander_vault_compact` rewrites the vault's history and force-pushes it.** Run it only when
  the person explicitly asks. After it runs, every existing clone must be re-cloned, not pulled.

## 4. Mail tools act inside a real person's mailbox

`commander_email_*` work inside **the connected person's own mailbox**: the one they read in their
mail app and on their phone. Nothing you do there is a sandbox.

- **Opening a message marks it read.** `commander_email_get` marks the message read for the person,
  everywhere they read mail. To triage without that side effect, work from `commander_email_list`
  and `commander_email_search`. Neither marks anything read. If you opened something the person hasn't
  seen, use `commander_email_mark_read` with `read: false` to mark it unread again.
- **`commander_email_move` moves the message in the person's real mailbox.** Folders are the
  person's own: list them with `commander_email_folder_list`, and never invent a destination.
- **The MCP can't send, reply or delete mail.** If the person asks you to, say so. Don't look for
  another route.
- **Check before you create a task from an email.** `commander_email_linked_tasks(uid)` shows the
  tasks already made from that message.
- **If a thread call errors, read the messages one at a time** with `commander_email_get`, or with
  `commander_shared_inbox_email_get` for a shared inbox.

**A shared inbox is a team's mailbox**, such as support@ or sales@, with its own tools
(`commander_shared_inbox_*`). Assigning, moving to a folder and adding an internal note are all
visible to the whole team. Notes are never sent to the customer. Nothing in the shared-inbox tools
sends mail.

## 5. What can be undone

Prefer a reversible call. Before any permanent one, confirm with the person and say what will be
lost, including anything that goes with it.

**Reversible, with the call that undoes it:**

| Call | Undo with |
|---|---|
| `commander_notes_delete` (moves the note to `archive/`) | `commander_notes_restore` |
| `commander_notes_update` | `commander_notes_history`, then `commander_notes_restore_version` (last 50 versions) |
| `commander_document_update` with `status: archived` | `commander_document_update` with `status: private` or `published` |
| `commander_email_move` / `commander_shared_inbox_move` | Move it back. A shared-inbox move takes the whole thread. |
| `commander_role_assign` with `action: unassign` | `commander_role_assign` |
| `commander_workspace_link_remove` | `commander_workspace_link_add` |

**Permanent, or impossible to undo with the tools:**

| Call | What goes with it |
|---|---|
| `commander_task_delete` | Its subtasks, comments, checklist, attachments and time entries |
| `commander_sheet_delete` | All its data and version history |
| `commander_table_delete` / `commander_table_row_delete` | Every row / that row |
| `commander_layer_delete` | Its child layers. Tasks stay on the board, unlayered. |
| `commander_event_delete` | For a recurring event, the **whole series**. There's no single-occurrence delete. |
| `commander_folder_delete` | Can't be restored. Its contents aren't deleted with it, so move or delete them first. |
| `commander_spark_delete`, `commander_chat_delete`, `commander_connection_delete`, `commander_journey_delete`, `commander_rush_delete`, `commander_contact_delete` | Gone from the person's view, with no restore call |
| `commander_app_revoke_all` | Every subscription to your app. All subscribers lose access at once. |
| `commander_notes_hard_delete`, `commander_vault_delete`, `commander_vault_compact` | The note or file, or (for compact) the vault's old history |

**There's no MCP delete for** documents (archive them instead), goals, projects, workspaces,
roles, people or taskboards. If the person wants one removed, they do it in Commander. The CLI has
delete commands for several of these: treat every CLI `delete` as permanent.

**Calls that reach other people:** `commander_event_create` and `commander_event_update` with
attendees involve real people. Don't assume a deleted event's attendees were told, so remind the
person to let them know. `commander_message_send` posts as the person. `commander_task_comment`
and `commander_shared_inbox_note_add` are visible to everyone who can see the item. Say what
you're about to send before you send it.

## Tool notes

**Tasks.** `commander_task_update` with `is_completed: true` is what completes a task. Moving it
to a "Done" column doesn't, and `commander_task_list_mine` keeps showing it until `is_completed` is
set. `progress` and `is_completed` are independent, so set both when you want them to agree.
`create_kanban` makes a board with To Do / In Progress / Done columns. Pass it a `project_id`.

**Documents.** These are rich-text documents. Notes belong in MyVault.
- **`text_content` is not what people see.** On `commander_document_create`, `text_content`
  becomes one plain paragraph, so markdown headings and lists show as literal characters. On
  `commander_document_update`, `text_content` changes only the search text, and the document's
  visible body stays the same. To change what people read, send `content` as structured
  (Tiptap JSON) content.
- **Who can see a document follows where it's filed.** A document in a project or workspace is
  visible to the people who can open that project or workspace. File it in the right place.
  Don't rely on `status` to share it.

**Sheets.** Rows and columns are **zero-indexed** in `commander_sheet_set_cell`. A1 ranges are
used everywhere else. `set_range` and `apply_formula` take up to 1,000 cells per call, and
`set_range` is atomic: one bad cell aborts the whole write. Declare typed columns *after* creating
the sheet, with `commander_sheet_set_column_type`. To write a header into a typed column, pass
`coerce: false`. `commander_sheet_validate_formulas` checks syntax only.

**Goals.** Create the objective (`commander_goal_create`), then add measurable key results
(`commander_key_result_create`). Through the MCP a goal can be active or completed. To cancel or
archive a goal, the person does it in Commander. Check the status you set by reading the goal back.

**People and roles.** Roles use `name`, `purpose` and `reports_to_id`, not `title`,
`description` or `reports_to`. `commander_role_assign` both assigns and unassigns: pass
`action: unassign` to remove someone. `commander_user_invite` only *stages* a person. The
invitation goes out when an admin activates them, either from the admin app or with
`waymaker users activate`. To find a person, use `commander_user_list`. To find who knows a
subject, use `waymaker_people_find`.

**Calendar.** Call `commander_event_list` for the range before proposing a time. **Always give
times with a UTC offset** (`2026-10-06T09:00:00+10:00`). A time with no offset is not read as the
person's local time. `commander_calendar_list` gives the `calendar_id` values, including any
connected external calendars.

**Files.** `commander_file_*` cover the person's own files (My Files). These are separate from
Host storage buckets. A large file added to MyVault is stored in My Files too, but add it with the
vault tools (§3) so the vault keeps its link to it. "I can't upload anything" is answered by
`commander_file_storage_usage`.

## Permissions you might not have

The tool list is filtered by the connection's permissions. A read-only key shows no write tools at
all, not even disabled ones. If a tool you expect is missing, the key lacks the permission. A
change to the key's permissions takes effect only after the client reconnects. **Pass only the
arguments a tool's schema lists.** Others aren't rejected, but they aren't supported either: they
may be ignored, or do something you didn't intend. If a parameter seems to do nothing, check the
schema.

More: [MCP tool catalogue](https://docs.waymakerone.com/mcp/tools) ·
[Connecting a client](https://docs.waymakerone.com/mcp/connecting) ·
[MyVault](https://help.waymakerone.com/my-vault) ·
[Workspaces & projects](https://help.waymakerone.com/commander/workspaces)
