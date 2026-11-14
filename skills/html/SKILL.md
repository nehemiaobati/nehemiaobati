---
name: ux-ui-bootstrap5-architect
description: End-to-end UX/UI design and front-end engineering standard for HTML5 and Bootstrap 5.3+ using a White, Black, and Blue brand palette.
version: 1.0.0
stack:
  - HTML5
  - Bootstrap 5.3+
  - Vanilla ES6+ JavaScript
brand_palette:
  white: "#FFFFFF"
  black: "#0A0A0D"
  blue: "#0057FF"
---

# UX/UI Design & Bootstrap 5.3+ Engineering Skill

## 1. Scope, Compliance & Normative Rules

* **Universal Scope:** This specification applies unconditionally to every layout, component, stylesheet, template, form, overlay, and client-side script.
* **Domain & Stack Agnosticism:** All patterns MUST remain backend- and domain-agnostic. Examples use generic placeholders (`[BrandName]`, `[Domain]`, `[Entity]`, `[Route]`, `[AssetPath]`).
* **Deterministic Language (RFC 2119):** Specifications use ONLY `MUST`, `MUST NOT`, `REQUIRED`, `SHALL`, `SHALL NOT`, `MAY`, `FORBIDDEN`.
* **Strict Bootstrap 5.3+ Boundary:**
  * All UI structures, spacing, typography, states, and themes MUST use native Bootstrap 5.3+ components, utility classes, CSS custom properties (`--bs-*`), and native `bootstrap.*` JavaScript APIs.
  * Custom component CSS classes, inline `style=""` attributes, inline `on*` JS event attributes, third-party CSS frameworks, hardcoded hex colors in HTML, and external runtime CDN links are **FORBIDDEN**.
  * Stylesheets MUST contain ONLY `:root` / `[data-bs-theme]` `--bs-*` and `--theme-*` variable mappings plus `:focus-visible` accessibility rules.
* **Zero Fallbacks & Zero Silent Errors:** Client-side scripts MUST NOT mask missing required DOM elements with silent fallbacks or swallow exceptions in empty `catch` blocks. Operations MUST complete deterministically or surface explicit error states.

---

## 2. Brand & Semantic Color System (White / Black / Blue)

### 2.1 Color Discipline

* **Single Brand Hue:** Blue (`#0057FF` light / `#3B82F6` dark) MUST serve as the sole primary hue for interactive focus, links, active metrics, and primary Calls-to-Action (CTAs).
* **Monochrome Surfaces:** White (`#FFFFFF`, `#F8F9FA`, `#F1F3F5`) and Black (`#0A0A0D`, `#121217`, `#1A1A22`) MUST form all canvas, card, and ink layers.
* **Semantic Colors Only for State:** `success`, `danger`, and `warning` tokens are **FORBIDDEN** for decorative styling and MUST signify system state exclusively (`success` = valid/completed, `danger` = error/destructive, `warning` = caution/non-blocking alert).

### 2.2 Mandatory CSS Variable Mapping

Place inside the self-hosted theme stylesheet or CSP-nonced layout `<style>` block:

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

  --bs-link-color: #60a5fa;
  --bs-link-hover-color: #93c5fd;
  --bs-focus-ring-color: rgba(59, 130, 246, 0.35);

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

### 2.3 Dual-Theme, Surface & Shadow Rules

* **Opacity Hierarchy:** Primary labels MUST use `text-body`; supporting labels and timestamps MUST use `text-body-secondary` (~62% contrast), including when font sizes are identical.
* **Dark Mode (`data-bs-theme="dark"`):**
  * High-contrast white borders on cards are **FORBIDDEN**. Cards MUST use `border` (`--bs-border-color`) and create depth via lighter surface stepping (`bg-body-tertiary` cards over `bg-body` or `bg-body-secondary`).
  * Status chips/badges MUST use desaturated subtle utilities: `.badge.bg-{variant}-subtle.text-{variant}-emphasis.border.border-{variant}-subtle`.
  * Heavy drop shadows in dark mode are **FORBIDDEN**.
* **Light Mode Shadows (`data-bs-theme="light"`):**
  * Drop shadows MUST NOT be the first noticed visual element. Resting cards MUST use `shadow-sm border`; `shadow-lg` is reserved exclusively for floating overlays (`dropdown-menu`, `popover`, `modal`, `toast`).

---

## 3. Typography & Hierarchy Scale

* **Single Font Stack:** All UI MUST use Bootstrap's single sans-serif stack (`--bs-font-sans-serif`). Mixing multiple font families is **FORBIDDEN**.
* **Six-Size Maximum:** Any single view MUST NOT exceed 6 distinct font sizes.
* **Scale by Information Density:**
  * **Marketing / Landing Pages:** Hero heading via `.display-4`–`.display-5.fw-bold.lh-1`; section titles `h2`–`h3`; body `fs-6`; metadata `small`.
  * **Dashboards / Dense Workspaces:** `h1` MUST NOT exceed `fs-4` (`1.5rem` / `24px`); `h2` `fs-5`; `h3` `fs-6`; metadata `small`.
* **Copy Conciseness:** Hero headlines MUST NOT exceed 7 words; hero subtext MUST NOT exceed 14 words.

---

## 4. 4pt Grid, Spacing & Proximity Rules

All layout spacing MUST use Bootstrap's 4pt-compatible scale (`-1` = `4px`, `-2` = `8px`, `-3` = `16px`, `-4` = `24px`, `-5` = `48px`).

### 4.1 Proximity Grouping

1. **Tightly Coupled (`gap-2` / `mb-2` — `8px`):** Heading to subtext; icon to label; entity title to timestamp.
2. **Eyebrow Separation (`mb-3` — `16px`):** Status badge above a heading.
3. **Action Separation (`mt-4` — `24px`):** Copy block to CTA button group (`.hstack.gap-2.mt-4`).
4. **Section Whitespace:** Major vertical sections MUST preserve generous breathing room (`py-5`).

### 4.2 Grid Allocation

* **Structured Repeating Content:** Galleries, catalogs, dashboards, and multi-feature bento grids MUST use Bootstrap 12-column grids (`.row.g-4` with `.col-12.col-md-6.col-xl-4`).
* **Custom Hero Layouts:** Hero regions MUST prioritize whitespace and visual balance over rigid 12-column alignment of decorative assets.

---

## 5. UX Architecture & Component Patterns

### 5.1 Mental Models & User Intent

1. **Intent-First Structure:** Views MUST prioritize the user's primary task first (e.g., search/action bar) and secondary browsing intent second (e.g., filter pills + listing grid). Functionality MUST expand only as user intent expands.
2. **Reading Flow:** Content MUST flow top-to-bottom and left-to-right. Top navigation MUST use `header.navbar.navbar-expand-lg.sticky-top.bg-body.border-bottom` with `<= 5` links and one primary `btn-primary` CTA on the right. Overflow links MUST move into `offcanvas.offcanvas-end`.
3. **Explicit Pagination Over Infinite Scroll:** Multi-page collections MUST render either an explicit `<button type="button" class="btn btn-outline-secondary">` ("Load more" with loading state) or Bootstrap `.pagination.pagination-sm.justify-content-center.flex-wrap.mb-0` so users retain control and can reach `<footer>`.

### 5.2 Affordances, Signifiers & Buttons

* **Segmented Controls:** MUST use `.nav.nav-pills.bg-body-tertiary.p-1.rounded-pill.d-inline-flex`. Active tabs MUST carry `.active.rounded-pill`; disabled tabs MUST carry `.disabled.opacity-50`.
* **Button Padding & Pairing:** Buttons MUST maintain a **2:1 horizontal-to-vertical padding ratio** via native `.btn` (`.px-3.py-2` or `.btn-lg`). Primary + secondary CTA pairs MUST use `.hstack.gap-2` combining `.btn.btn-primary` with `.btn.btn-outline-secondary` (or `.btn.btn-link.text-decoration-none.text-body` for ghost actions).
* **Icon Sizing:** Inline SVG icons MUST match the adjacent text `line-height` (`width="20" height="20"` or `24x24` with `aria-hidden="true"`).
* **Icon-Only Controls:** Every icon-only button MUST define `aria-label`, `data-bs-toggle="tooltip"`, and `data-bs-title`.

### 5.3 Entity Cards (Spreadsheet Layout Prohibited)

Rendering entity summary cards as two-column `Label: Value` key-value tables is **FORBIDDEN**. Entity cards MUST follow this hierarchy:

1. **Header (`.hstack.gap-3.align-items-center`):** Square icon/thumbnail box (`.rounded-3.bg-body-secondary.p-2`) + `.vstack` (Bold truncated title `.fw-semibold.text-truncate` + muted subtitle `.small.text-body-secondary`) + Right-aligned metric (`.ms-auto.fw-bold.text-primary`).
2. **Body:** Structural grouping via `.border-start.ps-3.vstack.gap-2` for related metadata (e.g., origin/destination).
3. **Footer (`.hstack.gap-2.justify-content-end`):** Primary + secondary actions.

### 5.4 Image Overlays & Edge-Case Resilience

* **Directional Contrast Scrim:** Placing unbacked text directly over photography or flat dark full-card overlays is **FORBIDDEN**. Text inside `.card-img-overlay` MUST sit inside `.mt-auto.p-3.p-md-4.bg-dark.bg-opacity-75.text-white.rounded-bottom` (parent `.card-img-overlay.d-flex.flex-column.p-0`).
* **Overlay Action Icons:** Floating action icons over bright images MUST sit inside `.bg-dark.bg-opacity-75.text-white.rounded-circle.p-2`.
* **Truncation:** Variable-length dynamic titles and subtitles MUST apply `.text-truncate` to prevent layout breaks.

---

## 6. Mandatory Interaction States & Multi-Step UX Flows

### 6.1 Five Button States

Every interactive button MUST support:

1. **Default:** `.btn-primary`, `.btn-outline-secondary`, or `.btn-outline-danger` / `.btn-danger`.
2. **Hover:** Native `--bs-btn-hover-*`.
3. **Active / Pressed:** Native `--bs-btn-active-*`.
4. **Disabled:** `disabled` attribute (`aria-disabled="true"` on anchors).
5. **Loading:** `disabled` attribute + DOM-injected `<span class="spinner-border spinner-border-sm me-2" role="status" aria-hidden="true"></span>`.

### 6.2 Five Form Input States

Forms MUST use `.form-floating` (or `.form-label` + `.form-control` / `.form-select`) and support:

1. **Default:** `.form-control`
2. **Focus:** `.form-control.focus-ring`
3. **Error:** `.form-control.is-invalid` + `<div class="invalid-feedback">`
4. **Warning (Non-blocking):** `.form-control.border-warning` + `<div class="form-text text-warning-emphasis">`
5. **Valid:** `.form-control.is-valid` + `<div class="valid-feedback">`

Form submission groups MUST use `.hstack.gap-2.justify-content-end` (`btn-outline-secondary` Cancel + `btn-primary` Submit).

### 6.3 Multi-Step Flows & Conditional Branching

1. **Progress Bar:** Multi-step journeys MUST display `.progress` with `role="progressbar"` and updated `aria-valuenow`.
2. **Escape Hatch:** Registration/onboarding steps MUST include a secondary escape link (`Already have an account?` -> `[LoginRoute]`).
3. **Branching Logic:** Qualifying fields (e.g., `team_size`) MUST deterministically branch subsequent views (`team_size > 1` -> Invite Team step -> Workspace; `team_size === 1` -> Workspace directly).

---

## 7. Plug-and-Play HTML5 & Bootstrap 5.3+ Reference Templates

### 7.1 HTML5 Document Shell (Dual-Theme, SEO, JSON-LD & CSRF-Ready)

```html
<!DOCTYPE html>
<html lang="en" data-bs-theme="light">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="csrf-token" content="[CSRF_TOKEN]">
  <meta name="csrf-header" content="[CSRF_HEADER]">
  <title>[PageTitle]</title>
  <meta name="description" content="[MetaDescription]">
  <meta name="keywords" content="[MetaKeywords]">
  <meta name="robots" content="[RobotsTag]">
  <link rel="canonical" href="[CanonicalUrl]">
  <meta property="og:type" content="website">
  <meta property="og:title" content="[PageTitle]">
  <meta property="og:description" content="[MetaDescription]">
  <meta property="og:url" content="[CanonicalUrl]">
  <meta property="og:image" content="[MetaImage]">
  <link rel="stylesheet" href="[AssetPath]/vendor/bootstrap/css/bootstrap.min.css">
  <style nonce="[CSP_STYLE_NONCE]">
    /* Insert Section 2.2 Mandatory CSS Variable Mapping here */
  </style>
</head>
<body class="bg-body text-body min-vh-100 d-flex flex-column">

  <header class="navbar navbar-expand-lg sticky-top bg-body border-bottom">
    <nav class="container" aria-label="Primary navigation">
      <a class="navbar-brand fw-bold" href="[HomeRoute]">[BrandName]</a>
      <div class="d-flex align-items-center gap-2 ms-auto order-lg-3">
        <button id="themeToggle" type="button" class="btn btn-outline-secondary btn-sm" aria-pressed="false" aria-label="Toggle color theme">Theme</button>
        <a href="[PrimaryActionRoute]" class="btn btn-primary btn-sm px-3">[PrimaryAction]</a>
        <button class="navbar-toggler" type="button" data-bs-toggle="offcanvas" data-bs-target="#primaryOffcanvas" aria-controls="primaryOffcanvas" aria-label="Open menu">
          <span class="navbar-toggler-icon"></span>
        </button>
      </div>
      <div class="offcanvas offcanvas-end" tabindex="-1" id="primaryOffcanvas" aria-labelledby="primaryOffcanvasLabel">
        <div class="offcanvas-header border-bottom">
          <h2 class="h5 offcanvas-title mb-0" id="primaryOffcanvasLabel">[BrandName]</h2>
          <button type="button" class="btn-close" data-bs-dismiss="offcanvas" aria-label="Close"></button>
        </div>
        <div class="offcanvas-body">
          <ul class="navbar-nav me-auto gap-lg-2">
            <li class="nav-item"><a class="nav-link" href="[RouteA]">[NavItemA]</a></li>
            <li class="nav-item"><a class="nav-link" href="[RouteB]">[NavItemB]</a></li>
          </ul>
        </div>
      </div>
    </nav>
  </header>

  <main class="flex-grow-1">
    <!-- Flash Alert Slot -->
    <div class="container pt-4">
      <div class="alert alert-success alert-dismissible fade show d-none" role="alert" id="statusAlert">
        <span id="statusAlertMessage"></span>
        <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
      </div>
    </div>
    <!-- Section Content -->
  </main>

  <footer class="bg-body-secondary border-top py-4 mt-auto">
    <div class="container d-flex flex-column flex-sm-row justify-content-between align-items-center gap-2 small text-body-secondary">
      <span>&copy; [Year] [BrandName]</span>
      <span>[FooterLinks]</span>
    </div>
  </footer>

  <!-- Global Feedback Toast Container -->
  <div class="toast-container position-fixed bottom-0 end-0 p-3">
    <div id="feedbackToast" class="toast shadow-lg border" role="status" aria-live="polite" aria-atomic="true">
      <div class="toast-header">
        <strong class="me-auto text-body">[BrandName]</strong>
        <button type="button" class="btn-close" data-bs-dismiss="toast" aria-label="Close"></button>
      </div>
      <div id="feedbackToastBody" class="toast-body"></div>
    </div>
  </div>

  <script src="[AssetPath]/vendor/bootstrap/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

### 7.2 4-Zone Hero Pattern

```html
<section class="py-5 border-bottom">
  <div class="container py-lg-4">
    <div class="row align-items-center g-5">
      <div class="col-12 col-lg-6">
        <span class="badge bg-primary-subtle text-primary-emphasis border border-primary-subtle rounded-pill mb-3">[EyebrowTag]</span>
        <h1 class="display-5 fw-bold lh-1 mb-2">[SevenWordsOrFewerHeadline]</h1>
        <p class="lead fs-6 text-body-secondary mb-0">[FourteenWordsOrFewerSupportingSummary]</p>
        <div class="hstack gap-2 mt-4">
          <a href="[PrimaryRoute]" class="btn btn-primary px-4 py-2">[PrimaryAction]</a>
          <a href="[SecondaryRoute]" class="btn btn-outline-secondary px-4 py-2">[SecondaryAction]</a>
        </div>
        <div class="row g-3 mt-5 pt-4 border-top">
          <div class="col-4">
            <div class="small fw-semibold text-body">[MetricA]</div>
            <div class="small text-body-secondary">[MetricLabelA]</div>
          </div>
          <div class="col-4 border-start">
            <div class="small fw-semibold text-body">[MetricB]</div>
            <div class="small text-body-secondary">[MetricLabelB]</div>
          </div>
          <div class="col-4 border-start">
            <div class="small fw-semibold text-body">[MetricC]</div>
            <div class="small text-body-secondary">[MetricLabelC]</div>
          </div>
        </div>
      </div>
      <div class="col-12 col-lg-6">
        <div class="card overflow-hidden border shadow-sm">
          <img src="[HeroImagePath]" class="card-img object-fit-cover" alt="[HeroImageAlt]">
          <div class="card-img-overlay d-flex flex-column p-0">
            <div class="mt-auto p-3 p-md-4 bg-dark bg-opacity-75 text-white">
              <div class="fw-semibold text-truncate">[OverlayTitle]</div>
              <div class="small opacity-75 text-truncate">[OverlaySubtitle]</div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</section>
```

### 7.3 Entity Card, Form & Pagination Patterns

```html
<!-- Entity Card -->
<article class="card bg-body-tertiary border shadow-sm h-100">
  <div class="card-body vstack gap-3">
    <div class="hstack gap-3 align-items-center">
      <div class="rounded-3 bg-primary-subtle text-primary-emphasis p-2 fw-bold">[Icon]</div>
      <div class="vstack gap-1 overflow-hidden">
        <h2 class="h6 fw-semibold mb-0 text-truncate">[EntityTitle]</h2>
        <span class="small text-body-secondary text-truncate">[EntityTimestamp]</span>
      </div>
      <span class="ms-auto fw-bold text-primary">[MetricValue]</span>
    </div>
    <div class="border-start ps-3 vstack gap-2 small">
      <div class="text-body">[OriginLabel]</div>
      <div class="text-body-secondary">[DestinationLabel]</div>
    </div>
    <div class="hstack gap-2 justify-content-end mt-auto pt-2">
      <button type="button" class="btn btn-outline-secondary btn-sm" data-copy-text="[CopyableValue]">Copy</button>
      <a href="[EntityDetailRoute]" class="btn btn-primary btn-sm">[PrimaryAction]</a>
    </div>
  </div>
</article>

<!-- Validated Form Pattern -->
<form action="[SubmitRoute]" method="post" class="vstack gap-3" novalidate>
  <div class="form-floating">
    <input type="text" name="domain_id" id="domainId" class="form-control focus-ring is-invalid" placeholder="[Placeholder]" required>
    <label for="domainId">[FieldLabel]</label>
    <div class="invalid-feedback">[ValidationErrorMessage]</div>
  </div>
  <div class="hstack gap-2 justify-content-end mt-2">
    <a href="[CancelRoute]" class="btn btn-outline-secondary">Cancel</a>
    <button type="submit" class="btn btn-primary" data-loading-btn>[SubmitAction]</button>
  </div>
</form>

<!-- Pagination Pattern -->
<nav aria-label="Page navigation">
  <ul class="pagination pagination-sm justify-content-center flex-wrap mb-0">
    <li class="page-item disabled"><span class="page-link" aria-hidden="true">&laquo;</span></li>
    <li class="page-item active" aria-current="page"><a class="page-link" href="[Page1Route]">1</a></li>
    <li class="page-item"><a class="page-link" href="[Page2Route]">2</a></li>
    <li class="page-item"><a class="page-link" href="[NextRoute]" aria-label="Next">&raquo;</a></li>
  </ul>
</nav>
```

---

## 8. Bootstrap 5.3+ JS Controller (XSS-Safe, CSRF-Aware, Zero Fallbacks)

```javascript
'use strict';

window.getCSRFToken = () => {
  const meta = document.querySelector('meta[name="csrf-token"]');
  if (!(meta instanceof HTMLMetaElement)) {
    throw new Error('Required meta[name="csrf-token"] element is missing.');
  }
  return meta.content;
};

window.updateCSRFToken = (token) => {
  if (typeof token !== 'string' || token === '') {
    throw new Error('CSRF token update requires a non-empty string.');
  }
  const meta = document.querySelector('meta[name="csrf-token"]');
  if (meta instanceof HTMLMetaElement) {
    meta.content = token;
  }
  document.querySelectorAll('input[name="csrf_token"]').forEach((input) => {
    if (input instanceof HTMLInputElement) {
      input.value = token;
    }
  });
};

document.addEventListener('DOMContentLoaded', () => {
  document.querySelectorAll('[data-bs-toggle="tooltip"]').forEach((el) => {
    new bootstrap.Tooltip(el);
  });

  const themeBtn = document.getElementById('themeToggle');
  if (themeBtn instanceof HTMLButtonElement) {
    themeBtn.addEventListener('click', () => {
      const root = document.documentElement;
      const next = root.getAttribute('data-bs-theme') === 'dark' ? 'light' : 'dark';
      root.setAttribute('data-bs-theme', next);
      themeBtn.setAttribute('aria-pressed', next === 'dark' ? 'true' : 'false');
    });
  }

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

      const originalLabel = btn.textContent ?? '';
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
        console.error('Clipboard operation failed:', err);
      } finally {
        window.setTimeout(() => {
          btn.classList.remove('btn-success', 'btn-danger');
          btn.classList.add('btn-outline-secondary');
          btn.textContent = originalLabel;
        }, 1800);
      }
    });
  });

  document.querySelectorAll('[data-loading-btn]').forEach((btn) => {
    if (!(btn instanceof HTMLButtonElement)) {
      return;
    }
    btn.addEventListener('click', () => {
      if (btn.disabled) {
        return;
      }
      const spinner = document.createElement('span');
      spinner.className = 'spinner-border spinner-border-sm me-2';
      spinner.setAttribute('role', 'status');
      spinner.setAttribute('aria-hidden', 'true');

      btn.dataset.originalLabel = btn.textContent ?? 'Submit';
      btn.disabled = true;
      btn.replaceChildren(spinner, document.createTextNode('Processing...'));
    });
  });
});
```

### Standard Async Envelope Contract

When integrating client-side `fetch()` calls, UI controllers MUST expect and handle:
`{ "status": "success" | "error" | "validation_error", "message": string, "result": mixed, "errors": object, "csrf_token": string }`, calling `window.updateCSRFToken(payload.csrf_token)` immediately upon response receipt.

---

## 9. Pre-Delivery Verification Gate

Before marking any UI task complete, verify every item:

1. **HTML5 & Template Discipline:** Semantic `<header>`, `<nav>`, `<main>`, `<section>`, `<article>`, `<footer>` landmarks present; conditional view nesting depth `<= 1`.
2. **Bootstrap 5.3+ Purity:** Zero custom component CSS classes, zero inline `style=""`, zero inline `on*` JS handlers, zero external CDNs, zero hardcoded hex colors in HTML.
3. **Brand Fidelity:** Strictly White (`#FFFFFF`/`#F8F9FA`), Black (`#0A0A0D`/`#141419`), Blue (`#0057FF`/`#3B82F6`), with semantic hues restricted strictly to status/validation.
4. **Dual-Theme Parity:** Both `data-bs-theme="light"` and `data-bs-theme="dark"` render legible contrast, muted borders, stepped dark surfaces (`bg-body-tertiary`), and desaturated subtle badges.
5. **Hierarchy & Edge Cases:** `<= 6` font sizes; 4pt proximity spacing (`gap-2` coupled, `mt-4` before CTAs); `.text-truncate` on variable titles; `.bg-dark.bg-opacity-75` scrims behind overlay text/icons.
6. **States & XSS Hygiene:** All 5 button states and 5 input states present; DOM writes exclusively via `textContent` / `replaceChildren()`; zero silent `catch` blocks.
