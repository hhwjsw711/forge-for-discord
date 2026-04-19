Created: 2026-04-19 03:02 UTC
Last Updated: 2026-04-19 03:02 UTC
Status: Done

# Slash command debug logging and form unpublish

## Problem

Two separate pains fell out of the 2026-04-19 Discord interactions endpoint routing bug.

1. When a member runs a slash command in Discord, the HTTP handler collapses three very different failure modes into one user-facing string ("This form is not published yet.") with zero log output. The three modes are:
   a. The Discord guild id carried by the interaction does not match any installed `guilds` row. This is the one that bit us. Interactions were being routed to the dev Convex deployment while the form lived in prod, so `getByCommand` returned null every time and we could not tell from logs.
   b. A `guilds` row exists but has no matching form for the command name. Happens if the command was registered by hand or if the form was deleted.
   c. A form exists but `published` is false (draft, or auto-reverted after a config edit).
   All three silently log nothing, which made the original deployment-mismatch bug invisible.

2. There is no way to unpublish a form once it has been published. Admins can edit a form and Discord will keep serving the old command, or they can delete the form (which also cascades submissions and audit history). There is no "remove this slash command from Discord and mark the Forge form as draft" path.

## Root cause for bug 1

`convex/http.ts` around line 131 reads:

```ts
const form = await ctx.runQuery(internal.forms.getByCommand, { discordGuildId, commandName });
if (!form || !form.published) {
  return ephemeralResponse("This form is not published yet.");
}
```

Two different failure modes, one generic response, one code path, zero logging.

## Proposed solution

### 1 Differentiated logging

Split the branch into two distinct paths:

- If `form` is null, call `console.warn("slash_command_form_not_found", { discordGuildId, commandName })` and return the ephemeral message unchanged. Do not write an audit row. We do not have a form id to attach it to, and writing to a guild-scoped audit log based on an un-verified Discord guild id would let anyone spam rows into a neighbor's audit log by shouting at the wrong endpoint.
- If `form.published === false`, call `console.warn("slash_command_form_unpublished", { discordGuildId, commandName, formId: form._id })` AND write an audit row via a new internal mutation `recordUnpublishedSlashAttempt`. Action name: `slash_command_unpublished_attempt`, severity info (falls through the default branch of the existing ERROR_ACTIONS set in `convex/auditLog.ts`). Metadata carries `commandName` and the actor id (Discord member user id when present).

### 2 Unpublish flow

Add a new action `convex/discord.ts#unregisterCommand` that:

1. Authenticates the caller with `requireAllowedWorkspaceUser` (same guard that `registerCommand` uses).
2. Reads bot credentials + the current `discordCommandId` via a new internal query `internal.forms.getForUnpublish`. Unlike `getForPublish` this one skips `validateFormReadyForPublish` because we do not need the form to be ready, we need the command to go away.
3. If `discordCommandId` is set, issues `DELETE /applications/{appId}/guilds/{guildId}/commands/{commandId}`. Treats 204 and 404 as success (404 means the command was already removed on Discord side, for example by a user deleting the bot install and reinstalling).
4. Calls `internal.forms.setPublicationState` with `published: false` and `discordCommandId: undefined` to flip the DB row.
5. Calls a new internal mutation `internal.forms.recordCommandUnpublished` to write a `form_unpublished` audit row so the Results log reflects the action.

Wire a confirmation dialog + button into `src/pages/EditForm.tsx`. The button:

- Only renders when `form.published === true` or `form.discordCommandId` is set.
- Lives next to the Publish button in both the sticky top toolbar and the fields-pane bottom row, matching the existing pair pattern.
- Opens a `UnpublishConfirmDialog` component (mirrors `ConfirmDeleteDialog` in `FormResults.tsx`, uses the site design system, no browser confirm).
- Calls the new action, shows a success banner, and updates `isPublished` locally.

## Files to change

- `convex/http.ts` — split the branch + log.
- `convex/forms.ts` — add `recordUnpublishedSlashAttempt`, `getForUnpublish`, `recordCommandUnpublished`.
- `convex/discord.ts` — add `unregisterCommand` action.
- `src/pages/EditForm.tsx` — add the Unpublish button, confirm dialog, and wiring.
- `changelog.md`, `files.md`, `TASK.md` — doc sync after the work lands.
- `prds/slash-command-debug-logging-and-unpublish.md` — this file.

## Edge cases

- Call to `unregisterCommand` when `form.published === false` and `discordCommandId` is unset: no Discord call, flip the flag idempotently, still write the audit row so there is a trace.
- Discord returns 401 or 403 on the DELETE because the bot token got rotated: surface a `ConvexError({ code: "discord_unregister_command_failed", status })` so the UI banner can say "Could not unpublish on Discord side. Reconnect the server and try again."
- Someone disconnects the guild between editor load and button click: `getForUnpublish` returns null, action throws `form_not_found`, UI banner falls back to the generic error.
- Two admins click Unpublish at the same time: the second request hits a form where `discordCommandId` is already undefined, so the DELETE is skipped and the second `setPublicationState` is a no-op patch. Safe.
- Someone runs the slash command between "flag flipped to false" and "Discord DELETE returns": the HTTP interaction handler hits the "form not published" branch, logs + writes the audit row. Expected.

## Verification steps

- `npx tsc --noEmit -p tsconfig.app.json` clean
- `npx tsc --noEmit -p convex/tsconfig.json` clean
- `ReadLints` on every touched file returns no new errors
- Manually call the unpublish path in dev once the code lands (re-register a command, then unpublish, verify Discord stops showing the slash command and the audit log on the Results page shows `form_unpublished`)

## Task completion log

- [x] 2026-04-19 03:02 UTC PRD drafted
- [x] 2026-04-19 03:02 UTC Backend logging split in `convex/http.ts`
- [x] 2026-04-19 03:02 UTC Backend helpers in `convex/forms.ts` + `unregisterCommand` action in `convex/discord.ts`
- [x] 2026-04-19 03:02 UTC Unpublish button and confirm dialog in `src/pages/EditForm.tsx`, new audit labels in `src/pages/FormLogs.tsx`
- [x] 2026-04-19 03:02 UTC `npx tsc --noEmit` on both projects, `npx eslint` on touched files, and `ReadLints` all clean
- [x] 2026-04-19 03:02 UTC Docs sync: `changelog.md`, `TASK.md`, `files.md`
