# Agent Skill: UX/UI Design & Bootstrap 5.3+ Engineering Standard

* **Role:** Principal Product Designer & Front-End Systems Architect
* **Stack:** HTML5, Bootstrap 5.3+ (Strict: native Bootstrap components, utility classes, JS APIs, and `--bs-*` / `--theme-*` CSS custom properties ONLY)
* **Brand Palette:** White (`#FFFFFF` / `#F8F9FA`), Black (`#0A0A0D` / `#141419`), Blue (`#0057FF` / `#3B82F6`)
* **Scope:** Framework- and backend-agnostic. Applies unconditionally to all markup, styles, layouts, interactive flows, and client-side scripts.
* **Normative Language (RFC 2119):** `MUST`, `MUST NOT`, `REQUIRED`, `SHALL`, `SHALL NOT`, `MAY`, `FORBIDDEN`.

---

## 1. Core Mandate & Prohibitions

1. **Intent Before Aesthetics:** Every interface MUST be derived from primary user intent, secondary browsing intent, and failure/edge cases before visual styling is applied.
2. **Respect Mental Models:** Primary navigation MUST anchor at the top (`navbar`), content MUST flow top-to-bottom and left-to-right, and primary Calls-to-Action (CTAs) MUST remain high-contrast and immediately discoverable in the top-right header and hero region.
3. **Strict Bootstrap 5.3+ Boundary:**
   * All layout, spacing, color, elevation, and interaction patterns MUST use native Bootstrap 5.3+ classes and components (`.container`, `.row`, `.col-*`, `.vstack`, `.hstack`, `.gap-*`, `.form-floating`, `.btn-*`, `.card`, `.nav-pills`, `.offcanvas`, `.modal`, `.toast`, `.tooltip`, `.badge`, `.alert`, `.pagination`).
   * Custom component CSS classes, inline `style=""` attributes, hardcoded hex colors inside HTML, third-party CSS frameworks, and external runtime CDNs are **FORBIDDEN**.
   * Stylesheets MUST contain ONLY `:root` / `[data-bs-theme]` `--bs-*` / `--theme-*` CSS variable mappings and `:focus-visible` rules, loaded via self-hosted assets or CSP-nonce-protected `<style>` blocks.
4. **Zero Fallbacks & Zero Silent Errors:** Client-side logic MUST NOT swallow exceptions silently or mask missing required DOM targets. Operations MUST succeed deterministically or surface explicit feedback.

---

## 2. Brand & Semantic Color System (White / Black / Blue)

Blue MUST serve as the sole primary brand hue for interactive focus, links, and primary CTAs. Semantic colors (`success`, `danger`, `warning`) are **FORBIDDEN** for decoration and MUST be used strictly as state signifiers (`success` = completion/valid, `danger` = error/destructive, `warning` = caution/non-blocking alert).

### Mandatory CSS Token Mapping (`[data-bs-theme="light" | "dark"]`)

```css
:root,
[data-bs-theme="light"] {
  --bs-primary: #0057ff;
  --bs-primary-rgb: 0, 87, 255;
  --bs-primary-bg-subtle: #ebf1ff;
  --bs-primary-border-subtle: #b8d0ff;
  --bs-primary-text-emphasis: #003399;

  --bs-body-bg: #ffffff;
  --bs-body-bg-rgb: 255, 255, 255;
  --bs-body-color: #0a0a0d;
  --bs-body-color-rgb: 10, 10, 13;

  --bs-secondary-bg: #f8f9fa;
  --bs-tertiary-bg: #f1f3f5;
  --bs-secondary-color: rgba(10, 10, 13, 0.62);
  --bs-border-color: rgba(10, 10, 13, 0.1);

  --bs-link-color: #0057ff;
  --bs-link-hover-color: #0041c2;
  --bs-focus-ring-color: rgba(0, 87, 255, 0.28);

  --bs-box-shadow-sm: 0 2px 8px rgba(10, 10, 13, 0.04);
  --bs-box-shadow: 0 8px 24px rgba(10, 10, 13, 0.07);
  --bs-box-shadow-lg: 0 16px 40px rgba(10, 10, 13, 0.12);
}

[data-bs-theme="dark"] {
  --bs-primary: #3b82f6;
  --bs-primary-rgb: 59, 130, 246;
  --bs-primary-bg-subtle: rgba(59, 130, 246, 0.14);
  --bs-primary-border-subtle: rgba(59, 130, 246, 0.3);
  --bs-primary-text-emphasis: #93c5fd;

  --bs-body-bg: #0a0a0d;
  --bs-body-bg-rgb: 10, 10, 13;
  --bs-body-color: #f8f9fa;
  --bs-body-color-rgb: 248, 249, 250;

  --bs-secondary-bg: #121217;
  --bs-tertiary-bg: #1a1a22;
  --bs-secondary-color: rgba(248, 249, 250, 0.62);
  --bs-border-color: rgba(248, 249, 250, 0.1);

  --bs-box-shadow-sm: none;
  --bs-box-shadow: 0 0 0 1px rgba(248, 249, 250, 0.08);
  --bs-box-shadow-lg: 0 16px 40px rgba(0, 0, 0, 0.55);
}

:root,
[data-bs-theme="light"],
[data-bs-theme="dark"] {
  --theme-bg-canvas: var(--bs-body-bg);
  --theme-card-bg: var(--bs-card-bg);
  --theme-text-primary: var(--bs-body-color);
  --theme-border-color: var(--bs-border-color);
  --theme-accent: var(--bs-primary);
}

:focus-visible {
  outline: 2px solid var(--theme-accent) !important;
  outline-offset: 3px !important;
}
```

### Theme & Elevation Rules

* **Opacity-Based Hierarchy:** Primary labels MUST use `text-body`; secondary metadata MUST use `text-body-secondary` (~62% contrast), including when adjacent font sizes are identical.
* **Dark Mode (`data-bs-theme="dark"`):**
  * High-contrast bright borders around cards are **FORBIDDEN**. Cards MUST use `border` (`--bs-border-color`) and achieve depth by stepping surfaces lighter than canvas (`bg-body-tertiary` cards over `bg-body` / `bg-body-secondary`).
  * Status badges MUST use desaturated subtle tokens: `bg-primary-subtle text-primary-emphasis border border-primary-subtle` (or `success` / `danger` / `warning` equivalents).
* **Light Mode Shadows:** Drop shadows MUST NOT dominate visual attention. Resting cards MUST use `shadow-sm` or `shadow-none border`; `shadow-lg` is reserved exclusively for elevated overlays (`dropdown-menu`, `popover`, `modal`, `toast`).

---

## 3. Typography & Scale Rules

* **Single Font Family:** Layouts MUST use one sans-serif font stack (`--bs-font-sans-serif`). Mixing multiple font families is **FORBIDDEN**.
* **Scale Cap (Max 6 Levels):** Any single view MUST NOT exceed 6 distinct font sizes.
* **Context-Locked Scales:**
  * **Marketing / Landing Pages:** Hero `h1` via `.display-4`–`.display-5` (`fw-bold lh-1`), section headings `h2`–`h3`, body `fs-6` (`1rem`), captions `small` (`0.875rem`).
  * **High-Density Dashboards / Tables:** `h1` MUST NOT exceed `fs-4` (`1.5rem` / `24px`); `h2` `fs-5` (`20px`); `h3` `fs-6` (`16px`); metadata `small` (`0.875rem` / `14px`).

---

## 4. Grid, Spacing & Proximity Architecture

All spacing MUST adhere to Bootstrap's 4pt-compatible scale (`-1` = `0.25rem`/4px, `-2` = `0.5rem`/8px, `-3` = `1rem`/16px, `-4` = `1.5rem`/24px, `-5` = `3rem`/48px).

### Proximity Grouping Rules

1. **Tightly Coupled (`gap-2` / `mb-2` — `8px`):** Heading to subtext; icon to label; entity title to timestamp.
2. **Eyebrow Separation (`mb-3` — `16px`):** Status badge/pill above a primary heading.
3. **Action Separation (`mt-4` — `24px`):** Text block to CTA button group (`hstack gap-2 mt-4`).
4. **Layout Grids:**
   * Repeating collections (catalog cards, dashboards, bento feature grids) MUST use Bootstrap's 12-column grid (`row g-4` with `col-12 col-md-6 col-xl-4`).
   * Hero sections MUST prioritize whitespace (`py-5 my-lg-4`) over artificial column density.

---

## 5. UX Patterns & Native Bootstrap 5.3+ Mappings

### A. Affordances & Signifiers

* **Segmented Controls:** MUST use `nav nav-pills bg-body-tertiary p-1 rounded-pill d-inline-flex`. Active option MUST use `nav-link active rounded-pill`; inactive options MUST use `nav-link text-body-secondary`; disabled options MUST use `nav-link disabled opacity-50`.
* **Buttons & Icons:**
  * Buttons MUST preserve a **2:1 horizontal-to-vertical padding ratio** using Bootstrap native `.btn` (`py-2 px-3` or `.btn-lg`).
  * Primary vs Secondary CTA pairs MUST use `.hstack.gap-2` with `.btn.btn-primary` paired with `.btn.btn-outline-secondary` (or `.btn.btn-link.text-decoration-none.text-body` for ghost actions).
  * Icon dimensions MUST match adjacent text `line-height` (`width="20" height="20"` or `24x24` inline SVG with `aria-hidden="true"`).
  * Icon-only buttons MUST declare `aria-label` and initialize `bootstrap.Tooltip`.

### B. Entity Cards (No Spreadsheet Tables)

Rendering single-entity summary cards as two-column `Label: Value` tables is **FORBIDDEN**. Entity cards MUST follow this structural hierarchy:

1. **Header (`hstack gap-3 align-items-center`):** Thumbnail/icon container (`rounded-3 bg-body-secondary p-2`) + `vstack` (Bold Title `fw-semibold text-truncate` + Timestamp `small text-body-secondary`) + Right-aligned metric (`ms-auto fw-bold text-primary`).
2. **Body:** Visual relationship indicators (`border-start ps-3 vstack gap-2`) instead of redundant key-label strings.
3. **Footer (`hstack gap-2 justify-content-end`):** Primary action (`btn-primary`) + Secondary action (`btn-outline-secondary`).

### C. Hero Section Anatomy (4 Mandatory Zones)

1. **Navbar (`header.navbar.navbar-expand-lg.sticky-top.bg-body.border-bottom`):** `<= 5` top-level nav links; right-aligned primary CTA (`btn-primary`); overflow navigation moved into Bootstrap `offcanvas`.
2. **Text Block:** Optional eyebrow badge (`badge bg-primary-subtle text-primary-emphasis rounded-pill`) + Heading (`<= 7 words`) + Subtext (`<= 14 words`, `text-body-secondary`) + CTA pair (`hstack gap-2 mt-4`).
3. **Stats Strip (Optional):** 3-column grid separated by `border-start`, uniform font size (`small`), differentiated strictly by `fw-semibold text-body` vs `text-body-secondary`.
4. **Focal Visual:** Clean, unobstructed subject. Text MUST NOT overlap bright or high-frequency focal subjects.

### D. Image Overlays & Edge-Case Resilience

* **Contrast Scrims:** Placing raw text directly over photography is **FORBIDDEN**. Text inside `.card-img-overlay` MUST reside inside a native Bootstrap dark surface block (`mt-auto p-3 p-md-4 bg-dark bg-opacity-75 text-white rounded-bottom`) ensuring WCAG AA legibility.
* **Floating Overlay Icons:** Action icons over bright listing images MUST sit inside `.bg-dark.bg-opacity-75.text-white.rounded-circle.p-2`.
* **Long-String Protection:** Dynamic titles MUST apply `.text-truncate` so variable-length data never breaks card alignment.

### E. Progressive Disclosure & Pagination

* **Condensing Navigation/Search:** Multi-input filters or secondary navigation MUST condense cleanly into Bootstrap `Collapse`, `Modal`, or `Offcanvas` at smaller viewports or scrolled states.
* **Explicit Pagination Over Infinite Scroll:** Large collections MUST use an explicit `<button class="btn btn-outline-secondary">` ("Load more") or Bootstrap `.pagination.pagination-sm.justify-content-center.flex-wrap.mb-0` (with `.page-item.active` `aria-current="page"` and `.page-item.disabled`) so users retain control and can reach `<footer>`.

---

## 6. Interaction Feedback & Required States

Every user interaction MUST produce immediate, deterministic visual feedback.

### Mandatory 5 Button States

1. **Default:** `.btn-primary`, `.btn-outline-secondary`, or `.btn-outline-danger` / `.btn-danger`.
2. **Hover:** Native Bootstrap `--bs-btn-hover-*` state.
3. **Active / Pressed:** Native Bootstrap `--bs-btn-active-*` state.
4. **Disabled:** `disabled` attribute (`aria-disabled="true"` on links).
5. **Loading:** Set `btn.disabled = true` and render a DOM-constructed `<span class="spinner-border spinner-border-sm me-2" role="status" aria-hidden="true"></span>` alongside status text (`Saving...` / `Loading...`).

### Mandatory 5 Form Input States

All forms MUST use `.form-floating` (or explicit `.form-label` + `.form-control`) and native validation classes:

1. **Default:** `.form-control`
2. **Focus:** `.form-control.focus-ring`
3. **Error:** `.form-control.is-invalid` + `<div class="invalid-feedback">`
4. **Warning (Non-blocking):** `.form-control.border-warning` + `<div class="form-text text-warning-emphasis">`
5. **Valid:** `.form-control.is-valid` + `<div class="valid-feedback">`

Form submission button groups MUST use `.hstack.gap-2.justify-content-end` (`btn-outline-secondary` Cancel + `btn-primary` Submit). Flash/status alerts MUST use `.alert.alert-{success|danger}.alert-dismissible.fade.show`.

---

## 7. Multi-Step UX Flow Architecture

1. **Progress Signifier:** Multi-step flows MUST render a Bootstrap `.progress` bar (`role="progressbar"`, `aria-valuenow`, `aria-valuemin="0"`, `aria-valuemax="100"`).
2. **Escape Hatch:** Onboarding/registration views MUST include an explicit secondary route (`Already have an account?` -> `Log in`).
3. **Verification Friction:** Systems MUST explicitly enforce either blocking verification (`disabled` Continue button until verified) or soft verification (clickable Continue with `.alert-warning` reminder).
4. **Conditional Branching:** Qualifying attributes (e.g., `team_size`) MUST deterministically branch subsequent steps (`team_size > 1` -> Invite Team step -> Workspace; `team_size === 1` -> Workspace directly).

---

## 8. XSS-Safe, Deterministic Bootstrap 5.3+ JS Controller

* Inline `on*` HTML attributes are **FORBIDDEN**.
* Mutating `.innerHTML` with dynamic/interpolated strings is **FORBIDDEN** (use `document.createElement()`, `textContent`, and `replaceChildren()`).
* Silent `catch` blocks are **FORBIDDEN**.

```javascript
'use strict';

document.addEventListener('DOMContentLoaded', () => {
  // 1. Initialize Bootstrap Tooltips
  document.querySelectorAll('[data-bs-toggle="tooltip"]').forEach((el) => {
    new bootstrap.Tooltip(el);
  });

  // 2. Deterministic Light/Dark Theme Toggle
  const themeBtn = document.getElementById('themeToggle');
  if (themeBtn instanceof HTMLButtonElement) {
    themeBtn.addEventListener('click', () => {
      const root = document.documentElement;
      const current = root.getAttribute('data-bs-theme');
      const next = current === 'dark' ? 'light' : 'dark';
      root.setAttribute('data-bs-theme', next);
      themeBtn.setAttribute('aria-pressed', next === 'dark' ? 'true' : 'false');
    });
  }

  // 3. Copy Micro-Interaction with Toast Feedback (Zero innerHTML, Zero Silent Swallows)
  const toastEl = document.getElementById('feedbackToast');
  const toastBody = document.getElementById('feedbackToastBody');
  const toast = toastEl instanceof HTMLElement ? new bootstrap.Toast(toastEl, { delay: 2500 }) : null;

  document.querySelectorAll('[data-copy-text]').forEach((btn) => {
    if (!(btn instanceof HTMLButtonElement)) {
      return;
    }
    btn.addEventListener('click', async () => {
      const value = btn.getAttribute('data-copy-text');
      if (value === null || value === '') {
        throw new Error('Missing non-empty data-copy-text attribute.');
      }

      const originalText = btn.textContent ?? '';
      try {
        await navigator.clipboard.writeText(value);
        btn.classList.remove('btn-outline-secondary');
        btn.classList.add('btn-success');
        btn.textContent = 'Copied';

        if (toast !== null && toastBody instanceof HTMLElement) {
          toastBody.textContent = `Copied "${value}" to clipboard.`;
          toast.show();
        }
      } catch (err) {
        btn.classList.remove('btn-outline-secondary');
        btn.classList.add('btn-danger');
        btn.textContent = 'Copy Failed';
        if (toast !== null && toastBody instanceof HTMLElement) {
          toastBody.textContent = 'Clipboard permission denied or unavailable.';
          toast.show();
        }
        console.error('Clipboard write failed:', err);
      } finally {
        window.setTimeout(() => {
          btn.classList.remove('btn-success', 'btn-danger');
          btn.classList.add('btn-outline-secondary');
          btn.textContent = originalText;
        }, 1800);
      }
    });
  });

  // 4. Deterministic Loading State Helper for Async Submit/Action Buttons
  document.querySelectorAll('[data-loading-btn]').forEach((btn) => {
    if (!(btn instanceof HTMLButtonElement)) {
      return;
    }
    btn.addEventListener('click', () => {
      if (btn.disabled) {
        return;
      }
      const originalText = btn.textContent ?? 'Submit';
      const spinner = document.createElement('span');
      spinner.className = 'spinner-border spinner-border-sm me-2';
      spinner.setAttribute('role', 'status');
      spinner.setAttribute('aria-hidden', 'true');

      btn.disabled = true;
      btn.replaceChildren(spinner, document.createTextNode('Processing...'));

      btn.dataset.originalLabel = originalText;
    });
  });
});
```

---

## 9. Pre-Delivery Verification Gate

Every generated interface MUST pass all checks before completion:

1. **HTML5 Landmarks & Nesting:** Uses `<header>`, `<nav>`, `<main>`, `<section>`, `<footer>`; conditional template nesting depth `<= 1`.
2. **Strict Bootstrap 5.3+ Compliance:** Zero custom CSS component classes, zero inline `style=""`, zero inline `on*` JS handlers, zero external CDN links, zero hardcoded hex colors in HTML.
3. **Brand Compliance:** Strictly White (`#FFFFFF`/`#F8F9FA`), Black (`#0A0A0D`/`#141419`), Blue (`#0057FF`/`#3B82F6`), with semantic colors reserved strictly for status/validation.
4. **Dual-Theme Parity:** Both `data-bs-theme="light"` and `data-bs-theme="dark"` render accessible contrast (`>= 4.5:1` body text), muted card borders, and proper surface stepping (`bg-body-tertiary` over `bg-body`).
5. **Typography & Proximity:** `<= 6` font sizes; `8px` (`gap-2`) coupled pairs; `24px` (`mt-4`) separation before CTA groups; `.text-truncate` on variable-length titles.
6. **Complete States & XSS Hygiene:** All 5 button states and 5 input states implemented; DOM updates strictly via `textContent`/`replaceChildren()`; zero silent `catch` blocks.
