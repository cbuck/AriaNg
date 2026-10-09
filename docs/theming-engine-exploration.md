# Modular theming for AriaNg

## Recommendation

Add an opt-in theme module with a token-based CSS adapter. Keep the current
light/dark/system preference, introduce a separate palette ID, and preserve the
original appearance when the palette is `default`. Avoid rewriting existing
styles or changing download and RPC logic.

This is an architecture proposal based on the current checkout, not an
implemented or browser-verified engine.

## Existing integration points

- AngularJS 1.6, Bootstrap 3.4, and AdminLTE 2.4 provide the UI foundation.
- `src/styles/theme/default.css` and `default-dark.css` contain literal colors
  and selectors scoped to `.skin-aria-ng` and `.theme-dark.skin-aria-ng`.
- `src/scripts/core/root.js` selects appearance on startup, settings changes,
  and system color-scheme changes. It exposes `currentTheme` to chart templates.
- `src/scripts/services/ariaNgSettingService.js` persists `theme`. Its import
  routine accepts primitive keys only when they exist in default options.
- `src/views/settings-ariang.html` hardcodes the three appearance choices.
- `src/scripts/directives/chart.js` supplies light/dark ECharts themes at chart
  creation. It does not watch appearance changes to recreate an existing chart.
- `src/scripts/directives/pieceBar.js` paints a canvas. The task-detail template
  passes a literal blue color. `pieceMap.js` creates DOM elements and is CSS
  themeable.
- `src/index.html` explicitly lists styles and scripts for Gulp bundling.
  The project supports both a normal distribution and a single-file bundle.

## Module boundary

Suggested new files:

```text
src/scripts/themes/
  registry.js                 # Angular provider; registers built-in packs
  engine.js                   # resolves palette + mode, applies tokens
  palettes/ocean.js           # declarative light/dark token values
src/styles/theme/adapter.css  # existing selectors mapped to semantic tokens
```

Keep the adapter as the only module that knows Bootstrap/AdminLTE/AriaNg
selectors. Theme authors change values, not application markup. Use an Angular
provider for registration during configuration; reject duplicate IDs and
unsupported schema versions. An ordinary Angular service owns runtime state.

Conceptual pack contract:

```js
{
    schemaVersion: 1,
    id: 'ocean',
    label: 'Ocean',
    light: {
        surface: '#ffffff',
        surfaceMuted: '#f3f7fb',
        text: '#183049',
        border: '#c7d5e2',
        accent: '#0067b8'
    },
    dark: {
        surface: '#172330',
        surfaceMuted: '#101923',
        text: '#e4edf5',
        border: '#40566d',
        accent: '#66b8ff'
    }
}
```

These are illustrative values, not a complete or contrast-validated palette.
The final contract also needs control, hover, focus, selection, disabled,
sidebar, tooltip, status, chart, and piece-progress tokens. Use semantic names;
do not collapse all blue values into one token because their roles differ.

## Runtime and compatibility

1. Keep `theme: light | dark | system` unchanged. Add `themePalette: default`
   to default options and expose a small getter/setter in the settings service.
   This integrates with the existing primitive import/export behavior.
2. Resolve the effective mode in one engine. Maintain `theme-dark` and
   `currentTheme` for existing consumers. Replace the root's duplicated theme
   functions/listener with calls into that engine.
3. For a custom palette, set a validated `data-aria-ng-palette` attribute and
   engine-owned `--aria-ng-*` properties on `body`. Keeping them on `body`
   covers tooltips, dialogs, and popovers inserted outside the main wrapper.
4. Load adapter CSS after both original theme styles. Scope every adapter rule
   to the palette attribute, and match legacy selector specificity explicitly.
   Account for existing `!important` rules individually.
5. On return to `default`, remove the attribute and every engine-owned token.
   The original styles then determine the appearance. Unknown palette IDs,
   invalid packs, and missing CSS-variable support resolve to this default.
6. Emit one resolved-theme-change event after applying tokens, including a
   palette revision. Consumers such as charts and canvas piece bars can redraw
   even when the mode stays the same and only the palette changes.

Provide complete fallback values before activating a custom palette. Feature
detect custom properties; the current browser policy needs to be checked before
making them mandatory. Palette changes should not leave stale values from the
previous pack. Registration must be complete before the initial resolution.

## Expected original-file changes

| File | Purpose |
| --- | --- |
| `src/index.html` | Include registry, engine, packs, and adapter in existing build blocks |
| `src/scripts/core/root.js` | Delegate theme startup, selection, and system changes |
| `src/scripts/config/constants.js` | Add default palette ID |
| `src/scripts/services/ariaNgSettingService.js` | Persist palette ID |
| `src/scripts/controllers/settings-ariang.js` | Expose registry and palette selection |
| `src/views/settings-ariang.html` | Add palette selector beside appearance mode |
| `src/scripts/config/defaultLanguage.js`, language resources | Add translated UI labels |
| `src/scripts/directives/chart.js` | Map chart tokens and refresh live charts |
| `src/scripts/directives/pieceBar.js` | Resolve progress token and redraw on theme change |

The last two are required for full visual coverage, but can follow a clearly
limited CSS proof of concept. No changes should be needed to RPC services, task
controllers, route definitions, or the original two theme styles.

For a few built-in packs, list their registration scripts in the existing
HTML build block. This avoids a new runtime loader and should fit both existing
build modes; verify both builds. Truly drop-in external packs require a later
build discovery step or a runtime loading contract. Do not advertise that
capability in the first version.

## Scope choices

| Approach | Original-code footprint | Tradeoff |
| --- | --- | --- |
| CSS overlay only | Smallest | Cannot fully theme canvas or offer an integrated persistent selector |
| Registry + engine + adapter | Small, localized | Recommended; one adapter must track upstream selector changes |
| Replace original CSS with tokens | Broad | Cleaner styling foundation but more merge conflicts |

Avoid prototype monkey-patching, Angular service decorators that intercept
private behavior, and DOM-injected settings controls. They hide integration
costs and make upgrades harder to inspect.

## Staged prototype

1. Add one built-in palette and the scoped adapter. Cover shell, task table,
   forms, buttons, dialogs, and transient overlays, preserving existing layout.
2. Wire registry, persistence, and system mode. Retain the existing default
   appearance as the compatibility baseline.
3. Integrate charts and canvas piece bars; check theme changes while open.
4. Document the token contract and how to register another built-in pack.

Keep user-imported themes out of this first prototype. A later data-only import
format can validate token names, values, size, and schema version. Loading raw
CSS or JavaScript is a separate capability with a much larger contract.

## Validation before calling the prototype complete

- Build both normal and single-file distributions and check asset inclusion.
- Confirm unchanged default light/dark appearance and existing saved settings.
- Exercise reload, import/export, reset, unknown IDs, and incomplete packs.
- Switch palettes and modes repeatedly; test a system-mode change while open.
- Inspect download list, new download, settings, task details, dialogs,
  dropdowns, notifications, tooltips, and mobile navigation in both modes.
- Check readable text, visible focus, disabled controls, and status distinctions.
- Check live chart/piece-bar recoloring and cleanup of listeners on destruction.

The upstream issue search surfaced requests about persistent theme settings
([#598](https://github.com/mayswind/AriaNg/issues/598)) and setting themes through
the command API ([#657](https://github.com/mayswind/AriaNg/issues/657)). These are
useful context for keeping persistence and public theme selection coherent;
they do not establish an upstream theme-pack API.
