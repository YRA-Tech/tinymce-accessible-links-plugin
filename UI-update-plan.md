# UI Update Plan: Accessible Links Plugin

## Overview

Modernize the Link Accessibility Options dialog to improve usability, accessibility, and code quality. The current dialog uses checkboxes with manual-uncheck `onChange` hacks to simulate radio button behavior. This plan replaces that pattern with proper ARIA radio groups via `htmlpanel`, adds a live SVG preview for the custom symbol option, and updates the plugin for TinyMCE 8 compatibility.

## Current Problems

### 1. Checkboxes simulating radio buttons
The Symbol section has 7 checkboxes (`noSymbol`, `downArrow`, `topPage`, `neArrow`, `rightArrow`, `overlappingSquares`, `customSvgSymbol`) that are mutually exclusive. The `onChange` handler manually unchecks the others when one is selected. Same pattern for the Screen-reader text section (4 mutually exclusive checkboxes + 1 independent `srTextExternal`).

**Why this is bad:**
- Checkboxes signal multi-select to the user — confusing when only one can be active
- The `onChange` hack is fragile and hard to follow
- Not semantically accessible — screen readers announce "checkbox" not "radio button"

### 2. No preview of custom SVG
The user pastes SVG markup into a text input but has no way to see what it looks like until they click Update.

### 3. Hardcoded icon URL
The toolbar icon references `wittjeff.github.io` which no longer resolves after the repo moved to the YRA-Tech org. The icon path should be relative, using the `url` parameter passed to the plugin setup function.

### 4. Stale `tinymce.init()` call in Plugin.ts
The plugin source file includes a `tinymce.init()` call at the bottom of the exported default function. This is a leftover from development — the consuming application handles initialization, not the plugin.

## Proposed Changes

### Change 1: Replace checkboxes with ARIA radio groups

Use `htmlpanel` to render proper radio groups with `role="radiogroup"` and native `<input type="radio">` elements.

**Symbol section — before:**
```typescript
{ type: 'checkbox', name: 'noSymbol', label: 'No symbol' },
{ type: 'checkbox', name: 'downArrow', label: '↓' },
// ... 5 more checkboxes
// + 20 lines of onChange hack to uncheck others
```

**Symbol section — after:**
```typescript
{
  type: 'htmlpanel',
  html: `
    <fieldset>
      <legend>Symbol</legend>
      <div role="radiogroup" aria-label="Symbol">
        <label><input type="radio" name="symbol" value="none" checked> No symbol</label>
        <label><input type="radio" name="symbol" value="downArrow"> ↓ Down arrow</label>
        <label><input type="radio" name="symbol" value="topPage"> ↥ Top of page</label>
        <label><input type="radio" name="symbol" value="neArrow"> ↗ External link</label>
        <label><input type="radio" name="symbol" value="rightArrow"> → Right arrow</label>
        <label><input type="radio" name="symbol" value="overlappingSquares"> 🗗 New window</label>
        <label><input type="radio" name="symbol" value="customSvg"> Custom SVG symbol</label>
      </div>
    </fieldset>
  `
}
```

**Screen-reader text section — after:**
```typescript
{
  type: 'htmlpanel',
  html: `
    <fieldset>
      <legend>Screen-reader text</legend>
      <div>
        <label><input type="checkbox" name="srTextExternal" value="external"> external site</label>
      </div>
      <div role="radiogroup" aria-label="Screen-reader text type">
        <label><input type="radio" name="srText" value="none" checked> None</label>
        <label><input type="radio" name="srText" value="newTab"> opens in a new tab</label>
        <label><input type="radio" name="srText" value="scrollDown"> scrolls down this page</label>
        <label><input type="radio" name="srText" value="topPage"> returns to top of page</label>
      </div>
    </fieldset>
  `
}
```

Note: `srTextExternal` remains a checkbox because it is independent — it can be combined with any of the radio options.

**Reading values on Update/Submit:**
Since `htmlpanel` doesn't participate in `getData()`, read selected values from the DOM:

```typescript
function getSelectedSymbol(): string {
  const selected = document.querySelector('input[name="symbol"]:checked') as HTMLInputElement;
  return selected ? selected.value : 'none';
}

function getSelectedSrText(): { external: boolean; type: string } {
  const external = (document.querySelector('input[name="srTextExternal"]') as HTMLInputElement)?.checked ?? false;
  const selected = document.querySelector('input[name="srText"]:checked') as HTMLInputElement;
  return { external, type: selected ? selected.value : 'none' };
}
```

**Impact on `onChange`:**
The entire mutual-exclusion `onChange` hack (lines ~140-165 of Plugin.ts) can be deleted. Native radio buttons handle exclusivity automatically.

### Change 2: Live custom SVG preview

Add a preview container that renders the SVG in real-time when "Custom SVG symbol" is selected.

```typescript
// Add to the Symbol fieldset htmlpanel:
<div id="svg-preview-container" style="margin-top: 8px; display: none;">
  <label>Preview:</label>
  <div id="svg-preview"
       style="display: inline-block; width: 2em; height: 2em;
              border: 1px solid #ccc; border-radius: 4px; padding: 4px;
              vertical-align: middle;">
  </div>
</div>
```

**Behavior:**
- When the `customSvg` radio is selected, show `#svg-preview-container` and render the SVG input's contents
- When a different symbol radio is selected, hide the preview
- Update the preview on every input event in the Custom SVG textarea
- Sanitize before rendering: strip `<script>` tags and `on*` event handlers to prevent XSS

**Implementation — attach listeners after dialog open:**

```typescript
// After editor.windowManager.open():
setTimeout(() => {
  const radioButtons = document.querySelectorAll('input[name="symbol"]');
  const svgInput = document.querySelector('input[name="customSvg"], textarea[name="customSvg"]');
  const previewContainer = document.getElementById('svg-preview-container');
  const preview = document.getElementById('svg-preview');

  function sanitizeSvg(raw: string): string {
    // Strip script tags and event handler attributes to prevent XSS
    return raw
      .replace(/<script[\s\S]*?<\/script>/gi, '')
      .replace(/\bon\w+\s*=/gi, '');
  }

  function updatePreview() {
    const isCustom = (document.querySelector('input[name="symbol"]:checked') as HTMLInputElement)?.value === 'customSvg';
    if (previewContainer) previewContainer.style.display = isCustom ? 'block' : 'none';
    if (isCustom && preview && svgInput) {
      const sanitized = sanitizeSvg((svgInput as HTMLInputElement).value);
      // Use DOMParser to safely parse SVG, then append the node
      const parser = new DOMParser();
      const doc = parser.parseFromString(sanitized, 'image/svg+xml');
      const svgElement = doc.documentElement;
      preview.replaceChildren();
      if (svgElement.tagName === 'svg') {
        preview.appendChild(document.importNode(svgElement, true));
      }
    }
  }

  radioButtons.forEach(radio => radio.addEventListener('change', updatePreview));
  if (svgInput) svgInput.addEventListener('input', updatePreview);
}, 0);
```

The `setTimeout(..., 0)` ensures the DOM is rendered before attaching listeners.

**Important:** The Custom SVG text input should remain a TinyMCE `input` component (not part of the `htmlpanel`) so it integrates with `getData()`. Keep it as:

```typescript
{
  type: 'input',
  name: 'customSvg',
  label: 'Custom SVG',
}
```

### Change 3: Fix icon path

**Before (Plugin.ts line 7):**
```typescript
editor.ui.registry.addIcon('custom-links-icon',
  '<img src="/icons/box-with-ne-arrow.svg" style="height: 24px; width: 24px; margin-top: 3px;"/>');
```

The built `.min.js` resolves this to the GitHub Pages URL via the build process. After the org transfer, the GitHub Pages URL is dead.

**After:**
```typescript
editor.ui.registry.addIcon('custom-links-icon',
  `<img src="${url}/icons/box-with-ne-arrow.svg" style="height: 24px; width: 24px; margin-top: 3px;"/>`);
```

The `url` parameter is the second argument to the plugin setup function — TinyMCE passes the base URL from `external_plugins` automatically. This makes the icon path relative to wherever the plugin is deployed.

**Requires:** The `icons/` directory must be co-located with the plugin JS file. When used via `external_plugins: { 'a11y-links': '/tinymce-plugins/a11y-links.js' }`, TinyMCE will pass `url = '/tinymce-plugins'`, so the icon resolves to `/tinymce-plugins/icons/box-with-ne-arrow.svg`.

### Change 4: Remove stale `tinymce.init()` from Plugin.ts

**Delete these lines at the bottom of Plugin.ts:**
```typescript
tinymce.init({
  selector: 'textarea.tinymce',
  plugins: 'a11y-links',
  toolbar: 'a11y-links',
  content_style: "tox-editor-container {background-color: red; }",
});
```

This is a dev leftover. The plugin should only register itself via `PluginManager.add()`. The consuming application controls `tinymce.init()`.

### Change 5: Handle `redial()` for Previous/Next navigation

When the user clicks Previous/Next, `redial()` rebuilds the entire dialog DOM. This means:
- Radio button selections reset to defaults
- Event listeners attached in Change 2 are destroyed
- The SVG preview disappears

**Solution:** After every `redial()` call, re-attach listeners:

```typescript
function attachRadioListeners() {
  // Same listener setup as Change 2
}

// In onAction, after redial:
setTimeout(() => {
  attachRadioListeners();
}, 0);
```

Note: For Previous/Next navigation, the dialog intentionally resets to defaults for each new link (current behavior). Only re-attach listeners, don't restore prior selections.

## Files to Modify

| File | Changes |
|------|---------|
| `src/main/ts/Plugin.ts` | All 5 changes above |
| `icons/box-with-ne-arrow.svg` | No change (already exists, just needs to ship with plugin) |

## Migration Notes

- No changes to the plugin's public API — it still registers as `a11y-links` with one toolbar button
- No changes to `getSymbolHtml()` or `getSrTextHtml()` return values — the HTML injected into editor content is unchanged
- The dialog's data model changes from multiple boolean fields to string-valued selections, but this is internal to the dialog and doesn't affect editor output
- Consumers using `external_plugins` get the icon fix for free — no config changes needed

## Testing Checklist

- [ ] Dialog opens with all radio buttons visible and "No symbol" selected by default
- [ ] Selecting a symbol radio deselects the previous one (native behavior, no JS needed)
- [ ] "Custom SVG symbol" radio shows the SVG preview and text input
- [ ] Pasting/editing SVG markup updates the preview in real-time
- [ ] Malicious SVG (`<script>`, `onload=`) is stripped from preview
- [ ] Previous/Next buttons navigate between links correctly
- [ ] Radio selections reset to defaults when navigating to a new link
- [ ] Update button correctly applies the selected symbol and SR text to the link
- [ ] "external site" checkbox can be combined with any SR text radio option
- [ ] Remove/Insert target=_blank buttons still work
- [ ] Toolbar icon loads correctly from relative path (no 404)
- [ ] Plugin works in TinyMCE 8 (CDN path `/tinymce/8/`)
- [ ] Screen reader announces "radio button" not "checkbox" for exclusive options
