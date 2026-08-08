# Solvronix Desk — Bug Report & Fix Plan

**App:** Solvronix Desk
**Version installed:** 2.1.0 (branch: `main`)
**Framework:** Frappe v16 / ERPNext
**Site:** erp.local (WSL, production bench)
**Reported by:** Client (tested in browser DevTools, no console errors observed)

---

## Issue 1 — `Ctrl+M` Module Switcher shortcut does not work

### Description
The README documents a "Module Switcher" feature:
> Press `Ctrl+M` from anywhere to open a searchable workspace switcher. Type a module name and press Enter to jump to it instantly.

Pressing `Ctrl+M` does nothing. No error is thrown in the browser console (checked via DevTools → Console, cleared before testing).

### Steps to Reproduce
1. Log in to the desk (any page).
2. Press `Ctrl+M`.
3. **Expected:** a searchable workspace switcher opens.
4. **Actual:** nothing happens. No console errors.

### Likely Root Causes (to confirm in code)
- The keyboard listener for this shortcut may not be registered at all (dead/removed code, or gated behind a condition/feature flag that's off).
- The listener may be attached but the key-check logic is wrong (e.g. relying on deprecated `event.keyCode` instead of `event.key`, or an incorrect modifier check), causing a silent no-op — this would **not** throw a console error.
- The event could be intercepted earlier (browser/extension) before reaching the app's handler — should be ruled out by testing in an Incognito window with extensions disabled, and in a second browser.

### Fix Steps for Developer
1. Search the codebase for the shortcut binding: grep for `Ctrl+M`, `ctrlKey`, `keyCode`, `77`, `module switcher`, `module_switcher`. Likely candidates based on the file names referenced in the project's own v1.1.0 changelog: `solvronix_desk.js`, `command_palette.js`.
2. Confirm whether the event listener is actually being attached on page load (temporary `console.log` at the top of the handler, or a breakpoint).
3. Verify the modifier/key check is correct: should check `event.ctrlKey` (and ideally `event.metaKey` for macOS) together with `event.key.toLowerCase() === 'm'`, and call `event.preventDefault()` early enough to stop any default browser behavior.
4. Rule out the browser eating the shortcut first — test in Incognito mode with all extensions disabled, and in a second browser, before concluding it's a code bug.
5. Once fixed, rebuild assets (`bench build --app solvronix_desk`), clear cache, hard-refresh, and confirm the shortcut opens the switcher from multiple different pages (not just the home screen).

---

## Issue 2 — Module/workspace names not translated to Arabic (All Apps page & All Options panel)

### Description
With the desk language set to Arabic, most of the UI is correctly translated and RTL-aware. However, module/workspace names remain in English in two places:
- The **All Apps** page (`/app/all-apps`) — e.g. "Buying", "Tenure", "POS Awesome", "HR Setup", "Financial Reports", "Leaves", "Shift & Attendance", "Recruitment", "Performance", "Assets", "Stock", "Expenses", "Selling", "Tax & Benefits", "Quality", "Payroll", "Subcontracting", "Manufacturing", "Build", "Support", "Website", "ERPNext Settings", "Users", "CRM".
- The **All Options** side panel (opened from the toolbar search/menu icon) — same module list, same problem.

### Context
The project's own v1.1.0 release notes state that i18n fixes ("custom UI strings not translating") were applied specifically to `notification_center.js` and `command_palette.js`. **Neither the All Apps page nor the All Options panel are mentioned in that fix**, which strongly suggests these two views use a separate code path that was not covered by the earlier i18n pass.

### Diagnostic Already Performed
A manual Arabic `Translation` record was **not yet conclusively tested** by the client (next action item below) — this single test will immediately tell us whether this is a **data** problem or a **code** problem:

> Create a `Translation` doc: Language = Arabic, Source Text = `Buying`, Translated Text = `المشتريات`. Reload the All Apps page with **no code changes**.
> - If "Buying" now shows translated → **data-only issue** (missing translation entries).
> - If it still shows "Buying" → **code issue** (the string is not passed through `__()` before being rendered).

### Fix Steps for Developer
1. Run the diagnostic test above first — it determines which of the two paths below is needed (possibly both).
2. **If code issue:** locate the file(s) responsible for rendering the All Apps grid and the All Options panel (likely under `solvronix_desk/public/js/`). Check whether module/workspace labels are inserted directly (e.g. `${module.label}` or `module.label`) instead of being wrapped in the translation function (`${__(module.label)}`). Wrap any unwrapped labels.
3. **If data issue (or in addition to the code fix):** add Arabic `Translation` records for every custom/standard module name currently shown untranslated in these two views (see full list above). This can be done via `/app/translation` one at a time, or in bulk — the site already has a `custom_words` app installed, which should provide a bulk translation management interface; use that if available instead of adding records one by one.
4. After code changes: `bench build --app solvronix_desk`, `bench --site erp.local clear-cache`, then hard-refresh (`Ctrl+Shift+R`) and re-test both the All Apps page and the All Options panel with Arabic selected.
5. Confirm no regression: switch back to English and verify all module names still render correctly in English.

---

## Notes for Developer
- Both issues should be fixed on a branch other than `main` if possible, so they can be reviewed/merged cleanly and optionally submitted upstream (repo is MIT-licensed).
- No console errors were observed for Issue 1, which should narrow investigation time — the bug is very likely a silent logic error, not a crash.
- Issue 2's root cause is almost certainly connected to the fact that the earlier i18n fix (v1.1.0) explicitly did not cover the file(s) behind All Apps / All Options.
