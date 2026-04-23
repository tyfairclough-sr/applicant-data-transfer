# Lessons Learned — SPL Prototyping

Collected from building prototypes in this workspace. Updated as new patterns are discovered.

---

## Form Components

### spl-checkbox / spl-radio label attribute (applicant-data-transfer, 2026-04-23)
- Both `spl-checkbox` and `spl-radio` render their visible label from the **`label` attribute** — slotted text inside the tag is silently ignored.
- For checkboxes: `<spl-checkbox label="Allow users to change..." checked="true"></spl-checkbox>`.
- For radios with custom inline content (e.g. an info icon next to the text), use `ariaLabel` on the radio and a separate adjacent `<label for="radio-id">…</label>` element. Keep an underlying click target with `for=` matching the radio's `id` for accessibility.

### Manual radio group exclusivity (applicant-data-transfer, 2026-04-23)
- Multiple `spl-radio` elements with the same `name` attribute do **not** automatically deselect siblings on click — they are independent web components.
- Wire a `spl-change` listener on the radio group (`document.querySelectorAll('spl-radio[name="..."]')`) and set `other.checked = false` on siblings to enforce single-select behaviour in prototypes.

### SPL form components emit `spl-change`, not native `change` (applicant-data-transfer, 2026-04-23)
- `spl-checkbox` and `spl-radio` dispatch a custom `spl-change` event, NOT the native `change` event. Listening on `change` will silently never fire.
- Always use: `el.addEventListener('spl-change', handler)`.

### Native `[hidden]` is overridden by component CSS `display` rules (applicant-data-transfer, 2026-04-23)
- A class rule like `.slideout { display: flex }` overrides the user-agent default `[hidden] { display: none }` because both selectors have the same specificity and the class rule comes later in the cascade.
- Add a top-of-stylesheet rule `[hidden] { display: none !important; }` (or scope to the relevant components) so the `hidden` attribute reliably hides elements that also have explicit `display: flex/grid/block` styles. Symptom: a panel that should start hidden appears on first paint.

### spl-checkbox state is `.value`, not `.checked` (applicant-data-transfer, 2026-04-23)
- `spl-checkbox` defines a setter `set checked(e) { this.value = e }` but **no `checked` getter** — reading `.checked` returns `undefined`.
- To read the current state, use `cb.value === true` (or `Boolean(cb.value)`).
- Setting `cb.checked = true/false` still works (it forwards to `.value`).
- `spl-radio`, by contrast, has both getter and setter for `.checked` and behaves normally.

## Layout

### Settings/form page above page header (applicant-data-transfer, 2026-04-23)
- For settings pages with a back-arrow + title above a white card, render the page header **outside** `prototype-content` (still inside `.prototype-wrapper`). Use `<spl-back-navigation title="..." tooltipContent="...">`; it produces the arrow + vertical divider + title in one component.
- Keep `prototype-content` (default white card) for the form body. The card sits below the page header and uses its built-in `padding: 32px`.

## Components

### spl-back-navigation back-click event (applicant-data-transfer, 2026-04-23)
- `spl-back-navigation` dispatches a `back-click` custom event when the arrow is clicked. Listen on the element itself: `el.addEventListener('back-click', ...)` — there is no `href` prop.

## Topbar navigation

### Override navItems href to navigate between prototypes (applicant-list, 2026-04-23)
- The default `sr-topbar` `navItems` use hash hrefs (`#/home`, `#/jobs`, …) which don't navigate between prototypes.
- To wire a top-nav link to another prototype folder, override the entire `navItems` array in `window.__SPL_TOPBAR_CONFIG` and set the `href` to the prototype path, e.g. `{ id: 'jobs', label: 'Jobs', href: '/applicant-list/' }`.
- To make no tab look active on a given page, set `currentPage: ''` (the topbar only highlights when `currentPage === item.id`).

## Save / dirty-state pattern

### Page-level Save button — disabled until dirty (applicant-data-transfer, 2026-04-23)
- Start the primary Save button as `disabled="true"` in HTML so the user can't submit a no-op on first paint.
- Listen for `spl-change` on a parent container (e.g. the main settings card) — the event bubbles from `spl-checkbox`/`spl-radio`/etc. and is a single hook for "any control changed → mark dirty".
- On Save click, toggle the disabled attribute back on (`saveBtn.setAttribute('disabled','true')`). On any further `spl-change` event, remove it again. This produces the expected "no pending changes" affordance.
- For sibling panels that commit their own state (e.g. a slideout's own Save), expose a small global hook like `window.__markSettingsDirty = () => …` and call it after a successful commit so the page-level Save re-enables.

## Dense list / table pages

### Applicant list layout (applicant-list, 2026-04-23)
- For Jobs-style list pages, use `prototype-content--flush` (no white-card chrome) and stack two custom `.surface` sections: the job header card + the applicants card. Gives full control over inner layout (counters strip, sub-tabs, search, filter pills, table) without fighting `spl-card` paddings.
- Lay the applicant table with `display: grid; grid-template-columns: …` per row instead of `<table>`; this lets each cell stack a primary line + secondary muted line cleanly and handles avatar + name + role + badges in one cell.
- Sub-tab "active" indicator: a `::after` 3px-tall green bar pinned to the bottom of the active tab, with a 1px border-bottom on the tab strip itself. Matches SmartRecruiters style without bringing in `spl-tabs` (which wants its own data model).

## Cross-prototype state

### Sharing admin/settings between prototypes via localStorage (2026-04-23)
- Static prototypes live in separate folders under the same Vite origin, so `localStorage` is shared across pages (e.g. `/applicant-data-transfer/` ↔ `/applicant-list/`).
- Pattern: on the settings page Save handler, write a small JSON blob (`localStorage.setItem('adtAdminSettings', JSON.stringify({...}))`); on the consuming page, read it on dialog open with sensible defaults so the UX still works on a fresh visit.
- For `spl-checkbox` state use `el.value === true` (not `.checked`); for `spl-radio` use `el.checked === true`. Keep the persisted shape minimal (just the booleans/enums you need).

### Dialog sections that adapt to admin settings (applicant-list ADT dialog, 2026-04-23)
- Pre-render every variant of a conditional section as siblings inside the dialog, each with its own id and the `hidden` attribute. On open, hide them all then `el.hidden = false` on the one that matches the current admin state. Keeps the markup declarative and avoids string-templating.
- Always include `[hidden] { display: none !important; }` in the page styles — many flex/grid containers override the UA `display: none` for `[hidden]` and a hidden variant will leak into layout.

### Inline search-as-you-type dropdown (no spl-autocomplete needed)
- Wrap a plain `<input>` + suffix icon in a `position: relative` container, then absolute-position a `.results` panel with `top: calc(100% + 4px); left/right: 0`. Listen for `input` on the field, render `<button class="result">` rows with `data-*` attributes, and on click write the selected title back into the input.
- Close the dropdown on outside-click by listening on `document` and checking `!wrapper.contains(e.target)`.

### Mirroring nested settings into a confirmation step (2026-04-23)
- When a settings page has nested groups (e.g. "Screening questions" → Default / Job specific / DEI), persist the parent toggle and the sub-state separately (e.g. `dataItems['data-screening']` + `screening: { default, jobSpecific, dei }`).
- In the consuming dialog, only render a sub-row when the parent toggle is on, then map sub-keys to user-facing labels ("Preconfigured screening questions responses", "Job specific screening questions"). Some sub-options can be intentionally hidden in the destination view (e.g. DEI) — keep that mapping in one place.
- Restore on load: the settings page should also `restoreAdminSettings()` from localStorage on page load so the user sees their last-saved state when navigating back.

## Topbar avatar menu

### Custom dropdown attached to sr-topbar avatar (main.js, 2026-04-23)
- `sr-topbar` renders the avatar as `.sr-topbar-avatar` but doesn't ship a dropdown. Add a single `setupAvatarMenu()` in `main.js` that runs after `injectSRTopbar` so every prototype gets the same menu for free.
- Build the menu as a `position: fixed` div appended to `<body>` (not nested inside the topbar) to avoid clipping/overflow issues. Position it on open by reading `avatar.getBoundingClientRect()` and pinning `top` and `right`.
- Use inline SVGs for menu icons (cog, account-plus, etc.) instead of registering one-off icons in `@sr/spl-icons` — keeps the patch surface small and ensures icons paint immediately without the `__splPatchIcons` round-trip.
- Wire "Settings" to the actual settings prototype (`href="/applicant-data-transfer/"`); other items can stay as placeholder `#` links.

## Responsive layout — keeping the topbar avatar reachable (2026-04-23)

### Symptom
On narrow viewports (≈700–1080px) the avatar in the top-right became unreachable: clicking right of the bell pushed nothing because the body had grown wider than the viewport. The user has to horizontal-scroll the whole page to reach the avatar dropdown — a poor experience.

### Root causes
1. **Wide grid table** (`.applicants-table` with `grid-template-columns` summing to ~1080px) inflates its parent, which inflates the page and forces body horizontal scroll. Sticky topbar moves left/right with body scroll, so the avatar disappears off the viewport.
2. **Sub-tabs nav** with 8 non-wrapping items (~700px) does the same on really narrow widths.
3. **`sr-topbar`** flex children don't shrink past their natural sizes — the nav (8 links + brand) plus search + buttons + avatar overflow when viewport < ~1080px.

### Fixes
- Wrap the wide table in a dedicated scroller: `<div class="applicants-table-scroll" style="overflow-x: auto">` and give the table itself an explicit `min-width` (e.g. `1080px`) so it scrolls inside its own card without expanding the page.
- Make the sub-tabs scroll horizontally too: `overflow-x: auto` on `.subtabs`, plus `white-space: nowrap` and `flex-shrink: 0` on each tab button.
- Tighten the topbar at narrower breakpoints in `shared/sr-topbar.js`:
    - 1200px → reduce nav gap and shrink search input.
    - 1080px → hide the main nav entirely (the avatar dropdown carries the navigation slack), tighten right-side gap, reduce nav-link padding.
    - 900px → search bar reduces.
    - 640px → search bar reduces further.
    - 520px → hide the search entirely; keep brand + buttons + avatar.

### Verification
At 1400 / 1024 / 900 / 700 / 500 px viewports, `body.scrollWidth === window.innerWidth` and the `.sr-topbar-avatar` rect is fully inside the viewport. Internal table/sub-tabs scrolling preserves all data without ever pushing the avatar off-screen.
