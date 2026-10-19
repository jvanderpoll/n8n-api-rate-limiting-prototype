# API rate limiting prototype: how it works

A single-file, no-build HTML prototype of rate limiting settings for the n8n public API (Linear API-257). Open `index.html` in a browser. There is no framework, bundler or dependency, apart from the Inter font loaded from Google Fonts.

This file is the source of truth for continuing the work. It covers what the prototype does, every state, the exact copy, the rules we decided, and how the code is organized. `CONTEXT.md` has the history and the research behind it, but it was written earlier and is partly outdated (it still describes per-key limits and an older layout). Where the two disagree, this file wins.

## Files

| File | What it is |
| --- | --- |
| `index.html` | The prototype. CSS, markup and JavaScript in one file, with a `noindex` meta tag. |
| `PROTOTYPE.md` | This document. |
| `CONTEXT.md` | Background, sources, research and design history. Partly outdated. |

## What it shows

Two pages of the n8n settings area, recreated from screenshots of the real product, with the new rate limiting UI on the Policies page.

1. **Policies page.** The starting page. A "Limits" section with one card holding two rows: an inert "Usage limits" row (Jan's placeholder) and our "Enable API key rate limits" setting.
2. **API keys page.** The existing page, unchanged except for a banner that points to Policies. It has no rate limit column and no per-key limits.

## Review controls (not part of the product)

A bar above the page is for reviewing only:

- **Instance type:** Self-hosted or Cloud. Switching resets all state (see State model).
- **Navigate to:** Policies page or API keys page.

The prototype opens on the Policies page, on Self-hosted, with rate limiting off.

## The design in one paragraph

Limits are global: they apply to every API key. They live on the Policies page as one setting with a toggle. Turning it on opens a dialog that holds all four limits (reads and writes, for all endpoints and for individual endpoints). An empty field is unlimited. Once enabled, the setting shows the limits written out below it, with one edit icon that reopens the dialog. On cloud the feature is always on and the toggle is locked, so admins can only lower limits or reset them.

## Policies page

Layout, top to bottom:

- Title "Policies". No back link or breadcrumb.
- Subtitle: "Rules for everyone on this instance: who must sign in how, what workflows may use, and how much they may run. Learn more in the documentation ↗".
- Section title "Limits", then one card with two rows separated by a divider.

**Row 1, "Usage limits" (inert).** Title "Usage limits", description "Cap production executions per period.", status "No instance limit · 3 projects limited" with a chevron. It does nothing. It exists to show where our row sits.

**Row 2, the setting.**

- Title: "Enable API key rate limits".
- Description: "Cap how many requests your API keys can make with rate limits ↗". "API keys" is a link to the API keys page. "rate limits" is the documentation link (placeholder, goes nowhere). The arrow is glued to its link so it can never wrap onto a line of its own.
- A toggle on the right.
- When enabled, a values block appears below, with spacing but no divider.

### The values block

Shown only while rate limiting is on. Two lines, one per group, with the values right-aligned and a single edit (pencil) icon at the far right.

- Left labels: "All endpoints" and "Individual endpoints", each with a small info icon. The icon tooltip explains the group: "Combined requests from one key to every endpoint." and "Counted separately for each endpoint." The tooltip is visible on hover and on keyboard focus.
- All text in the block is the primary color. The info icons stay muted.
- The right edges of the toggle, the usage chevron and the pencil are aligned on one vertical line (the pencil and chevron have small negative margins to compensate for empty space inside their icons).

**How a group's value is written.** Numbers are formatted with thousands separators, windows are always in seconds, and the parts are separated with a middle dot ( · ).

| What is set in the group | Text |
| --- | --- |
| Reads only | `100 reads / 60 seconds · unlimited writes` |
| Writes only | `unlimited reads · 5 writes / 60 seconds` |
| Both, same window | `2 reads · 2 writes / 60 seconds` |
| Both, different windows | `2 reads / 60 seconds · 2 writes / 120 seconds` |
| Neither | `unlimited reads and writes` |

Singular forms are handled: `1 read / 1 second`. Both groups always show, so there are always two lines.

## The dialog

One dialog serves both enabling and editing. It is opened with `openL('enable')` or `openL('edit')`.

| | Enable mode | Edit mode |
| --- | --- | --- |
| Opened by | Turning the toggle on | The pencil icon |
| Title | Enable API key rate limits | Edit API key rate limits |
| Primary button | Enable limits | Save changes |

Subtitle in both modes: "Limits how many requests each API key can make. Requests over the limit get a 429 response until it resets."

**Fields.** Two groups, "All endpoints" and "Individual endpoints" (each with the same info icon as the summary). Each group has two rows, Read and Write. Every row is `[limit] requests per [window] s`, and all fields are always visible. The window is whole seconds only: there is no unit dropdown. The limit placeholder is "Unlimited". The window defaults to 60 and is dimmed while its limit is empty.

**Empty means unlimited.** Leaving a limit empty means no limit. Typing 0 (or 00, and so on) is accepted and turns into an empty field when you leave it. If a limit becomes empty while its window is invalid, the window resets to 60.

**Overrides and defaults.** A row whose value differs from its default gets a stronger border and a small reset icon at its right. Hovering the icon shows "Reset to default: X" (X is the default value, or "Unlimited"). Each row resets on its own. In self-hosted the default is unlimited. In cloud the defaults are 1,000 / 200 reads and writes for all endpoints, 100 / 50 for individual endpoints, all per 60 seconds.

**Footer**, left to right: a "Reset all" link (only visible when something differs from the defaults), a red status message, Cancel, and the primary button.

**Button states.**

- Enable mode: the button is disabled until at least one limit has a value. It looks disabled, shows the forbidden cursor, and a tooltip on hover explains: "Set at least one limit to enable rate limiting."
- Edit mode: the button is always enabled, so you can save to close even when nothing changed.
- Edit mode, self-hosted, every field empty: the button becomes "Disable limits", and a line on the left of the footer says "With no limits set, rate limiting is disabled." The footer line replaces the status message, so the dialog never changes height. Clicking it saves the empty values and turns the feature off.

**Closing.** Cancel, the close icon, Esc and a click on the overlay all close without saving. Edits are a draft until the primary button is used.

## Validation

Errors appear as helper text under the row, after you leave the field. The invalid value stays in the field and the field gets a red border. Save is blocked while an error remains: it focuses the first invalid field and shows "Fix the highlighted fields." in the footer. The message clears when you edit.

| Rule | Message |
| --- | --- |
| A limit must be a whole number, 1 or more (0 and empty mean unlimited) | Enter a whole number, 1 or more. |
| A window must be a whole number of seconds, 1 or more (checked only when the limit is set) | Enter a window in seconds, 1 or more. |
| Cloud: a field cannot be emptied | Cloud limits can't be removed. Reset to restore the default. |
| Cloud: a value cannot be looser than the default (compared by rate, so 1,000 per 120 s is stricter than 1,000 per 60 s) | Can't exceed the cloud default, 1,000 / 60 s. |

**Soft warning (amber, does not block saving).** If a limit on individual endpoints is higher than the matching limit on all endpoints (same kind, compared by rate), the row shows: "Higher than the all-endpoints limit, so it never applies."

Invalid text such as "abc" counts as a change, so the enable button becomes enabled, and clicking it shows the error.

## Flows and states

**Self-hosted, first load.** Toggle off, no values block.

**Enabling.** Click the toggle. The dialog opens in enable mode, prefilled with any stored values. The toggle stays off until the button is used. Cancel leaves it off. After "Enable limits", the dialog closes, the toggle turns on and the values block appears.

**Editing.** Click the pencil. The dialog opens in edit mode with the saved values. "Save changes" commits them and closes.

**Turning off with the toggle.** Clicking an on toggle turns it off immediately. The stored values are kept, so enabling again opens the dialog prefilled, and the button is enabled because values exist.

**Turning off by clearing.** Clearing every field in the edit dialog offers "Disable limits". Saving stores the empty values and turns the feature off. Enabling again then opens an empty dialog with the button disabled.

**Cloud.** The toggle is on and locked, with a forbidden cursor and a tooltip on hover: "n8n Cloud keeps rate limiting on to protect the platform. You can set lower limits, but not higher ones." The values block always shows, with the defaults until edited. Only the edit dialog is used. Its button is always "Save changes" (never "Disable limits"), fields cannot be emptied, and values cannot go above the defaults.

**Switching Instance type.** Resets everything: self-hosted becomes off with no values, cloud becomes on with the defaults. All dialogs close.

## API keys page

- Sidebar with "n8n API" highlighted. Title "API" with a subtitle that has three links (API playground, Webhook node, documentation), each with its arrow glued to the link.
- Tabs "Mine" and "All", a search field and a "Create API key" button.
- Table with columns Name, Scopes, Expiration, Last used, and a row actions menu (three horizontal dots) with Edit and Revoke. There is no rate limit column and no per-key limit action.
- Pagination (static).
- Banner at the bottom: "Configure rate limits for your API keys in Policies." The word "Policies" is a link that opens the Policies page. The text does not say whether rate limiting is on or off.
- "Create API key" opens a static preview of today's modal. Only the scope cards (All 107, Read only 34, Custom) and closing work. "Custom" expands a scope list where the Workflow group expands further. Rate limits are deliberately not part of key creation.

## State model

All state lives in memory. A reload restores the starting state.

| Variable | Meaning |
| --- | --- |
| `CLOUD` | `true` for Cloud, `false` for Self-hosted. |
| `ON` | Whether rate limiting is enabled. Self-hosted starts `false`. Cloud forces `true`. |
| `lmode` | `'enable'` or `'edit'`, the current dialog mode. |
| `saved` | The committed limits. |
| `D` | The dialog draft, as raw strings. |
| `DEFC` | The cloud defaults and ceiling. |

Limits are keyed `ra`, `re`, `wa`, `we` (reads or writes, all or each endpoint). `saved[k]` is `{ l: number | null, w: seconds }`, where `null` means unlimited. The draft `D[k]` is `{ l: string, w: string }` so invalid input can stay visible while it is being fixed.

## Code map (all in `index.html`)

| Area | Where to look |
| --- | --- |
| Theme tokens | `:root` CSS variables. They follow `prefers-color-scheme`. Orange is `--or`, text is `--tx` (primary) and `--tx2` (secondary). |
| Icons | `P` (path strings) and `ic(name, size)`: inline SVG. |
| Navigation | `showPage('api'\|'pol')`, `setSide()`, `go()`. |
| Limits helpers | `def`, `isCust`, `fmt`, `resetAll`, `raw`, `num`, `parseK`, `resetDraft`, `custD`. |
| Validation | `check(k)` (errors) and `warn(k)` (the amber warning). |
| Dialog | `lrow()` builds a row, `renderL()` redraws the dialog and the button state, `openL(mode)` and `closeL()`. Handlers on `#lgrid`, `#lsave`, `#lresetall`. |
| Setting row | `renderPol()` syncs the toggle, the values block, the cloud lock and the banner. The toggle handler is on `#etoggle`. |
| Instance type | `setView(cloud)`. |
| Table and menu | `renderTable()`. |
| Create API key modal | `renderScopeRows()`, `setScope()`, `closeC()`. |

Key element ids: `etoggle` and `ewrap` (toggle and its tooltip wrapper), `limVals` and `limSum` (values block), `pen` (edit icon), `layer2` (dialog), `lgrid`, `lnote`, `lstat`, `lresetall`, `lcancel`, `lsave` and `lsaveWrap` (button and its tooltip wrapper), `lkKeys` and `goPol` (links), `bnTxt` (banner), `vSelf` and `vCloud`, `pPol` and `pApi`.

Rule of thumb: after changing state, call `renderPol()` (or `renderTable()`, which calls it).

## Styling notes and gotchas

- **Text colors.** The values block is all primary. Elsewhere the rule is two text colors only: primary and secondary.
- **Icons are blocks by default.** There is a global `svg{display:block}`. Icons inside description text must be inline, which `.sub svg, .d svg` handles. Every link arrow sits in a no-wrap wrapper with its link so it cannot land on its own line.
- **Tooltips.** `.tip` with a `data-tip` attribute. `.tip.r` opens right-aligned and `.tip.r.up` opens above. Multi-line tooltips use `white-space: pre-wrap`.
- **Disabled button tooltip.** A disabled button gets no mouse events, so the tooltip and the forbidden cursor live on the wrapper `#lsaveWrap`.
- **No layout jump.** The disable note sits in the footer row, not under the fields, so the dialog height stays fixed.

## Decisions that are settled

- Limits live on the Policies page, not on the API keys page.
- Limits are global. Per-key limits are out of scope for now.
- All four limits sit in one dialog behind one setting.
- Empty means unlimited. A toggle turns the feature on or off, and enabling needs at least one limit.
- Windows are in seconds only.
- On cloud, limits can be lowered or reset but not raised or removed.
- The setting shows its limits written out, so admins who cannot edit can read them.

## Assumptions and open questions

- The API treats 0 as "no limit" (the UI accepts it).
- Whether cloud needs a lower bound, and the real cloud defaults. The prototype uses its starting defaults as the ceiling.
- Whether reads (GET and HEAD) and writes (every other request) is the right split.
- Whether one value for individual endpoints is enough.
- What should happen to stored limits when the feature is switched off (the prototype keeps them with the toggle and drops them when cleared).
- The Usage limits row is an inert placeholder. Jan's version navigates to a subpage with a chevron. Ours opens a dialog from a pencil. This is intentional and for him to reconcile.

## Known simplifications

- Only the Workflow scope group expands in the Create API key modal, and the scope list is partial.
- Search, filters, pagination and the Edit and Revoke items do nothing.
- The "rate limits" and "documentation" links go nowhere.
- Sidebars use generic icons.
- Nothing is persisted.

## Testing checklist

Walk these after any change:

1. Self-hosted, off: the dialog opens from the toggle, the button is disabled with a tooltip, the toggle stays off after Cancel.
2. Set one limit: the button enables. Enable it: the toggle turns on and the values block appears with the right phrasing.
3. Edit: nothing changed and Save changes closes. Clear every field: "Disable limits" and the footer note appear, with no height change. Click it: the feature turns off.
4. Toggle off and on: the values come back prefilled.
5. Validation: "abc", a decimal, a negative, 0, an empty window, and the amber warning.
6. Cloud: the toggle is locked with a tooltip. Raising a value or emptying a field is rejected. Lowering works. Reset works.
7. API keys page: the banner link opens Policies, and "API keys" in the setting description opens the key list.

## Continuing in Cursor

Open the folder, open `index.html`, and serve it with any static server (`npx serve .`) or open the file directly. Keep it one file. Handy prompt: "Read PROTOTYPE.md, then change X. Keep the settled decisions intact, call renderPol() after any state change, and write copy in sentence case without em dashes."
