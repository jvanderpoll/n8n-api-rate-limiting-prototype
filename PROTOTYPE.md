# API rate limiting prototype: how it works

A single-file, no-build HTML prototype of rate limiting for the n8n public API. Open `index.html` in a browser. There is no framework, no bundler and no dependencies, apart from the Inter font loaded from Google Fonts.

This file is for continuing the work in Cursor. It covers what the prototype does, the rules we decided, how the code is organized, and how to change things safely.

## Files

| File | What it is |
| --- | --- |
| `index.html` | The prototype. About 510 lines: CSS, markup and JavaScript in one file. Has a `noindex` meta tag so search engines skip it. |
| `PROTOTYPE.md` | This document. |

The same code is published as a private Claude artifact (`api-rate-limiting-wireframe.html` is the identical file without the `noindex` tag). If you edit `index.html` locally, the artifact does not update on its own.

## What it shows

Two pages of the n8n settings area, recreated from screenshots of the real product, plus the new rate limiting UI on top.

1. **API keys page** (the real page today). Table of keys with columns Name, Scopes, Expiration, Last used, Rate limits, and a row actions menu (three horizontal dots). A banner at the bottom links to Policies.
2. **Usage limits page** (Policies, based on Jan's prototype). Existing Executions and Projects sections are static copies for context. The new section is **Public API**, with the "Enable API rate limit" card.

Plus these modals: **Set rate limits on [key]** (per key), **Set / Edit API rate limits** (the defaults), and a **Create API key** modal that is a static picture of today's modal (only close and the scope cards work).

## Controls at the top (not part of the product)

The dashed bar is for reviewing, not part of the design:

- **Instance type:** Self-hosted or Cloud. Switching resets all state (see below).
- **Settings reside in:** Policies page or API keys page. Decides where the "Enable API rate limit" card lives. This is an open design question, so both versions exist.
- **Navigate to:** jump between the Policies page and the API keys page.

## The decided behavior

These came out of the design discussions and are implemented. Treat them as the spec.

1. **Defaults are global; keys can override.** Instance-level default limits apply to every key. A key can use its own limits.
2. **Eight values.** Two groups, **All endpoints** (combined for one key) and **Each endpoint** (counted separately per endpoint). Each group has **Reads** and **Writes**, and each of those is "N requests per window". Reads and writes are kept separate on purpose, which is why the layout does not follow Jan's two-value version.
3. **Window units.** The window field has a unit dropdown, `ms` or `s`, default `s`. Minimum window is 50 ms. This 50 ms figure is a placeholder from a meeting and still needs confirming with the API Hardening team.
4. **0 means no limit.** Entering 0 shows "No limit" in place of the window.
5. **Enabling flow (Jan's pattern).** The toggle triggers a modal. Fill in the limits, save, and the modal closes and the toggle turns on. The card then shows plain text with one pencil icon, which reopens the same modal. Cancel leaves the toggle off. Turning the toggle off collapses the rows and keeps the values.
6. **Keys can only restrict further.** If a default is set, a key's value must be less than or equal to it. Comparison is by rate (limit divided by window in seconds), so 1,000 per 120 s is stricter than 1,000 per 60 s. Entering 0 on a key is rejected when the default sets a limit. If a default is itself "no limit", a key can set anything.
7. **Lowering the defaults lowers keys.** Saving a lower default clamps any custom key above it down to the new default. This is silent today. Whether to notify the admin is an open question.
8. **No per-key limits while rate limiting is off.** The Rate limits column shows "Off" for every key, the "Set rate limits" menu item is hidden, and the modal cannot open. Custom values stay stored and return when it is turned back on.
9. **Cloud.** Rate limiting is on by default and the toggle is locked on, with a tooltip. Users can set lower limits but not higher ones: the cloud ceiling is the fixed constants in `DEF`. Zero is rejected on cloud.
10. **Per-key modal.** One checkbox, "Use default limits", checked by default for keys that follow the defaults. Unchecking it makes the fields editable, starting from the defaults. The fields are dimmed and read-only while checked. A second checkbox, "Apply setting to all API keys", applies the shown limits to every key on save (and resets every key if "Use default limits" is checked). It resets to unchecked each time the modal opens.
11. **The Rate limits column** shows "Default" or "Custom" with no values, to keep the table light. Hovering shows a tooltip with the values that apply to that key, read live from state.
12. **Banner on the API keys page** reads "Rate limiting is on|off. Configure the limits for your API keys in Policies." with Policies as an inline link. It shows no values. It only appears when the settings live on the Policies page.

## State model

All state is in-memory JavaScript variables. There is no storage. A page reload restores the starting state.

| Variable | Meaning |
| --- | --- |
| `CLOUD` | `true` for Cloud, `false` for Self-hosted. |
| `ENABLED` | Whether rate limiting is on. Self-hosted starts `false`. Cloud forces `true`. |
| `saved` | The current default limits (instance level). Shape below. |
| `keys` | Array of API keys. Each has `n` (name), `m` (masked key), `s` (scopes), `e` (expiration), `l` (last used), `c` (true if it has custom limits) and `v` (custom limits, only when `c` is true). |
| `KEYS0` | Deep copy of the starting `keys`, used for resets. |
| `DEF` | The starting defaults and the cloud ceiling. |
| `cur` | Index of the key open in the per-key modal. |
| `lmode` | `'enable'` or `'edit'` for the defaults modal. |

Limits shape (used by `saved`, `DEF` and `key.v`): keys are `ra`, `re`, `wa`, `we` (reads or writes, all or each). Each value is `[limit, windowSeconds]`.

```js
{ ra:[1000,60], re:[100,60], wa:[200,60], we:[50,60] }
```

A window below one second is stored as a decimal (50 ms is `0.05`). The UI shows it as `50 ms`.

Switching **Instance type** runs `setView()`, which resets `ENABLED` (true on cloud, false on self-hosted), `saved` and `keys`, and closes every modal.

## Code map (all inside `index.html`)

| Area | Where to look |
| --- | --- |
| Design tokens and theme | `:root` CSS variables at the top. They follow `prefers-color-scheme` for light and dark. Accent orange is `--or`, toggle green is `--green`. |
| Icons | `P` (path strings) and `ic(name, size)`. Inline SVG, no icon font. |
| Navigation | `showPage('api'|'pol')`, `setSide()`, `go()` for link elements. |
| Placement toggle | `setPlace('pol'|'api')`. Moves the single `#limCard` element between `#polSlot` and `#embedSlot`, so both placements use the same component. |
| Instance type | `setView(cloud)`. |
| Limit grids | `mkGrid(prefix)` builds the 8-field layout. Prefixes: `'m'` for the per-key modal, `'l'` for the defaults modal. Field ids look like `mral`, `mraw`, `mrau`, `mram` (prefix, reads or writes, all or each, then l = limit, w = window, u = unit, m = message). |
| Reading and writing fields | `setV(prefix, values)` and `getV(prefix)`. |
| Validation | `validate(prefix)`. Handles format, 50 ms minimum, cloud ceiling (`'l'` on cloud), key cap (`'m'`), the per-endpoint-higher-than-total warning, and the "No limit" display. |
| Defaults modal | `openL(mode)`, `closeL()`, handler on `#lsave`. Calls `clampKeys()` after saving. |
| Per-key modal | `openM(i)`, `closeM()`, `setRate()`, handler on `#msave`. |
| Table | `renderTable()` redraws rows, wires cell clicks, tooltips and the actions menu, then calls `renderPol()`. |
| Policies card and banner | `renderPol()` syncs the toggle, the summary text and the banner. |
| Tooltips | `tipFor(key)`, `showTip()`, `hideTip()` for the table. CSS `data-tip` for the others. |
| Create API key modal | `layer3`, `renderScopeRows()`, `setScope()`. Only the scope cards and the Workflow group expand are interactive. |

Rule of thumb: any change to state should end by calling `renderTable()`, because it also refreshes the policy card and banner.

## Design principles used

These guided the wording and structure, so keep them when changing things:

- Simple and intuitive first. Detail appears through interaction (tooltips, validation messages, conditional text) and not as permanent text.
- Reusable. Set once, reuse across keys.
- Transparent about control. When something is locked or capped, say by whom and why.
- Consistent everywhere (UI, API and environment variables should use the same names and structure).
- Strong defaults. Do not let people break an integration by accident.
- Plain language, sentence case, verbs on buttons. No em dashes. Avoid hardcoding numbers in explanatory text.

## Known simplifications

- Only the **Workflow and executions** scope group expands in the Create API key modal. The scope list is partial.
- Executions and Projects sections on the Policies page are static copies.
- The Create API key modal is inert except for the scope cards and closing.
- Page numbers, search, filters and the Edit and Revoke menu items do nothing.
- Sidebars use generic icons. The two pages use different sidebars on purpose (current product versus Jan's prototype).
- Nothing is persisted.

## Open questions

1. When rate limiting is switched off, should existing custom limits be kept (current behavior) or removed, with a warning?
2. When the global limit is lowered below a key's own limit, the key is silently lowered. Should the admin see a notice or a pre-save warning?
3. Where should the settings live: Policies page (current direction) or the API keys page?
4. Are milliseconds supported by the backend and API? Is 50 ms the right minimum?
5. Who can change per-key limits: any key editor, or instance admins only?
6. Does the main switch apply to all keys, including ones with custom limits? The prototype assumes yes.
7. Should the Rate limits column support a filter (Default or Custom) once there are many keys?
8. Cloud ceiling: the prototype uses the starting defaults as the cloud ceiling, which is an assumption. Cloud may also need a lower bound.

## How to continue in Cursor

1. Open the folder in Cursor and open `index.html`.
2. Serve it with any static server, for example `npx serve .`, or open the file directly in a browser.
3. Make changes in `index.html`. Keep it one file unless you decide to split it.
4. After a logic change, test the states that matter: Self-hosted off, Self-hosted on, Cloud, a key set to Custom, and the defaults lowered below a custom key.
5. Hosting: any static host works (GitHub Pages, Vercel). It makes network requests only for the Inter font.

Handy prompt for Cursor: "Read PROTOTYPE.md, then change X. Keep the rules under 'The decided behavior' intact and call renderTable() after any state change."

## Source material

- Ticket: API-257, "Add rate-limit settings UI" in Linear. It defines the eight values, global and per-endpoint layers, and 0 meaning no limit.
- Jan's policies prototype, for the toggle-modal-pencil pattern, the "Public API" section and the reads and writes definitions (reads are GET and HEAD requests, writes are every other request).
- The design doc in Notion, "API rate limiting: experience outline". It still describes an earlier version of the design (inline editing on the API keys page) and needs updating to match this file.
