# 12 — DOM Compatibility

**LIRAQ v1.0**

---

## 1 Purpose

This document describes how a LIRAQ UIDL document maps to a browser DOM for
surfaces with the `visual` capability running in a web browser. This is one
possible surface renderer implementation — it is not the normative presentation
model.

The DOM compatibility layer is a **projection concern**: it sits inside a
visual-browser surface renderer and translates semantic UIDL into HTML elements
styled by presentation profiles.

---

## 2 Structural Equivalence

UIDL and DOM share structural properties:

| Property | UIDL | DOM |
|----------|------|-----|
| Tree structure | Yes | Yes |
| Unique identifiers | `id` attribute | `id` attribute |
| Parent-child relationships | Element nesting | Node nesting |
| Ordered children | Yes | Yes |
| Attribute key-value pairs | Yes | Yes |

A UIDL document maps structurally 1:1 to a DOM subtree. Every UIDL element
becomes a DOM element. Every parent-child relationship is preserved. Element
order is preserved.

---

## 3 Element Type Mapping

Each UIDL element type maps to an HTML element with appropriate ARIA roles:

| UIDL Element | HTML Element | ARIA Role | Notes |
|-------------|-------------|-----------|-------|
| `<region>` | `<section>` | `region` | Landmark region |
| `<group>` | `<div>` | `group` | Generic container |
| `<separator>` | `<hr>` | `separator` | |
| `<meta>` | — | — | Not rendered; metadata stored as `data-*` attributes on root |
| `<label>` | `<span>` or `<p>` | — | `<p>` if block-level context |
| `<media>` | Varies | — | See §4 |
| `<symbol>` | `<span>` | `img` | With `aria-label` for the symbol's meaning |
| `<canvas>` | `<canvas>` | `img` or `application` | Depends on interactivity |
| `<action>` | `<button>` | `button` | |
| `<input>` | `<input>` or `<textarea>` | — | Type from `input-type` attribute |
| `<selector>` | `<select>` or `<fieldset>` | `listbox` or `radiogroup` | Single vs. multi mode |
| `<toggle>` | `<button>` | `switch` | `aria-checked` bound to state |
| `<range>` | `<input type="range">` | `slider` | |
| `<collection>` | `<div>` or `<ul>` | `list` | `<ul>` if items are homogeneous |
| `<table>` | `<table>` | `table` | With `<thead>`, `<tbody>` |
| `<indicator>` | `<div>` | `progressbar` or `status` | Based on `mode` |

---

## 4 Media Representation in DOM

A `<media>` element on a visual-browser surface selects its `visual`
representation:

| Rep `type` attribute | DOM rendering |
|---------------------|---------------|
| `src` pointing to `.svg`, `.png`, `.jpg`, `.webp` | `<img>` element |
| `src` pointing to `.mp4`, `.webm` | `<video>` element |
| `src` pointing to `.mp3`, `.ogg` | `<audio>` element (for auditory surfaces; may be hidden on visual) |
| `text` attribute | `<span>` with text content |

The `description` representation maps to `alt` (for `<img>`) or
`aria-description` (for other elements).

---

## 5 Operation Mapping

UIDL document operations map directly to DOM methods:

| UIDL Operation | DOM Equivalent |
|---------------|----------------|
| `add-element` | `parentNode.insertBefore(newNode, referenceNode)` |
| `remove-element` | `parentNode.removeChild(node)` |
| `move-element` | `newParent.insertBefore(node, referenceNode)` |
| `set-attribute` | `element.setAttribute(name, value)` |
| `remove-attribute` | `element.removeAttribute(name)` |
| `replace-children` | `element.replaceChildren(...newChildren)` |
| `set-text` | `element.textContent = text` |

---

## 6 Arrangement → CSS Mapping

Arrangement hints map to CSS layout:

| Arrangement | CSS Layout |
|-------------|-----------|
| `dock` | CSS Grid with named areas |
| `flex` | Flexbox (`display: flex`) |
| `stack` | CSS Grid with `grid-area: 1/1` for all children (stacking) |
| `flow` | Normal flow (`display: block` / `display: inline`) |
| `grid` | CSS Grid (`display: grid`) |
| `none` | Default flow |

### 6.1 Dock Mapping

```css
/* dock-position mapping */
.dock-container { display: grid; grid-template: "before before before" auto
                                                 "start fill end" 1fr
                                                 "after after after" auto / auto 1fr auto; }
[dock-position="before"] { grid-area: before; }
[dock-position="start"]  { grid-area: start; }
[dock-position="fill"]   { grid-area: fill; }
[dock-position="end"]    { grid-area: end; }
[dock-position="after"]  { grid-area: after; }
```

### 6.2 Flex Mapping

```css
.flex-container { display: flex; gap: var(--gap); }
/* flex-weight → flex-grow */
[flex-weight] { flex-grow: attr(flex-weight); flex-basis: 0; }
```

---

## 7 Attention → Focus Mapping

UIDL's attention model maps to the DOM focus model:

| UIDL Concept | DOM Equivalent |
|-------------|----------------|
| Attention | `:focus` / `document.activeElement` |
| `navigate-next` | Tab key / `focus()` on next element |
| `navigate-prev` | Shift+Tab |
| `attention-order` | `tabindex` attribute |
| `attention-order="none"` | `tabindex="-1"` |
| Dialog attention trap | `<dialog>` element or `aria-modal` with focus trap script |

### 7.1 Focus Management

The visual-browser surface renderer maintains a mapping between UIDL attention
state (`_surfaces.<id>.attention`) and DOM focus. When:

- UIDL attention changes → call `element.focus()` on the corresponding DOM node.
- DOM focus changes (user interaction) → update `_surfaces.<id>.attention` in
  the state tree.

---

## 8 Announcement → ARIA Live Region Mapping

The visual-browser renderer creates a set of ARIA live regions for
announcements:

```html
<div id="liraq-announce-polite" aria-live="polite" aria-atomic="true" class="sr-only"></div>
<div id="liraq-announce-assertive" aria-live="assertive" aria-atomic="true" class="sr-only"></div>
```

When an `announce` operation fires:

1. Select the appropriate live region based on priority.
2. Set its `textContent` to the announcement message.
3. For `interrupt` priority, also set `role="alert"`.

The `.sr-only` class hides the region visually while keeping it accessible to
screen readers. For visual rendering of announcements (toasts, overlays), a
separate visual component handles the presentation profile's rules.

---

## 9 Presentation Profile → CSS Mapping

Visual presentation profile properties map to CSS:

| Profile Property | CSS Property |
|-----------------|-------------|
| `font-family` | `font-family` |
| `font-size` | `font-size` (in px) |
| `font-weight` | `font-weight` |
| `color` | `color` |
| `background` | `background-color` |
| `border` | `border` |
| `border-radius` | `border-radius` (in px) |
| `padding` | `padding` (in px; arrays map to multi-value) |
| `gap` | `gap` (in px) |
| `opacity` | `opacity` |
| `cursor` | `cursor` |
| `outline` | `outline` |
| `text-transform` | `text-transform` |
| `line-height` | `line-height` |
| `transition-duration` | `transition-duration` (in ms) |
| `animation` | `animation-name` |

The renderer generates CSS custom properties or inline styles based on the
resolved cascade.

---

## 10 Accommodation → CSS/ARIA Mapping

| Accommodation | CSS/DOM Effect |
|--------------|----------------|
| `text-scale` | `font-size` multiplied; or `html { font-size: calc(100% * scale) }` |
| `high-contrast` | Substitute CSS custom properties with high-contrast palette |
| `reduced-motion` | `@media (prefers-reduced-motion)` or force `animation: none; transition: none;` |
| `color-shift` | CSS `filter` or SVG filter for color matrix transformation |
| `attention-indicators: enhanced` | Thicker, higher-contrast focus outlines |

---

## 11 Semantic Role → ARIA Mapping

| UIDL Role | ARIA Role / Property |
|-----------|---------------------|
| `navigation` | `role="navigation"` |
| `primary` | `role="main"` |
| `secondary` | `role="complementary"` |
| `status` | `role="status"` |
| `alert` | `role="alert"` |
| `notification` | `role="log"` |
| `control` | `role="toolbar"` |
| `data` | `role="region"` with `aria-label` |
| `input` | `role="form"` |
| `header` | `role="banner"` or `<h1>`–`<h6>` |
| `footer` | `role="contentinfo"` |
| `toolbar` | `role="toolbar"` |
| `dialog` | `role="dialog"` + `aria-modal="true"` |
| `tooltip` | `role="tooltip"` |

---

## 12 Implementation Architecture

```
LIRAQ Runtime
     │
     ▼
Visual-Browser Surface Renderer
     │
     ├─ DOM Builder       ← creates/updates HTML elements from UIDL
     ├─ CSS Generator     ← generates styles from presentation profile
     ├─ Focus Manager     ← syncs UIDL attention ↔ DOM focus
     ├─ Live Region Mgr   ← routes announcements to ARIA live regions
     └─ Input Adapter     ← translates DOM events to UIDL input events
```

The renderer listens for UIDL document mutations and incrementally updates the
DOM. It does NOT re-render the entire tree on every change — it applies diffs.

---

## 13 Limitations

The DOM compatibility layer is a convenience for web-browser deployment. It is
not required, and it introduces constraints:

- **DOM ordering:** HTML requires a single linear child order. UIDL's `stack`
  arrangement (concurrent layering) must be simulated with CSS positioning.
- **Inline styles vs. classes:** The renderer may use either approach. CSS class
  generation is more efficient for large documents.
- **Custom elements:** Implementations may use Web Components (`<liraq-action>`,
  `<liraq-indicator>`, etc.) instead of standard HTML elements.
- **Shadow DOM:** Encapsulation via Shadow DOM is permitted but not required.

This mapping is one surface renderer among many. The same UIDL document may
simultaneously be projected onto an auditory surface via speech synthesis and a
tactile surface via braille output, neither of which involve the DOM at all.
