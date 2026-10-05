---
name: waymakeros-commander
description: >-
  Work in Waymaker Commander through the MCP server or the `waymaker` CLI: the order to do things
  in, the prerequisites that dead-end you, and the calls that report success without doing what you
  meant. Use before creating tasks, documents, sheets, goals, roles or calendar events, before
  reading or triaging someone's mail or a shared inbox, and before touching MyVault notes or files.
  Trigger phrases: "add a task in Commander", "create a Commander document", "update the doc",
  "fill in the sheet", "write to my vault", "save this to MyVault", "my vault note is missing",
  "triage my inbox", "check the shared inbox", "set up a workspace", "book a meeting in Waymaker",
  "update the org chart", "who reports to".
---

# Working in Waymaker Commander

The tool descriptions say what each call does. This page says what they don't: which organisation
you're in, where things live, what each write touches, and which calls can't be undone.

**Read the top five sections before your first write.** They cover the mistakes an agent makes
most often. The tool-by-tool notes after them are reference, but read the note for a tool before
you use it: several tools succeed without doing what their description suggests.

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
- **One exception reads across organisations:** `commander_task_list_mine` returns the person's
  tasks from every organisation they belong to. Check each task's board before acting on it.
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
        └── Taskboards (→ columns, layers → tasks), Documents, Sheets, Folders, Sparks
```

- **Find before you create.** `commander_workspace_list` → `commander_project_list(workspace_id)`
  → `commander_taskboard_list(project_id)`. `commander_search` finds tasks and layers by name.
  **It does not find documents** made with `commander_document_create`: list those with
  `commander_document_list(workspace_id | project_id)`.
- **A task needs a board, and a board needs a column.** `commander_status_list(taskboard_id)`
  returns the columns in display order, each with its `is_completed` flag (that is how you find the
  Done column). **Always pass `status_section_id` when you create a task.** Without it, the column
  the task lands in is not reliably the first one people see.
- **`create_kanban` makes five columns:** Backlog, To Do, In Progress, Review, Done (its
  description says three). Pass it a `project_id`.
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
by hand. **Pull at the start of every session** that works in a clone, and again before you push.

- **Read the vault's own rules first.** `AGENTS.md` at the vault root is the contract: the
  frontmatter spec, folder layout, wiki-link syntax and commit-message prefixes. The person may
  have edited it. Subfolders can add their own `AGENTS.md`.
- **Ask before you touch many notes.** "I'm about to edit 47 notes in `clients/`. Confirm?"

### Writing through the MCP

- **Updates replace the whole note.** `commander_notes_update` has no patch mode, so send the full
  new body. Keep `id` and `created`, and set `updated`.
- **Pass `base_ref` from your last `commander_notes_read`.** The update then fails instead of
  overwriting an edit made since you read. **`base_ref` is checked against the whole vault, not
  the one note:** any commit anywhere in the vault since your read returns `base_ref mismatch`.
  On a busy vault, read the note again, re-apply your change to what you just read, and retry.
  Never retry by dropping `base_ref`.
- **A read can be up to 10 minutes old after someone pushes from a clone.** That is another reason
  to always send `base_ref`: without it, an update built on a stale read overwrites the pushed
  edit, silently.
- **`commander_notes_update` does not create**: a missing path returns 404. **`commander_notes_create`
  fails if the path exists**, which protects the existing note, so don't work around it by choosing
  a new path. Read the note, then update it.
- **Slow down.** Writes are limited to 30 a minute per person, and the same path can't be written
  twice within 10 seconds, so create-then-immediately-update fails. A `RATE_LIMITED` or
  `COMMIT_TOO_FAST` error means wait, not retry in a loop.
- **Bulk imports go through `commander_notes_create_many`**, up to 200 notes per call. A malformed
  item rejects the whole call; a path that already exists fails only that note. Check the result
  per note.
- **A note can be at most 1 MB**, and a full vault refuses writes with `QUOTA_EXCEEDED`.
- **Writes to the root `archive/` folder, and to any path segment starting with `.`, are
  refused.** `commander_notes_delete` moves a note to `archive/`; deleting a second note with the
  same path replaces the first archived copy.
- **`commander_notes_vault_clone_token` and `commander_notes_vault_graph` do not work through the
  MCP.** To clone, the person uses Commander or Commander Desktop.

### Large files are not stored in the vault's git history

A non-text file over **100 KB**, or any file over **1 MB**, is stored by the platform outside git.
The vault keeps a small `<name>.workspace-ref` pointer in its place.

- **To add a file, use the file tools, not git.** For files up to 10 MB (base64) or 25 MB (from a
  URL), use `commander_vault_attach`. It can also write the note that references the file, in the
  same call: **check `note_created` in the result**, because the call reports success even when
  the file was stored and the note was not. For larger files (up to 5 GB), use
  `commander_vault_upload_init`, PUT the bytes to the signed URL, then call
  `commander_vault_upload_complete`. From a terminal, use `waymaker vault add <file>`.
- **Attaching to a path that already has a file replaces it**, without warning.
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
  everywhere they read mail. **To read without that side effect, use `commander_email_thread`**,
  which returns full bodies and marks nothing read, or work from `commander_email_list` and
  `commander_email_search`. If you opened something the person hasn't seen, use
  `commander_email_mark_read` with `read: false` to mark it unread again.
- **`commander_email_move` moves the message in the person's real mailbox, and only to standard
  folders:** `inbox`, `sent`, `drafts`, `trash`, `archive`, `spam`. A move to a custom folder from
  `commander_email_folder_list` is refused. **Moving to `trash` is deleting, as far as the person
  is concerned**: ask first.
- **The MCP can't send or reply.** If the person asks you to, say so. Don't look for another
  route.
- **`commander_email_search`** returns at most 50 results, with no paging. Keep queries to plain
  words: commas and brackets in a query break it.
- **`commander_email_linked_tasks` does not work yet** (it is refused). Before creating a task from
  an email, search the board for the email's subject instead.

**A shared inbox is a team's mailbox**, such as support@ or sales@, with its own tools
(`commander_shared_inbox_*`). Assigning, moving to a folder and adding an internal note are all
visible to the whole team. Notes are never sent to the customer. Nothing in the shared-inbox tools
sends mail.

- **`commander_shared_inbox_email_list` ignores `folder_id` and `limit`.** It returns the 50 newest
  incoming emails across all folders. Filter the result yourself, and say it may not be complete.
- **Assigning covers one email, not the thread**, and can set off the team's automations. Assign
  to a workspace member, by user ID.
- **A shared-inbox move takes the whole thread.**
- **A thread view can include unrelated mail** with the same subject from the last 30 days, when
  the messages carry no thread ID. Check senders and dates before you summarise "the thread".
- Reading a shared-inbox email does not mark it read.

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

**Calls that reach other people:** `commander_message_send` posts as the person.
`commander_task_comment` and `commander_shared_inbox_note_add` are visible to everyone who can see
the item. Say what you're about to send before you send it.

## Tool notes

### Tasks

- **A column is not a status.** Moving a task to the Done column does not complete it, and
  completing it does not move it. `commander_task_update` with `is_completed: true` is what
  completes a task; `commander_task_list_mine` and board counts read that. Some Commander views
  (calendar, timeline) go by the column instead. To finish a task properly, do both: set
  `is_completed: true` **and** move it to the Done column (`status_section_id`).
- `progress` and `is_completed` are independent, so set both when you want them to agree.
- **A task with no assignee is assigned to its creator**, the person whose connection you use.
  Assign deliberately with `commander_task_assign`, by **user ID** (`commander_user_list`), not
  email. Copy it exactly from the list.
- **Priority set through the MCP (1–4) does not show on the task card**, which only shows low,
  medium and high. Tell the person if priority matters.
- **Due dates are stored in UTC.** A date with no time is midnight UTC, which shows as the
  previous day for anyone west of UTC. Pass a full timestamp with the person's offset.
- **Markdown in `description` is shown as plain text**, characters and all.
- **No subtasks or nested layers through the MCP.** Create them in Commander.
- **`commander_task_list_mine`:** `filter: open` and `filter: completed` do nothing (open tasks come
  back either way). `show_completed: true` adds completed tasks; there is no completed-only view.
- **New tasks go to the bottom of their column.** Moving a task to another column keeps its old
  position number, so it can land part-way down.
- `commander_taskboard_tasks` and `waymaker_kanban_view` return the whole board, with no paging.
  On a big board, prefer `commander_search` or `commander_task_list_mine`.

### Documents

These are rich-text documents. Notes belong in MyVault.

- **`content` is what people see; `text_content` is not.** On `commander_document_create`,
  `text_content` becomes one plain paragraph, so markdown headings and lists show as literal
  characters. Send `content` as structured (Tiptap JSON) content. **When you send `content`, send
  a plain-text `text_content` too**: it is what search and summaries read, and nothing fills it
  in for you.
- **Do not update the `content` of a document someone has opened in Commander.** Once a document
  has been opened in the editor, the editor keeps its own copy. `commander_document_update` changes
  the stored content, reports success, and the person still sees the old text; their next edit then
  overwrites yours. Only update `content` on a document you created and nobody has opened. To
  change an existing document, create a new one (or a new version of it) and tell the person.
- **Who can see a document follows where it's filed.** A document in a project or workspace is
  visible to the people who can open that project or workspace. File it in the right place.
  Don't rely on `status` (`private`, `published`, `archived`) to share it.
- `commander_document_list` ignores `folder_id` and returns every document's full content. Filter
  by project.
- From the CLI, `waymaker docs create -c` and `docs update -c` set only the search text: the
  document opens empty, or unchanged.

### Sheets

- **Rows and columns are zero-indexed** in `commander_sheet_set_cell`, in `col_index` for
  `set_column_type` / `remove_column_type`, and in what `get_cells` returns. A1 ranges are used
  everywhere else.
- **Before writing to a sheet people edit in Commander, read it.** If `commander_sheet_get_cells`
  returns no cells for a sheet you can see has data, **stop and do not write**: the MCP cannot read
  that sheet's layout, and a write can damage it. Tell the person. Sheets created through the MCP
  and only written through it are fine.
- `set_range` and `apply_formula` take up to 1,000 cells per call, and `set_range` is atomic: one
  bad cell aborts the whole write. **Missing cells in a ragged `values_2d` are skipped**, not
  blanked; check `cells_written`. `null` writes an empty cell.
- **Formulas are not calculated by the server.** `apply_formula` stores the formula with an empty
  value; `get_cells` returns stored values, not results. The person sees results in Commander.
- **Every write saves a full copy, with no check for concurrent writers.** Don't write to the same
  sheet from two agents at once; the later write wins.
- `get_cells` returns at most 5,000 cells. Check `truncated` and read in ranges.
- Declare typed columns *after* creating the sheet, with `commander_sheet_set_column_type`. To
  write a header into a typed column, pass `coerce: false`. A percent column turns `"15%"` into
  0.15 but keeps a bare `15` as 15 (1500%). A date column reads a date with no offset as UTC.
  A formula column refuses plain values.
- `commander_sheet_validate_formulas` checks syntax only.

### Goals

Create the objective (`commander_goal_create`), then add measurable key results
(`commander_key_result_create`). Through the MCP a goal can be active or completed. To cancel or
archive a goal, the person does it in Commander. Check the status you set by reading the goal back.

### People, roles and teams

- **Roles use `name`, `purpose` and `reports_to_id`**, not `title`, `description` or `reports_to`.
- **Changing roles needs an organisation owner or admin.** Inviting people does too.
- **`commander_role_update` with `reports_to_id` changes only the reporting line.** Any `name` or
  `purpose` in the same call is dropped. Make two calls. A reporting line can't be cleared through
  the MCP.
- **A new role is `active` even with nobody in it**, so `commander_role_list` with vacant-only
  misses it until someone has been assigned and unassigned.
- `commander_role_assign` both assigns and unassigns: pass `action: unassign` to remove someone.
  A role can hold several people, and assigning someone does not remove their other roles.
- **`commander_user_get` returns the whole list**, whatever `user_id` you pass. Use
  `commander_user_list` and pick the person out; it includes people of every status, so check
  `status`.
- `commander_user_invite` only *stages* a person. The invitation goes out when an admin activates
  them, either from the admin app or with `waymaker users activate`.
- To find who knows a subject, use `waymaker_people_find`. It matches skills from people's
  assessments, so it finds nobody in an organisation that hasn't taken them.

### Calendar

- **Pass `calendar_id` and `end_time` on every `commander_event_create`.** The server refuses
  without them, though the tool lists only `title` and `start_time` as required.
  `commander_calendar_list` gives the `calendar_id` values, including connected external
  calendars.
- **Always give times with a UTC offset** (`2026-10-06T09:00:00+10:00`). A time with no offset is
  read as UTC, not the person's local time.
- **All-day events:** pass the start as `YYYY-MM-DDT00:00:00Z` for the date you mean, and the end
  as midnight the next day. A local midnight with a positive offset lands on the previous day.
- **Attendees added through the MCP do not reach anyone today**, and Waymaker does not send
  invitations itself. To invite people, the person adds them in Commander or their calendar app.
- **Only update an event ID you got from `commander_event_list` or `commander_event_get`.** An
  update to an ID that doesn't exist can create a new "Untitled Event" instead of failing.
- **`commander_event_list` shows events that start inside the range.** An event that started
  before the range and is still running is missed, and a recurring event appears once, at its first
  occurrence. Widen the range, and read `rrule` for repeats, before saying a time is free.
- The CLI `waymaker calendar` commands are unreliable; use the MCP tools.

### Files

`commander_file_*` cover the person's own files (My Files). These are separate from Host storage
buckets. A large file added to MyVault is stored in My Files too, but add it with the vault tools
(§3) so the vault keeps its link to it. "I can't upload anything" is answered by
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
