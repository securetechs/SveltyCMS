# SveltyCMS Admin UI Audit: theme and UI consistency

- Repo: `SveltyCMS` branch `next`, commit `009bac1`
- Environment: local dev server `http://localhost:5173` (`bun run dev`), SQLite, fresh setup with the "Blog / Editorial" blueprint, SMTP skipped, non-essential cookies rejected
- Date: 2026-10-22
- Tooling: Python Playwright (Chromium), 1440x900 light and dark, 390x844 mobile (light) for the main routes. Full-page captures: when a page scrolls inside `AdminPageShell` the viewport was enlarged to the content height so the whole page is in the image.
- Screenshots: `..\shots\audit\<route>-<light|dark|mobile>.png` (128 files)
- Rules applied: `AGENTS.md` section 1 "Frontend / Admin UI" and `docs/contributing/style-guide-gui.mdx` (AdminPageShell at root, AdminCard per block, native controls from `@components/ui`, page action `variant="primary"`, preset tokens, status shade contract, logical properties, responsive at 390px).
- No code was changed. Nothing was pushed.

Method notes (so the results can be reproduced):

- The app firewall blocks the default headless Chromium user agent (`User-Agent is blocked`, `handle-security.ts:406`, and `Suspicious payload detected`). All captures use a normal desktop Chrome UA string.
- Theme was forced with the `theme` cookie plus `localStorage.theme` (`static/theme-init.js`, `src/stores/theme-store.svelte.ts:81`) and `prefers-color-scheme`.
- `iconify-icon` loads glyphs from the network, so every capture waits until all visible icons have rendered (max 30 s).
- Layout shift was measured with a `PerformanceObserver('layout-shift')` injected before load. Console errors and warnings were captured per page.
- The round icon at the bottom right of every screenshot is the Svelte dev inspector toggle (dev server only, see G9). It is not part of the product UI.

---

## 1. Routes enumerated

From `src/routes` (45 `+page.svelte`, 3 `+error.svelte`):

| Group | Route file | URL audited | Status |
| --- | --- | --- | --- |
| (app) | dashboard | `/dashboard` | audited L/D/M |
| (app) | `[language]/[...collection]` | `/en/collection/posts`, `/categories`, `/authors`, `/en/collection/posts?create=true` | audited L/D, M for posts list + create |
| (app) | mediagallery | `/mediagallery` | audited L/D/M |
| (app) | mediagallery/upload-media | `/mediagallery/upload-media` | audited L/D |
| (app) | user | `/user` | audited L/D/M |
| (app) | config | `/config` | audited L/D/M |
| (app) | config/access-management | `/config/access-management` | audited L/D/M |
| (app) | config/api | `/config/api` | audited L/D |
| (app) | config/automations | `/config/automations` | audited L/D/M |
| (app) | config/automations/[id] | `/config/automations/new` | audited L/D |
| (app) | config/design-system | `?tab=overrides, preview, themes, presets, layout, style, features, advanced` | audited L/D (all 8 tabs), M (themes) |
| (app) | config/monitor | `/config/monitor` | audited L/D/M |
| (app) | config/queue | `/config/queue` | audited L/D |
| (app) | config/redirects | `/config/redirects` | audited L/D/M |
| (app) | config/sync | `/config/sync` | audited L/D |
| (app) | config/system-settings | `/config/system-settings` (lands on `?group=cache`) | audited L/D/M |
| (app) | config/trash | `/config/trash` | audited L/D |
| (app) | config/webhooks | `/config/webhooks` | audited L/D/M |
| (app) | config/webhooks/logs | `/config/webhooks/logs` | audited L/D |
| (app) | config/workflows | `/config/workflows` | audited L/D/M |
| (app) | plugin | `/plugin` | redirects to `/config` (captured, same as config) |
| (app) | plugin/[...path] | none | not reachable: no plugin page link found |
| (app) | config/extensions | `/config/extensions` | OUT OF SCOPE (marketplace), light only |
| (app) | config/collectionbuilder | `/config/collectionbuilder` | OUT OF SCOPE (other developer), light only |
| (app) | config/collectionbuilder/[action]/[...contentPath] | none | OUT OF SCOPE, not opened |
| (app) | +error.svelte | `/config/this-does-not-exist`, `/this-route-does-not-exist-xyz` | audited L/D, M |
| (admin) | admin | `/admin` | audited L/D: renders blank (F1) |
| (admin) | admin/tenants | `/admin/tenants` | audited L/D: renders blank (F1) |
| root | email-previews | `/email-previews` | audited L/D/M |
| root | share/[token] | `/share/invalid-token-123` | audited L/D (404 page) |
| root | login | `/login` (chooser + sign in) | audited L/D/M |
| root | login/oauth | `/login/oauth` | audited L/D |
| root | login/+error.svelte | none | not triggered |
| root | setup | `/setup` steps 1 to 6 | audited L/D, M for step 1 |
| (site) | `/`, `[...slug]`, account/*, cart, checkout, legal/[page], shop, shop/[slug], widerruf (11 files) | none | public storefront, not admin: not audited |

Routes that failed to load: none returned an HTTP error, but `/admin` and `/admin/tenants` return 200 and render a completely blank page in both modes (F1).

---

## 2. Summary table

Shell/Card column: does the route file use `<AdminPageShell>` at root and `<AdminCard>` for content (from code, `grep`), y or n. Issue count = route specific issues in section 4 (systemic issues in section 3 are counted once there).

| Route | AdminPageShell / AdminCard | Issues | Worst issue |
| --- | --- | --- | --- |
| /setup (6 steps) | n / n (not an (app) route; uses its own wizard shell) | 11 | Raw `<span class=...>` HTML printed in the DB error message |
| /login (chooser, sign in) | n / n (not (app)) | 4 | Ignores dark mode entirely; cookie banner covers half the chooser |
| /login/oauth | n / n (not (app)) | 2 | Logo overlaps heading and token input |
| /dashboard | y / y | 4 | Page action uses `tertiary`; empty-state button is a restyled `outline` with custom classes |
| /en/collection/* list | n / n | 3 | No AdminPageShell; empty state text "No Collection posts Data" |
| /en/collection/posts?create=true | n / n | 7 | Mixed button colours and per mode chrome differences; toolbar clipped on mobile |
| /mediagallery | y / n | 4 | Two different colours for the same "upload" action on one page |
| /mediagallery/upload-media | y / n | 4 | Heading concatenated "Media Upload Drag files here to upload"; literal ">" in button |
| /user | y / y | 4 | 26 raw `<button>` elements; badge pills styled like buttons |
| /config | y / y | 3 | Migration tile styled differently from all other tiles |
| /config/access-management | y / y | 5 | Unstyled native checkboxes (browser teal) across the whole matrix |
| /config/api | y / y | 4 | Page header differs (no icon, no back button); truncated select |
| /config/automations | y / y | 2 | Dead override classes on AdminCard |
| /config/automations/new | y / y | 4 | Selected trigger tile unreadable in light mode; every event label printed twice |
| /config/design-system (8 tabs) | y / y | 9 | Literal `</Button`, `</Badge`, `</span`, `</a` text in labels |
| /config/monitor | y / y | 6 | Merged strings: "Security Incident Feed View All Logs", merged table header |
| /config/queue | y / y | 3 | Merged table header "JOB ID TASK TYPE STATUS ..." in first column |
| /config/redirects | y / y | 3 | Merged table header and duplicated placeholder |
| /config/sync | y / y | 4 | Native `<select>`; merged header "NAME TYPE" |
| /config/system-settings | y / y | 5 | Sticky footer covers content; mobile "Repair Cache" button loses its label |
| /config/trash | y / y | 1 | Empty state pattern differs from siblings |
| /config/webhooks | y / y | 2 | Page action `tertiary`; empty state without icon |
| /config/webhooks/logs | y / y | 2 | Error toast shown on a normal empty state |
| /config/workflows | y / y | 6 | Mobile header buttons overlap the title; third state node cut off |
| /admin | y / y | 1 | Page renders blank (height 0) |
| /admin/tenants | y / y | 1 | Page renders blank (height 0) |
| /email-previews | n / n | 4 | No admin shell, one-off header, broken logo image |
| /share/[token] (404) | n / n | 2 | Different error design; red banner covers the "404" |
| (app) +error (404) | y / y | 3 | "Page Not Found" printed three times; error icon turns green in dark |
| /config/extensions | y / y | out of scope | see section 6 |
| /config/collectionbuilder | y / y | out of scope | see section 6 |

Totals: 14 systemic issues (section 3), 113 route specific issues (section 4), 19 functional bugs (section 5), 3 out of scope notes (section 6). 32 route files audited, 128 screenshots.

---

## 3. Systemic issues (affect many routes)

G1. Accent colour changes hue between light and dark mode.
- What: the same element is blue (`tertiary-500`) in light and green (`primary-500`) in dark. Examples: config tile labels black in light, green in dark; setup intro text blue then green; setup step "completed" icons blue then green; 404 error icon blue then green; active sidebar item; tab underline. The pattern `text-tertiary-500 dark:text-primary-500` occurs 400 times in 115 `.svelte` files.
- Where: e.g. `src/routes/setup/database-config.svelte:310`, `src/routes/setup/review-config.svelte:127`, `src/components/ui/toggle.svelte:114`.
- Modes: light vs dark.
- Screenshots: `config-light.png` vs `config-dark.png`, `setup-step3-project-light.png` vs `-dark.png`, `error-404-app-light.png` vs `error-404-root-dark.png`.
- Note: this may be an intended brand choice. It is listed because it makes "primary" mean two colours and produces the inconsistencies in G2.
- Proposed fix: introduce one `--admin-accent` role token (set per mode in `utilities.css`) and replace the paired classes with it.

G2. Page primary action uses different variants on different routes.
- What: AGENTS.md says the page action is `variant="primary"`. Some routes use `primary` (green in both modes), others `tertiary` (blue in both modes), so in light mode the main button is green on some pages and blue on others, and in dark mode blue on some and green on others.
- `primary`: `config/automations/+page.svelte:232`, `user/+page.svelte:1276`, `mediagallery/+page.svelte:843`, API "Run Query Now".
- `tertiary`: `dashboard/+page.svelte:781`, `config/redirects/+page.svelte:194`, `config/webhooks/+page.svelte:229`, `config/workflows/+page.svelte:410`, `config/access-management/+page.svelte:160`. In `(app)` there are 59 `variant="tertiary"` vs 41 `variant="primary"`.
- Modes: both.
- Screenshots: `config-automations-light.png` (green), `config-redirects-light.png` (blue), `config-webhooks-dark.png` (blue in dark).
- Proposed fix: set every page header action to `variant="primary"`, keep `tertiary`/`ghost` for supporting actions.

G3. Two identical left arrows with different jobs, and the back button is not consistent.
- What: the sidebar collapse control (circle with left arrow next to the logo) looks identical to the `PageTitle` back button at the far right. The back button appears on 20 routes (`showBackButton`) but not on `/config/api` and `/config/workflows`.
- Where: `src/components/page-title.svelte:276-316`.
- Modes: all.
- Screenshots: `dashboard-light.png`, `config-api-light.png`, `config-workflows-light.png`.
- Proposed fix: use a distinct icon for sidebar collapse (e.g. `mdi:dock-left`) and set `showBackButton` uniformly on config sub pages.

G4. Corrupted translation strings show markup or merged sentences in the UI.
- What: several message values contain leftover JSX closing tags or two strings glued together. Visible on screen.
- Where (`src/messages/en.json`):
  - 2174 `theme_apply_instant` "Apply instantly — no reload needed</span" (same defect in all 8 locale files)
  - 2198 `theme_default` "Default</Badge"
  - 2204 `theme_export_json` "Export Theme JSON</Button"
  - 2213 `theme_import_theme` "Import Theme</Button"
  - 2226 `theme_marketplace_url` "marketplace.sveltycms.com</a"
  - 2240 `theme_reset_defaults` "Reset to Defaults</Button"
  - 2206 `theme_flat` "Flat No borders · No shadows" (title + description merged)
  - 2231 `theme_new_name_placeholder` "Theme name (e.g. ...) New theme name"
  - 1341 `monitor_audit_timestamp` "Timestamp User / Actor Event Type Result Details SHA-256 Hash"
  - 1356 `monitor_quick_links` "Quick Links Jump to common admin tools"
  - 1359 `monitor_security_feed` "Security Incident Feed View All Logs"
  - 1479 `queue_job_id` "Job ID Task Type Status Attempts Next Run Created Actions"
  - 1511 `redirects_from_path` "From Path To Path Type Status Actions"
  - 1518 `redirects_search_placeholder` "Search by path... Search redirects by path"
- Modes: all.
- Screenshots: `config-design-system-*`, `config-monitor-light.png`, `config-queue-light.png`, `config-redirects-light.png`.
- Proposed fix: correct the message values (content only, no styling change).

G5. AdminCard instances carry hand-written surface classes that the component ignores.
- What: many routes pass `bg-white dark:bg-surface-900/50` (or `/20`, or solid), `shadow-sm/xs/xl`, `border border-surface-500/30` and even `border-2 border-dashed` to `<AdminCard>`. AdminCard sets `background-color`, `border-*` and `box-shadow` inline (`src/components/admin-card.svelte:43-53`), so these classes are dead code and the dashed empty-state border becomes solid. They also contradict the style guide ("omit variant/preset so themes apply", no `bg-white`).
- Where: `config/automations/+page.svelte:246,324,339,366`; `config/automations/[id]/+page.svelte:469`; `config/monitor/+page.svelte:122,156,190,224,271,336,369,398`; `config/redirects/+page.svelte:206,318`; `config/webhooks/logs/+page.svelte:105,123`; `config/workflows/+page.svelte:416,627`; `(admin)/admin/tenants/+page.svelte:75`.
- Modes: code level.
- Proposed fix: drop the surface/border/shadow classes from these AdminCard calls and keep only layout classes (`p-6`, `space-y-4`, grid).

G6. Empty states have no common pattern.
- What: icon or no icon, CTA colour green or blue or none, inside AdminCard or bare on the page, different heading sizes.
  - dashboard: no card, large icon, blue CTA built from `variant="outline"` plus custom classes (`dashboard/+page.svelte:980-985`)
  - mediagallery: no card, grey icon, blue CTA
  - automations: card, blue icon, green CTA
  - webhooks: card, no icon, blue CTA
  - trash: card, grey icon, no CTA
  - collection list: no card, "No Collection posts Data" in accent colour
- Screenshots: `dashboard-light.png`, `mediagallery-light.png`, `config-automations-light.png`, `config-webhooks-dark.png`, `config-trash-light.png`, `collection-posts-list-dark.png`.
- Proposed fix: one shared empty state block (icon, title, muted text, optional `variant="primary"` CTA) inside an `AdminCard`.

G7. Toggle "off" uses the error colour.
- What: every switched off toggle is a solid red track with red "OFF" (`src/components/ui/toggle.svelte:114,137,145`), so harmless settings such as "Reduced Motion: off" look like errors.
- Screenshots: `config-design-system-features-light.png`, `config-design-system-overrides-light.png`.
- Proposed fix: off state `bg-surface-500/30` with muted text; keep error red for invalid states only. (Component documents green/red, so confirm intent.)

G8. Raw native checkboxes next to the shared Checkbox.
- What: raw `<input type="checkbox">` renders with Chrome's default teal tick and ignores the theme, while `<Checkbox>` elsewhere renders a themed square.
- Where: `config/access-management/permissions.svelte:418,485`, `config/access-management/sso-providers.svelte:578,630`, upload page "Optimize images before upload".
- Screenshots: `config-access-management-light.png`, `-dark.png`, `mediagallery-upload-light.png`; compare `config-automations-new-light.png` (shared Checkbox).
- Proposed fix: use `<Checkbox>` from `@components/ui`.

G9. Dev inspector toggle sits on top of primary buttons (dev server only).
- What: the round Svelte inspector button at bottom right overlaps "Next" (setup), "Save Theme", "Save" (system settings), "Complete". Measured on setup step 1: toggle at x1400 y860 32x32, Next button at x1341 y852 90x40.
- Where: `vite.config.ts:1537-1543` (`showToggleButton: "always"`, `toggleButtonPos: "bottom-right"`).
- Not in production builds (`shouldEnableInspector` returns false for build/test).
- Screenshots: `setup-step1-light.png`, `config-design-system-presets-light.png`, `config-system-settings-light.png`.
- Proposed fix: `toggleButtonPos: "bottom-left"` or `showToggleButton: "active"`.

G10. Mobile: floating nav button overlaps footer actions.
- What: the round blue floating button at bottom right covers the right side of sticky footers.
- Screenshots: `config-system-settings-mobile.png` (covers export/import buttons), `config-design-system-themes-mobile.png` (covers "Save Theme").
- Proposed fix: add bottom padding equal to the button size on sticky footers when the floating nav is visible, or move the button above the footer.

G11. Mobile page header crowding.
- What: page titles truncate to "D…", "Acc…", "Enterprise Mon…"; on workflows the action buttons sit on top of the title and the hamburger.
- Screenshots: `dashboard-mobile.png`, `config-access-management-mobile.png`, `config-monitor-mobile.png`, `config-workflows-mobile.png`.
- Proposed fix: let `PageTitle` actions wrap below the title under `md`, or collapse them into an overflow menu.

G12. Tab bars use four different styles.
- What: underline tabs (user, API, upload, access management in light), inverted filled pills where inactive tabs are solid blue and the active tab is pale (design system), green filled pill (sync), and access management tabs that gain a grey fill only in dark mode.
- Screenshots: `user-light.png`, `config-design-system-overrides-light.png`, `config-sync-light.png`, `config-access-management-dark.png`.
- Proposed fix: use the shared `Tabs` component from `@components/ui` everywhere with one visual style.

G13. "Unsaved changes" sticky bar on design system overflows the viewport.
- What: the bar uses `-mx-6` and extends past the right edge (detected on all design system tabs at 1440 and 390).
- Screenshots: `config-design-system-*-light.png`.
- Proposed fix: remove the negative margin or clip with the card padding.

G14. Mobile layout shift above the 0.1 "good" limit.
- `/config/design-system?tab=themes` CLS 0.282 (content block `px-2 pb-2 pt-4 space-y-6` moves when header actions render)
- `/en/collection/posts?create=true` CLS 0.121
- `/config/monitor` CLS 0.107 (the three stat cards resize after data loads)
- All desktop routes measured below 0.04.
- Proposed fix: reserve header action height and stat card min height.

---

## 4. Per route issues

### /setup (wizard)

1. Cookie banner overlaps the left step list and the legend. On step 1 the banner (fixed bottom left, `src/plugins/cookie-consent/cookie-consent.svelte:82`) covers the "Completed / Current Step" legend and the lower part of the step list. Light, dark. `setup-step1-light.png`, `setup-step1-dark.png`. Fix: on `/setup`, offset the banner to the content column or render it as a bottom bar above the footer.
2. Floating round icon overlaps the Next button. Verified: it is the dev inspector toggle (G9), not a product element. Light, dark. `setup-step1-light.png`.
3. Step legend clipped. Verified: the legend is clipped only while the cookie banner is open; after "Reject All" the full legend is visible at 1440x900. `setup-step1-light.png` vs `setup-step2-admin-filled-light.png`. Fix: same as issue 1.
4. Mobile: cookie banner covers the form and the Next button entirely. Mobile. `setup-step1-mobile.png`. Fix: compact bottom bar layout on small screens.
5. Database error shows raw HTML: `Database '<span class="text-tertiary-500 dark:text-primary-500 font-bold">sveltycms.db</span>' already exists`. `src/routes/setup/error-classifier.ts:119`, rendered by `connection-status.svelte:240`. Light, dark. `setup-step1-db-exists-error-light.png`. (Also listed as F2.) Fix: plain text name in the message, style it in the component.
6. Error panel details area in light mode is a heavy mid grey band, and the troubleshooting area is a brown/tan wash, low contrast for the small labels. In dark the same block is near black. Light. `setup-step1-db-exists-error-light.png` (`connection-status.svelte:325-339`). Fix: use `--admin-bg-chip` at the normal light value and `warning-500/10` as the contract says.
7. Intro text alignment changes per step: left on step 1 and 2, centred on step 3 and 4, left on step 6. `database-config.svelte:310`, `admin-config.svelte:118` (md:text-start) vs `project-config.svelte:50`, `system-config.svelte:368` (text-center). Light, dark. `setup-step1-light.png`, `setup-step4-system-light.png`. Fix: one alignment (`text-start`).
8. Language chips run the label into the badge: "ENGLISH (EN)TRANSLATED", no gap. `system-config.svelte:550-552`. Light, dark. `setup-step4-system-light.png`. Fix: add `ms-1` or a separator.
9. Review step: help icons are not aligned; label column width varies per group, some labels touch their icon ("Database Type:?", "Default Content Lang:?"), some rows have no icon ("Creates", "Shop language"). `review-config.svelte:120,200,244`. Dark, light. `setup-step6-complete-dark.png`. Fix: fixed label column width and consistent help icon placement.
10. Review step: "Not Configured (Skipped)" is shown in the success/accent colour next to a red cross. `review-config.svelte:298-300`. Dark, light. `setup-step6-complete-dark.png`. Fix: muted or warning text for the skipped state.
11. Blueprint carousel: the fourth card is cut at the right edge, including its complexity badge. Light, dark. `setup-step3-project-dark.png`. Fix: snap to whole cards or fade the edge.

### /login (chooser and sign in)

1. Dark mode has no effect; light and dark captures are identical (white left half, charcoal right half). `src/routes/login/+page.svelte` has 7 `bg-white` usages. Dark. `login-signin-light.png` vs `login-signin-dark.png`. Fix: use `--admin-bg-page`/`--admin-bg-card` for the panels.
2. Cookie banner covers the lower left quarter of the chooser including part of the Sign In panel. Light, dark, mobile. `login-chooser-light.png`. Fix: as setup issue 1.
3. Version pill "Ver. 0.9.0" is split across the two halves (half white, half dark). Light. `login-chooser-light.png`. Fix: give the pill its own background.
4. Sign In submit is a grey `surface` button while "Forgotten Password" is outlined; the main action is the least prominent element. Light. `login-signin-light.png`. Fix: `variant="primary"` for Sign In.

### /login/oauth

1. The round logo is drawn over the card heading ("Enter ... SignUp:") and over the Registration Token input. Light, dark. `login-oauth-light.png`. Fix: move the logo above the card or reserve space for it.
2. Dark band covers only the top 206 px; the rest of the page is light grey in both modes. Light, dark. `login-oauth-light.png`. Fix: full height background token.

### /dashboard

1. Page action "Add Widget" uses `variant="tertiary"` (`dashboard/+page.svelte:781`), see G2. Light, dark. `dashboard-light.png`.
2. Empty-state "Add Widget" is a `variant="outline"` button restyled with `rounded-full bg-tertiary-500 ... dark:bg-primary-500` (`dashboard/+page.svelte:980-985`); in dark the two "Add Widget" buttons on the same screen are different colours (header blue, body green). Dark. `dashboard-dark.png`. Fix: plain `<Button variant="primary">`.
3. Header shows a narrow tall bordered robot icon button (about 22x40 px) that does not match the 40x40 icon buttons next to it. Light, mobile. `dashboard-light.png`, `dashboard-mobile.png`. Fix: standard icon `Button` size.
4. Empty state is not inside an `AdminCard` (G6). `dashboard-light.png`.

### /en/collection/{posts,categories,authors} (list)

1. Route has no `AdminPageShell` or `AdminCard` (`[language]/[...collection]/+page.svelte`). The style guide's "Content insets" section describes this as deliberate (EntryList pads itself), but the header is a different design from every other admin page (small "COLLECTION" caption, smaller title, toolbar of square icon buttons). Light, dark. `collection-posts-list-light.png`. Fix: render the list header through `PageTitle` so it matches other pages.
2. Empty state reads "No Collection posts Data" (`src/messages/en.json:4`, `entry-list.svelte:2023`), uses lowercase collection name and accent colour text, no card. Light, dark. `collection-posts-list-dark.png`. Fix: "No entries in Posts yet" in the shared empty state (G6).
3. Header title uses lowercase "posts" while the create view shows "Posts". `collection-posts-list-light.png` vs `collection-posts-create-light.png`. Fix: use the collection label consistently.

### /en/collection/posts?create=true (edit form)

1. Buttons in one form use four colours: "Select Entry" / "Add Entries" bright green filled, "Linked" grey filled, links and icons blue, SEO device toggles bright blue with an orange "Heatmap". Light, dark. `collection-posts-create-light.png`, `-dark.png`. Fix: supporting actions `tertiary`/`ghost`, one accent.
2. Status badge reads "UNPUBLISH" (a verb) in orange; the "Unpublish" control below is red text on grey, low contrast in dark. Light, dark. `collection-posts-create-dark.png` (`status_unpublish`, `en.json:2103`). Fix: badge shows state ("Unpublished") with the status shade contract.
3. Language selectors ("EN") get a grey filled background in dark but none in light; Edit/API tab backgrounds differ (black vs dark grey) in dark only. Dark. `collection-posts-create-dark.png`. Fix: same chrome tokens in both modes.
4. Field inputs use the field name as placeholder ("title", "excerpt"). Light, dark. `collection-posts-create-light.png`. Fix: meaningful placeholder or none.
5. SEO block uses a different label style (smaller bold labels, bordered `{}` buttons) than the main fields (larger labels, unbordered `{}`). Dark. `collection-posts-create-dark.png`. Fix: reuse the main field label component.
6. Right sidebar footer: "Created" and "Updated" values show "-" and the date wraps to its own line, not aligned. Light, dark. `collection-posts-create-light.png`. Fix: two column definition list.
7. Mobile: the rich text toolbar is cut off at the right without scroll affordance; the header crowds language select, save, filter and close into one row. Mobile. `collection-posts-create-mobile.png`. Fix: `overflow-x-auto` on the toolbar, wrap header actions.

### /mediagallery

1. "Upload" (header) is green `primary`, "Upload First File" (empty state) is blue, same action. Light, dark. `mediagallery-light.png`. Fix: same variant.
2. Toolbar button shows only "S" (thumbnail size small, `mediagallery/+page.svelte:112`) with no visible meaning. Light, dark. `mediagallery-light.png`. Fix: icon plus tooltip or "Size: S".
3. Sidebar folder edit button is solid orange (warning colour) for a neutral action. Light, dark. `mediagallery-light.png`. Fix: `ghost` or `surface` icon button.
4. No `AdminCard` used (code: 0); toolbar and empty state sit directly on the page canvas. `mediagallery-light.png`. Fix: wrap toolbar and grid in `AdminCard`.

### /mediagallery/upload-media

1. Heading concatenates two strings: "Media Upload Drag files here to upload". Light, dark. `mediagallery-upload-light.png`. Fix: separate title and hint lines.
2. Button text contains a literal ">": "> Browse Files" (`en.json:1165`). Light, dark. `mediagallery-upload-light.png`. Fix: remove the character, use an icon if needed.
3. "Optimize images before upload" is an unstyled native checkbox (G8). `mediagallery-upload-light.png`.
4. No `AdminCard` used (code: 0); dashed drop zone on grey. Light. Fix: wrap in `AdminCard`.

### /user

1. 26 raw `<button>` elements instead of `<Button>` (help icons, avatar edit, 2FA badge, privacy button), e.g. `user/+page.svelte:1039,1142,1180,1342,1649,1826,2072`. Code. Fix: `<Button variant="ghost" size="sm">` or the shared `HelpIcon` used in setup.
2. "ADMINISTRATOR" (green) and "ID: 01a164c21b69…" (blue) are full width filled pills that look like buttons. Light, dark, mobile. `user-light.png`. Fix: `<Badge>` tonal style.
3. The read only fields are hand built bordered boxes (`user/+page.svelte:1167,1200,1234`) rather than `<Input readonly>`. Code, light. Fix: shared Input or a definition list in AdminCard.
4. Mobile: the 4th tab "User Management" is cut off with no scroll hint. Mobile. `user-mobile.png`. Fix: scrollable tab bar with fade or wrap.

### /config

1. "MIGRATION" tile differs from the other 16: larger title font, greyed text, subtitle line and a "Free" badge. Light, dark. `config-light.png`. Fix: same tile component, badge only.
2. Tile icons mix monochrome black (System Settings, System Monitor, Collection Builder, Content Sync, Email Previews) with coloured icons. Light. `config-light.png`. Fix: one icon colour rule.
3. "Email Previews" tile leaves the admin layout entirely (see /email-previews). Light. Fix: see that route.

### /config/access-management

1. Native checkboxes in the permission matrix (G8), `permissions.svelte:418,485`. Light, dark. `config-access-management-light.png`.
2. Tabs have no fill in light but grey filled tabs in dark (G12). Dark. `config-access-management-dark.png`.
3. "Save (0)" is pale (disabled look) in light but a solid blue button in dark with the same zero changes; "Reset" is plain grey text. Light, dark. Fix: same disabled styling in both modes, `ghost` Button for Reset.
4. Red "DELETE" action badges on the dark wash are low contrast. Dark. `config-access-management-dark.png`. Fix: `error-400` text on `error-900/20` per contract.
5. Mobile: Editor and Guest columns are cut off at the right edge of the card; Website Tokens and SSO tabs are hidden. Mobile. `config-access-management-mobile.png`. Fix: visible horizontal scroll container (`overflow-x-auto`) with sticky first column.

### /config/api

1. Page header has no icon and shows the description as a subtitle, while sibling config pages show icon plus title (and no back button, G3). Light, dark. `config-api-light.png`. Fix: pass `icon` and `showBackButton` like the others.
2. "Descending (Newest firs" select text truncated at 1440 px. Light, dark. `config-api-light.png`. Fix: shorter label or wider column.
3. Inputs mix monospace ("status", "createdAt") and sans ("published", "20") in the same row. Light. `config-api-light.png`. Fix: one font per field type.
4. Execution response placeholder is italic grey on near black, very low contrast. Light. `config-api-light.png`. Fix: `--admin-text-muted` on the code surface.

### /config/automations

1. Dead surface classes on AdminCard, including `border-2 border-dashed` on the empty state which renders solid (G5), `config/automations/+page.svelte:246,324,339,366`. Code.
2. Empty state CTA "Get Started" green while sibling empty states use blue or nothing (G6). `config-automations-light.png`.

### /config/automations/new

1. Selected trigger tile "Event Hook" is white text on a very pale green wash in light mode, practically unreadable; unselected tiles are dark grey filled in light mode. `config/automations/[id]/+page.svelte:510-513` (`variant="surface"` plus `bg-primary-500/10`). Light. `config-automations-new-light.png`. Fix: selected state `border-primary-500 bg-primary-500/10` with `--admin-text-body` text; unselected `ghost`.
2. Every event name is printed twice: Checkbox label and a second span ("Entry Created [icon] Entry Created CONTENT"), `[id]/+page.svelte:544-552`. Light, dark. `config-automations-new-light.png`. Fix: drop the second span or the Checkbox `label` (keep `sr-only`).
3. Two column event list wraps unevenly (some rows one line, some two), creating ragged row heights. Light, dark. `config-automations-new-light.png`. Fix: fixed row layout, label left, badge right.
4. Content is a narrow centred card unlike the full width layout of the list page and other config pages; "Next" is blue, "Cancel" dark grey filled. Light. `config-automations-new-light.png`. Fix: full width AdminCard, `primary` Next, `ghost` Cancel.

### /config/design-system (all tabs)

1. Literal markup in labels: "Apply instantly — no reload needed</span", "DEFAULT</BADGE", "Import Theme</Button", "Export Theme JSON</Button", "Reset to Defaults</Button", "marketplace.sveltycms.com</a" (G4). Light, dark. `config-design-system-overrides-light.png`, `-themes-dark.png`, `-presets-light.png`, `-advanced-dark.png`.
2. Inverted tab style: inactive tabs are solid blue buttons, the active tab is pale (G12). Light, dark. `config-design-system-overrides-light.png`.
3. "Unsaved changes — live preview active" bar appears on first load with no change (F6), and the warning icon sits on its own line above the text. Light, dark. `config-design-system-layout-light.png`. Fix: icon inline with text.
4. Discard is plain text in light and a grey filled button in dark. Light, dark. `config-design-system-layout-light.png` vs `-advanced-dark.png`. Fix: one `ghost` Button.
5. Content width changes between tabs: Layout, Style, Features, Advanced, Themes are limited to about 670 px while Overrides, Preview, Presets use the full width. Light, dark. Fix: one width rule for all tabs.
6. Visual Style cards repeat the description in the title ("Flat No borders · No shadows" + "No borders · No shadows"), `en.json:2206`. Dark. `config-design-system-style-dark.png`.
7. Live Preview palette swatches for secondary-950, success-50/950, warning-50/950, error-950 render empty (white boxes) because the variables resolve to an empty value at runtime (F10). Light. `config-design-system-preview-light.png`.
8. Presets copy errors: "Open Live Preview — — preview..." (double dash), "Save Themeto persist", "sveltycms-palette-start/endmarkers" (missing spaces). Light. `config-design-system-presets-light.png`. Fix: copy.
9. "Create New Theme" placeholder duplicates text "Theme name (e.g. Midnight Blue, High Contrast) New theme name" (`en.json:2231`). Dark. `config-design-system-themes-dark.png`.

### /config/monitor

1. Merged strings: card title "QUICK LINKS JUMP TO COMMON ADMIN TOOLS" plus subtitle repeating it; "Security Incident Feed View All Logs" next to a "View All Logs" button (G4). Light, dark, mobile. `config-monitor-light.png`.
2. Audit trail table: first header cell contains all headers "TIMESTAMP USER / ACTOR EVENT TYPE RESULT DETAILS SHA-256 HASH" followed by the real headers (G4). Light. `config-monitor-light.png`.
3. Service Health first column header reads "SERVICE STATUS" (`monitor_service`, `en.json:1360`) for the service name column. Light. Fix: "Service".
4. Security stat card icon tile is a solid blue square with no glyph, while the other three cards show a tinted tile with an icon. Light, mobile. `config-monitor-light.png`. Fix: same tinted tile and icon.
5. Mobile: Service Health message column and the audit trail table are clipped at the right with no scroll container; some cards touch each other (no gap) while others have a large gap. Mobile. `config-monitor-mobile.png`. Fix: `overflow-x-auto` wrappers, one `gap` value.
6. Dead surface classes on 8 AdminCards (G5). Code.

### /config/queue

1. First header cell contains all column names "JOB ID TASK TYPE STATUS ATTEMPTS NEXT RUN CREATED ACTIONS" (`queue_job_id`, `en.json:1479`). Light, dark. `config-queue-light.png`.
2. "Recent Jobs" heading sits directly under the stat cards with no spacing and outside a card. Light. `config-queue-light.png`. Fix: put heading and table in one AdminCard.
3. Every row has a solid red delete button, 20 strong red blocks in one column. Light. `config-queue-light.png`. Fix: `ghost` error icon button.

### /config/redirects

1. First header cell "FROM PATH TO PATH TYPE STATUS ACTIONS" plus real headers (`en.json:1511`). Light, dark. `config-redirects-light.png`.
2. Search placeholder "Search by path... Search redirects by path" (`en.json:1518`). Light, dark. `config-redirects-light.png`.
3. Page action uses `tertiary` (G2). `config/redirects/+page.svelte:194`.

### /config/sync

1. Merge mode is a native `<select>` (browser default style), not `<Select>`. Light, dark. `config-sync-light.png`. Fix: `@components/ui` Select.
2. Header row "NAME TYPE" merged in the first column, then "TYPE" again. Light. `config-sync-light.png`.
3. Mode toolbar is a heavy mid grey block in light mode, unlike other card chrome. Light. `config-sync-light.png`. Fix: `--admin-bg-chip` or plain card.
4. Tabs are a green filled pill style used nowhere else (G12). Light. `config-sync-light.png`.

### /config/system-settings

1. The sticky footer bar covers the next section: the "Multi-Tenancy Migration" button is half hidden behind it. Light, dark. `config-system-settings-light.png`. Fix: bottom padding equal to the footer height.
2. Footer mixes a solid red "Reset to Defaults", ghost "Discard", two unlabeled dark grey icon buttons (export, import) and the Save button. Light, dark, mobile. Fix: `ghost` error Reset, labelled or tooltip icon buttons.
3. Group navigation icons: "License Keys" uses a different (black) icon and its label starts further left than the others. Light. `config-system-settings-light.png`. Fix: same icon style and indent.
4. "Repair Cache" is a solid warning button; on mobile it loses its text and shows only a wrench. Light, mobile. `config-system-settings-mobile.png`. Fix: keep the label, use `surface` or `tertiary`.
5. Save label is "Save" on desktop and "Saved" on mobile for the same state. Mobile. `config-system-settings-mobile.png`.

### /config/trash

1. Empty state has an icon but no CTA and a different text style from siblings (G6). Light, dark. `config-trash-light.png`.

### /config/webhooks

1. Page action uses `tertiary` (G2), `config/webhooks/+page.svelte:229`. Dark. `config-webhooks-dark.png`.
2. Empty state has no icon, unlike logs/trash/automations (G6). Dark. `config-webhooks-dark.png`.

### /config/webhooks/logs

1. A red error toast "Webhook logs not found" appears on a normal empty list (F7). Light, dark. `config-webhooks-logs-light.png`.
2. "All Statuses" select in the header is a smaller size (about 32 px high, 12 px text) than other header controls. Light. Fix: standard Select size.

### /config/workflows

1. Mobile: "+ Add State", "+ Add Transition", "Save Workflow" overlap the page title and hamburger; "Save Workflow" is clipped at the right. Mobile. `config-workflows-mobile.png`.
2. Mobile: the canvas has a fixed height and cuts off the third state ("Published"); a connector line runs out of the canvas. Mobile. `config-workflows-mobile.png`.
3. Header buttons use a literal "+" character and dark grey filled `surface` style; page has no back button (G3). Light, dark. `config-workflows-light.png`. Fix: icon prop, `tertiary` for supporting actions.
4. "Target Collection" select is smaller (about 32 px, 12 px text) than the "Workflow Name" and "Description" inputs next to it (40 px, 14 px). Light, dark. `config-workflows-light.png`. Fix: same control size.
5. State colours are hard coded hex (`#94a3b8`, `#fbbf24`, `#22c55e`, `#3b82f6`) at `config/workflows/+page.svelte:73-75,245`, outside the theme tokens. Code. Fix: map to `surface`, `warning`, `success`, `tertiary` tokens.
6. Transition arrows between Draft and In Review are drawn as a squiggle overlapping the card edges. Light, dark. `config-workflows-light.png`. Fix: anchor connectors to card edges.

### /admin

1. Page is blank in light and dark: the AdminPageShell is `absolute inset-0` but the `(admin)` group has no layout, so its parent has height 0 and the shell is 0 px tall (measured). See F1. `admin-light.png`.

### /admin/tenants

1. Same blank page as /admin (F1). `admin-tenants-light.png`, `admin-tenants-dark.png`. Also carries the dead AdminCard surface classes (G5).

### /email-previews

1. No admin shell, no sidebar, its own header (`<h1>` in `src/routes/email-previews/+page.svelte`), reached from a config tile, no way back except the browser. Light, dark. `email-previews-light.png`. Fix: move under `(app)` with AdminPageShell and AdminCard.
2. Developer text in the UI: "Templates from src/components/emails. Rendered on the server; the preview runs in a sandboxed iframe." Light, dark. Fix: user facing description.
3. Template list shows raw file names ("custom-email", "order-refunded"). Light, dark. Fix: readable labels.
4. Logo image broken in the preview (F8); preview frame stays white in dark mode (acceptable for email, but the surrounding card is `bg-white`, `+page.svelte:82`). Dark. `email-previews-dark.png`.

### /share/[token] (invalid token)

1. Uses a separate always dark error design (rotated red URL banner, red pill buttons) that matches no other page. Light, dark. `share-invalid-token-light.png`. (Always dark error pages are an allowed exception in the style guide.)
2. The rotated red banner covers the "404" digits, and the URL in it is truncated. Light, dark. Fix: place the banner below the number.

### (app) +error (404 inside the admin)

1. "Page Not Found" appears three times (page title "404 — Page Not Found", heading, subheading). Light, dark. `error-404-app-light.png`. Fix: show the message once.
2. The error icon in the title follows G1 and turns green in dark mode. Dark. `error-404-root-dark.png`. Fix: `error-500`/`warning-500` icon in both modes.
3. Three identical outline buttons, no primary action. Light, dark. Fix: "Go to Dashboard" `variant="primary"`.

---

## 5. Functional bugs found (report only, not ours to fix)

F1. `/admin` and `/admin/tenants` render blank in every mode. DOM contains the content, but the shell height is 0 (chain: `.admin-theme-container absolute inset-0` h=0, parent `div.relative z-0` h=0, `body` h=0). There is no `src/routes/(admin)/+layout.svelte`. Screenshots `admin-light.png`, `admin-tenants-dark.png`.

F2. Setup database error message contains raw HTML that is printed as text (`src/routes/setup/error-classifier.ts:119`, rendered via `{@html}` at `connection-status.svelte:240` but shown escaped). `setup-step1-db-exists-error-light.png`.

F3. `/setup` throws `effect_orphan` (`$effect` called outside component init) from `src/stores/setup-store.svelte.ts:338`, called from `src/routes/setup/+page.svelte:120`. Seen as a page error on every setup load and in `dev.log`.

F4. The `(app)` layout POSTs `/api/theme/admin-theme` on every admin page load (verified: one POST on loading `/dashboard`; `dev.log` shows "Admin theme saved" after each navigation). The debounced `$effect` at `src/routes/(app)/+layout.svelte:300-330` runs on mount without a change check.

F5. Design System shows "Unsaved changes — live preview active" with Discard/Save on a fresh load with no edits (all admin tabs).

F6. `/config/webhooks/logs` calls `GET /api/webhooks/logs`, gets 404 "Webhook logs not found", and shows a red error toast on an empty but valid state.

F7. `/email-previews`: logo image blocked (`net::ERR_BLOCKED_BY_RESPONSE.NotSameOriginAfterDefaultedToSameOriginByCoep`) and `Blocked script execution in 'about:srcdoc'` (sandboxed iframe) on every load.

F8. Theme palette variables empty at runtime: `getComputedStyle(:root)` returns empty for `--color-secondary-950`, `--color-success-50`, `--color-success-950`, `--color-warning-50`, `--color-warning-950`, `--color-error-950`, although they are declared in `src/app.css:116,132,142,145,168` (other shades such as `--color-success-500` resolve). Visible as blank swatches in Live Preview.

F9. Monitor uptime is not stable: "16d 23h 30m" in the 1440 run and "18d 19h 41m" in the mobile run about 25 minutes later; the dev server was started the same evening.

F10. Setup review says "Once finished, you'll be redirected to the login page"; after Complete the browser lands on `/en/collection/posts` already logged in. The default "Production URL" is prefilled as `https://localhost:5173` while the dev server runs on http.

F11. Access Management lists "Admin Access" under both SYSTEM and ADMIN, and "Settings Management" under both SYSTEM and CONFIGURATION.

F12. Config Sync reports "0 new, 40 updated, 0 deleted" immediately after a fresh setup with no changes made.

F13. Collection create view: the page `<h1>` is "Edit posts Entry" while the visible header says "CREATE Posts". Console warning `[tiptap warn]: Duplicate extension names found: ['textStyle']`.

F14. Every page logs `Error with Permissions-Policy header: Unrecognized feature: 'web-share'`.

F15. `/plugin` redirects to `/config`; its own page never shows.

F16. System Settings "Cache & Performance" shows 1 second for all eight cache TTL fields on a fresh install (confirm whether intended).

F17. Version badge in the sidebar changed from green to orange during the session (green on early pages, orange from `/config/redirects` onward) without any user action.

F18. Cookie consent POST returns 503 during setup (`Setup incomplete`) and 403 for blocked user agents (seen in `dev.log`); the banner closes either way. Whether the choice is stored was not checked.

F19. Setup step 1 "Test Database Connection" creates `config/private.ts` and the SQLite file; restarting the wizard then requires "Yes, Overwrite It" (observed when re-running the wizard).

---

## 6. Out of scope for now (light audit only)

Marketplace (`/config/extensions`, marketplace tab) and the Collection Builder are excluded from the fix scope per client instruction. Seen in one light capture each:

- `/config/extensions` (`config-extensions-light.png`): Commerce card description is clipped mid line ("stays a payment adapter" cut); "ACTIVE" badges use tertiary blue while "DISABLED" is grey; Google PageSpeed card shows an orange warning dot touching the badge; Marketplace tab count "52" with "IN-APP" chip in a separate button style.
- `/config/collectionbuilder` (`config-collectionbuilder-OUTOFSCOPE-light.png`): the four header buttons each use a different status colour (warning "Quick Start", success "Introspect / Ingest", tertiary "Add Category", error red "Add Collection" for a create action); "COLLECTION" badges are solid red; Save is pale green in the page header. Builder edit routes were not opened.
- SEO widget inside the collection form shows a "Premium Trial Active ... Get License" banner (licensing related, `collection-posts-create-dark.png`).
