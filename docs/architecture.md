# Architecture: Glassmorphism CSS in Authentik's Shadow DOM

> How this theme works inside Authentik's web component tree, why standard CSS fails,
> and the three strategies that make glassmorphism possible across nested shadow boundaries.

**Applies to**: Authentik 2026.x (tested on 2026.2.1)

---

## Table of Contents

1. [Authentik's Frontend Architecture](#1-authentiks-frontend-architecture)
2. [Shadow DOM Structure: Login Page](#2-shadow-dom-structure-login-page)
3. [Shadow DOM Structure: Library Page](#3-shadow-dom-structure-library-page)
4. [Why Standard CSS Selectors Fail](#4-why-standard-css-selectors-fail)
5. [Three CSS Strategies](#5-three-css-strategies)
6. [Admin Isolation](#6-admin-isolation)
7. [SSO Button Specificity](#7-sso-button-specificity)
8. [CSS Variable Cascade](#8-css-variable-cascade)

---

## 1. Authentik's Frontend Architecture

Authentik's user-facing UI is built entirely with **web components**. Every visual element
you see on the login page, the library page, or the admin panel is rendered inside a
shadow root, not in the light DOM.

### Technology Stack

| Layer            | Technology                    | Role                                       |
|------------------|-------------------------------|---------------------------------------------|
| Web Components   | [Lit](https://lit.dev/) 2.x   | Base class for all custom elements           |
| Design System    | PatternFly 4                  | CSS class names (`.pf-c-*`), layout patterns |
| Shadow DOM       | Open shadow roots             | Style encapsulation per component            |
| Base Class       | `AKElement`                   | Authentik's extension of `LitElement`        |
| Style Injection  | `adoptedStyleSheets`          | How custom CSS reaches every shadow root     |

### How Custom CSS Gets In: `AKElement` and Adopted Stylesheets

When you paste CSS into **System > Brands > Custom CSS**, Authentik does not inject a
`<style>` tag into `<head>`. Instead, the `AKElement` base class (which every Authentik
component extends) adopts that CSS into each component's shadow root via the
[`adoptedStyleSheets`](https://developer.mozilla.org/en-US/docs/Web/API/ShadowRoot/adoptedStyleSheets)
API.

The mechanism works like this:

```
Brand settings (custom CSS string)
  --> AKElement base class reads it at startup
    --> Every component that extends AKElement
      --> this.shadowRoot.adoptedStyleSheets = [
            ...patternFlyStyleSheets,
            ...authentikStyleSheets,
            customBrandStyleSheet    // <-- your CSS here
          ]
```

This has a critical consequence: **your custom CSS is adopted into every shadow root in
the application**. It runs inside `ak-flow-executor`'s shadow root, inside
`ak-stage-identification`'s shadow root, inside `ak-library-impl`'s shadow root, and
also inside `ak-interface-admin`'s shadow root. You must scope your selectors carefully
or you will break the admin interface.

### PatternFly 4 Class Naming

Authentik uses PatternFly 4's BEM-style naming convention:

- **Components**: `.pf-c-login`, `.pf-c-card`, `.pf-c-button`
- **Modifiers**: `.pf-m-primary`, `.pf-m-block`, `.pf-m-plain`
- **Sub-elements**: `.pf-c-login__main`, `.pf-c-login__header`, `.pf-c-login__footer`
- **CSS variables**: `--pf-c-login__main--BackgroundColor`, `--pf-c-card--BoxShadow`

PatternFly components define most visual properties through CSS custom properties. This
is key to the glassmorphism strategy: you can override those variables on `:root, :host`
and they cascade through every shadow boundary.

---

## 2. Shadow DOM Structure: Login Page

The login page (flow interface) uses a nested shadow DOM tree with three levels. Your
custom CSS is adopted into every level, but selectors can only match elements within
the shadow root where the CSS is evaluated.

```
Document
│
├─ <body>                                    ─── light DOM (not reachable from shadow)
│
└─ <ak-flow-executor>                        ─── custom element
     │  class="pf-c-login"                        HOST class (invisible from inside)
     │
     └─ #shadow-root (open)                  ─── Level 1: flow executor shadow
         │  [adoptedStyleSheets includes your custom CSS]
         │
         ├─ <header class="pf-c-login__header">
         │   └─ <img class="pf-c-brand">          Logo
         │
         ├─ <main class="pf-c-login__main">  ─── THE glass container target
         │   │
         │   └─ <ak-stage-identification>     ─── custom element (stage)
         │        │
         │        └─ #shadow-root (open)     ─── Level 2: stage shadow
         │            │  [adoptedStyleSheets includes your custom CSS]
         │            │
         │            ├─ <ak-flow-card>       ─── custom element
         │            │    └─ #shadow-root   ─── Level 3: card shadow
         │            │        │  [adoptedStyleSheets includes your custom CSS]
         │            │        ├─ .pf-c-login__main-header  (title)
         │            │        ├─ .pf-c-login__main-body    (slot: form)
         │            │        └─ .pf-c-login__main-footer
         │            │
         │            ├─ <input name="uidField">          (.pf-c-form-control)
         │            ├─ <input name="password">          (.pf-c-form-control)
         │            ├─ <input type="checkbox">          (.pf-c-switch__input)
         │            ├─ <button type="submit">           "Log in"
         │            ├─ <ak-divider>                     "Or"
         │            ├─ <a name="passwordless">          "Use a security key"
         │            ├─ <button name="source-discord">   "Continue with Discord"
         │            └─ <button name="source-plex">      "Continue with Plex"
         │
         └─ <footer class="pf-c-login__footer">
              └─ <ak-brand-links>
                   └─ .pf-c-list.pf-m-inline
```

### Key Observations

- **Level 1** (flow executor shadow) is where `.pf-c-login__main` lives. That is the
  element you apply the glass surface, `backdrop-filter`, and `border-radius` to.
- **Level 2** (stage shadow) is where inputs, buttons, and SSO source buttons live.
  All form controls are styled here with direct selectors or `input[name="..."]`.
- **Level 3** (card shadow) contains `.pf-c-login__main-header`, `__main-body`, and
  `__main-footer`. These mostly inherit transparency from CSS variable overrides.
- The **host class** `.pf-c-login` is on `<ak-flow-executor>` itself. It is not
  visible from inside any of the three shadow roots.

---

## 3. Shadow DOM Structure: Library Page

The library page (user interface) has a flatter structure but introduces `::part()`
as the primary styling mechanism for application cards.

```
Document
│
└─ <ak-interface-user>                       ─── custom element
     │
     └─ #shadow-root (open)                  ─── Level 1: user interface shadow
         │  [adoptedStyleSheets includes your custom CSS]
         │
         ├─ <header class="pf-c-page__header">   ─── glass header bar
         │   ├─ <img class="pf-c-brand">
         │   └─ <div class="pf-c-page__header-tools">
         │        ├─ <ak-nav-buttons>
         │        │    └─ #shadow-root
         │        │        ├─ .pf-c-button.pf-m-plain  (icons)
         │        │        └─ .pf-c-avatar
         │        └─ (other tool buttons)
         │
         ├─ <ak-library-impl>                ─── custom element
         │    │
         │    └─ #shadow-root (open)         ─── Level 2: library shadow
         │        │  [adoptedStyleSheets includes your custom CSS]
         │        │
         │        ├─ <header class="pf-c-page__header pf-c-content">
         │        │    └─ <h1 class="pf-c-page__title">  "My applications"
         │        │
         │        ├─ <input part="search-input">         Search bar
         │        │
         │        ├─ <legend part="app-group-header">    Group label
         │        │
         │        ├─ <div part="card">                   Application card
         │        │    ├─ <div part="card-title">
         │        │    ├─ <div part="card-header-actions">
         │        │    │    └─ .pf-c-dropdown
         │        │    └─ (app icon, description)
         │        │
         │        └─ (more cards...)
         │
         ├─ <ak-notification-drawer>
         │    └─ #shadow-root
         │        └─ .pf-c-notification-drawer
         │
         └─ <ak-api-drawer>
              └─ #shadow-root
                  └─ .pf-c-drawer__panel
```

### Key Observations

- `ak-library-impl` exposes **CSS parts** on its internal elements: `card`,
  `card-title`, `search-input`, `app-group-header`, `card-header-actions`.
- You style these from the *parent* shadow root (Level 1) using the `::part()` pseudo-element.
- The page header `.pf-c-page__header` is in Level 1 and can be styled with direct selectors.
- Inside `ak-library-impl`, there is a second `.pf-c-page__header` with an additional
  `.pf-c-content` class. This is the "My applications" bar and must be differentiated
  from the main header (use `.pf-c-page__header.pf-c-content` or `.pf-c-page__header:not(.pf-c-content)`).

---

## 4. Why Standard CSS Selectors Fail

If you search for "authentik custom CSS" examples from pre-2025 or generic PatternFly
theming guides, you will find selectors that look reasonable but do not work in
Authentik 2026.x. Here is a concrete breakdown of each failure mode.

### 4.1 The Host Class Is Invisible

```css
/* BROKEN: .pf-c-login is on the host element */
.pf-c-login .pf-c-login__main {
  background: transparent;
}
```

`.pf-c-login` is a class on `<ak-flow-executor>`, which is the **host element**. Inside
the shadow root, the host element is not part of the DOM tree. You cannot select it
with regular class selectors. The descendant combinator `.pf-c-login .pf-c-login__main`
will never match because `.pf-c-login` does not exist in the shadow root's tree.

**Fix**: Drop the `.pf-c-login` prefix entirely. Write `.pf-c-login__main` as a direct selector.

### 4.2 Removed Elements

```css
/* BROKEN: .pf-c-login__container does not exist in 2026.x */
.pf-c-login__container {
  background: transparent;
}
```

Authentik 2026.x removed the `.pf-c-login__container` wrapper. The structure goes
directly from `<main class="pf-c-login__main">` to the stage component. CSS targeting
this element matches nothing.

### 4.3 Cross-Shadow Selectors

```css
/* BROKEN: body is in the light DOM, not accessible from shadow */
body:has(.pf-c-login) .some-element {
  background: transparent;
}
```

The `body` element is in the light DOM. Shadow DOM creates a boundary: selectors inside
a shadow root cannot reach outside of it, and selectors outside cannot reach in.
`:has()` does not cross shadow boundaries in either direction.

### 4.4 Scoped Selectors Across Shadows

```css
/* BROKEN: .pf-c-login__main is in Level 1, inputs are in Level 2 */
.pf-c-login__main input.pf-c-form-control {
  background: glass;
}
```

`.pf-c-login__main` lives in the flow executor's shadow root (Level 1). The inputs live
in the stage's shadow root (Level 2). A descendant selector cannot span two different
shadow roots. Each shadow root evaluates the CSS independently against its own DOM tree.

### Summary Table

| Selector Pattern | Why It Fails | Fix |
|---|---|---|
| `.pf-c-login .pf-c-login__main` | Host class invisible from inside shadow | `.pf-c-login__main` (direct) |
| `.pf-c-login__container` | Element removed in 2026.x | `.pf-c-login__main` |
| `body:has(.pf-c-login) ...` | `body` is outside shadow boundary | Use `:host()` for context |
| `.pf-c-login__main input` | Elements in different shadow roots | Target inputs with `input[name="..."]` in Level 2 |
| `.pf-c-card` in login context | No `.pf-c-card` in flow components | Use `.pf-c-login__main-body` in Level 3 |

---

## 5. Three CSS Strategies

The glassmorphism theme uses three complementary strategies to style elements across
the shadow DOM tree. Each strategy addresses a different type of boundary.

### 5.1 CSS Variables on `:root, :host`

**Purpose**: Override PatternFly design tokens and define glass layer values that
cascade through every shadow boundary.

**How it works**: CSS custom properties (variables) are the one thing that *does*
cascade through shadow DOM boundaries. When you set `--pf-c-login__main--BackgroundColor`
on `:root, :host`, that value is available to every component in the tree, regardless
of how deep the shadow nesting goes.

```css
:root, :host {
  /* Override PatternFly's opaque background */
  --pf-c-login__main--BackgroundColor: transparent !important;
  --pf-c-login--BackgroundColor: transparent !important;

  /* Define glass layer system */
  --glass-l1-bg: rgba(15, 23, 52, 0.55);
  --glass-l1-blur: blur(20px) saturate(1.5);
  --glass-l2-bg: rgba(20, 30, 70, 0.45);
  --glass-l3-bg: rgba(30, 45, 90, 0.30);

  /* Override PF card appearance */
  --pf-c-card--BackgroundColor: var(--glass-l2-bg) !important;
  --pf-c-card--BoxShadow: var(--glass-shadow) !important;
}
```

**Why `:root, :host`?** The CSS is evaluated in multiple contexts:

- In the *document* context (if it were in the light DOM), `:root` matches `<html>`.
- In each *shadow root* context, `:host` matches the host element of that shadow.
- By combining both, the variables are defined in every possible evaluation context.

**What this achieves**: PatternFly components read their visual properties from
`--pf-c-*` variables. By overriding those variables at the highest level, you change
the defaults for every component without writing a single direct selector. The login
main body becomes transparent, cards get glass backgrounds, and the change reaches
Level 3 (the flow card shadow) without you needing to target anything there.

### 5.2 Direct Selectors Without Host Prefix

**Purpose**: Apply `backdrop-filter`, `border`, `box-shadow`, and other properties
that have no PatternFly variable to override.

**How it works**: Because your custom CSS is adopted into every shadow root, a direct
selector like `.pf-c-login__main` will match that element in whichever shadow root
contains it. You do not need (and cannot use) the host prefix `.pf-c-login`.

```css
/* This runs inside ak-flow-executor's shadow root */
/* .pf-c-login__main exists here, so it matches */
.pf-c-login__main {
  background: var(--glass-l1-bg) !important;
  backdrop-filter: var(--glass-l1-blur) !important;
  border: var(--glass-border) !important;
  border-radius: var(--glass-radius) !important;
  box-shadow: var(--glass-shadow-container) !important;
}

/* This runs inside ak-stage-identification's shadow root */
/* input[name="uidField"] exists here, so it matches */
input[name="uidField"],
input[name="password"] {
  background: var(--glass-l3-bg) !important;
  backdrop-filter: var(--glass-l3-blur) !important;
  border: 1px solid rgba(var(--theme-border), 0.18) !important;
  border-radius: var(--glass-radius-sm) !important;
}
```

**Scoping with `:host()`**: To prevent login styles from affecting admin components,
you can scope selectors to specific host tags:

```css
:host(ak-stage-identification) input.pf-c-form-control,
:host(ak-stage-password) input.pf-c-form-control {
  /* Glass input styles - only applied when the host is a login stage */
}
```

`:host(ak-stage-identification)` matches when the CSS is evaluated inside a shadow root
whose host element is `<ak-stage-identification>`. It will not match inside
`<ak-interface-admin>` or any other component.

### 5.3 `::part()` for Library Cards

**Purpose**: Style internal elements of `ak-library-impl` from the parent shadow root.

**How it works**: `ak-library-impl` exposes certain elements with the `part` attribute.
The parent component (`ak-interface-user`) can style those parts using the `::part()`
pseudo-element. Since your custom CSS is adopted into `ak-interface-user`'s shadow root,
`::part()` selectors work there.

```css
/* Evaluated in ak-interface-user's shadow root (Level 1) */
/* Reaches into ak-library-impl's shadow root (Level 2) via parts */
ak-library-impl::part(card) {
  border-radius: 20px !important;
  background: rgba(var(--theme-tint), 0.30) !important;
  backdrop-filter: blur(14px) saturate(1.6) !important;
  border: 1px solid rgba(255, 255, 255, 0.12) !important;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.25) !important;
}

ak-library-impl::part(search-input) {
  background: var(--glass-l3-bg) !important;
  border-radius: 14px !important;
  backdrop-filter: blur(14px) !important;
}

ak-library-impl::part(card-title) {
  color: var(--glass-text) !important;
  text-shadow: 0 2px 8px rgba(0, 0, 0, 0.4) !important;
}

ak-library-impl::part(app-group-header) {
  color: rgba(255, 255, 255, 0.90) !important;
  font-weight: 600 !important;
}
```

**Available parts** (as of 2026.2.x):

| Part Name              | Element          | Description                      |
|------------------------|------------------|----------------------------------|
| `search-input`         | `<input>`        | Application search bar           |
| `card`                 | `<div>`          | Application card wrapper         |
| `card-title`           | `<div>`          | Card title text                  |
| `card-header-actions`  | `<div>`          | Dropdown menu trigger area       |
| `app-group-header`     | `<legend>`       | Group section label              |

**Why not use direct selectors for library cards?** You *could* use `.pf-c-card` inside
`ak-library-impl`'s shadow root, and it would match. But `::part()` is more explicit,
more future-proof, and matches Authentik's intended API for external styling.

### Strategy Decision Diagram

```
Need to style an element?
│
├─ Does a PF CSS variable control this property?
│   └─ YES → Override the variable on :root, :host        [Strategy 1]
│
├─ Is the element in the same shadow root as your CSS?
│   └─ YES → Use a direct selector (no host prefix)       [Strategy 2]
│
├─ Does the element have a part="..." attribute?
│   └─ YES → Use ::part() from the parent shadow           [Strategy 3]
│
└─ None of the above?
    └─ The element is unreachable. File a feature request
       for a part attribute on the Authentik repo.
```

---

## 6. Admin Isolation

Because your custom CSS is adopted into **every** shadow root (including admin), you
must explicitly neutralize glass effects in the admin interface. The theme uses two
complementary techniques.

### 6.1 `:host(ak-interface-admin)` Reset

The admin interface's root component is `<ak-interface-admin>`. By targeting `:host(ak-interface-admin)`,
you reset all glass variables back to PatternFly defaults:

```css
:host(ak-interface-admin) {
  /* Reset glass surfaces to opaque PF defaults */
  --glass-l1-bg: var(--pf-global--BackgroundColor--100, #151515) !important;
  --glass-l2-bg: var(--pf-global--BackgroundColor--200, #1a1a1a) !important;
  --glass-l3-bg: var(--pf-global--BackgroundColor--200, #1a1a1a) !important;

  /* Disable all blur effects */
  --glass-l1-blur: none !important;
  --glass-l2-blur: none !important;
  --glass-l3-blur: none !important;

  /* Disable all decorative properties */
  --glass-border: none !important;
  --glass-border-hover: none !important;
  --glass-shadow: none !important;
  --glass-shadow-hover: none !important;
  --glass-shadow-container: none !important;
  --glass-radius: 0px !important;
  --glass-radius-md: 0px !important;
  --glass-radius-sm: 0px !important;
  --glass-inset-glow: none !important;
}
```

This works because all glass layer selectors (`.pf-c-login__main`, inputs, cards) read
their visual properties from `--glass-*` variables. When those variables are reset to
flat values, the glass effect disappears without needing to write override rules for
every individual selector.

### 6.2 `:host(ak-stage-*)` Scoping for Form Controls

For input fields, the theme does not use a blanket `.pf-c-form-control` selector (which
would hit every input in admin forms). Instead, it scopes to login stage components:

```css
:host(ak-stage-identification) input.pf-c-form-control,
:host(ak-stage-password) input.pf-c-form-control,
:host(ak-stage-consent) input.pf-c-form-control,
:host(ak-stage-email) input.pf-c-form-control,
:host(ak-stage-prompt) input.pf-c-form-control,
:host(ak-stage-authenticator-validate) input.pf-c-form-control {
  /* Glass input styles */
}
```

The CSS also uses `input[name="uidField"]` and `input[name="password"]` as
high-confidence selectors. These named inputs only exist in login stages, so there is
zero risk of affecting admin forms.

### 6.3 Explicit Admin Component Resets

As a safety net, specific admin component classes are reset:

```css
:host(ak-interface-admin) .pf-c-page__header {
  backdrop-filter: none !important;
  background: var(--pf-c-page__header--BackgroundColor) !important;
}

:host(ak-interface-admin) .pf-c-card {
  backdrop-filter: none !important;
  border: none !important;
  border-radius: 0 !important;
}

:host(ak-interface-admin) .pf-c-form-control {
  backdrop-filter: none !important;
  background: var(--pf-c-form-control--BackgroundColor) !important;
}
```

### Why This Layered Approach?

A single `:host(ak-interface-admin)` variable reset *should* be sufficient since all
glass selectors use variables. The explicit property resets are a defensive layer for
two scenarios:

1. A selector uses a hardcoded value instead of a variable (e.g., `backdrop-filter: blur(20px)` without a variable).
2. A future CSS change introduces a new glass selector that forgets to use the variable system.

---

## 7. SSO Button Specificity

SSO provider buttons (Discord, Plex, etc.) require special attention because
PatternFly's default styles have high specificity, and the buttons use background
gradients via `background-image`.

### The Problem

SSO source buttons in Authentik are rendered as:

```html
<button class="pf-c-button pf-m-primary pf-m-block" name="source-discord">
  Continue with Discord
</button>
```

PatternFly styles `.pf-c-button.pf-m-primary` with a `background-image` gradient.
The combined specificity of `.pf-c-button.pf-m-primary.pf-m-block` is **0,3,0**
(three class selectors).

If you try to override with a low-specificity selector:

```css
/* Specificity: 0,1,1 (one class + one attribute) — TOO WEAK */
button[name="source-discord"] {
  background: rgba(88, 101, 242, 0.25);
}
```

This will lose to PatternFly's 0,3,0 selector, and the button keeps its default
PatternFly blue gradient.

### The Solution

Match PatternFly's specificity and exceed it by adding the attribute selector:

```css
/* Specificity: 0,4,0 (three classes + one attribute) — WINS */
.pf-c-button.pf-m-primary.pf-m-block[name="source-discord"] {
  background: rgba(88, 101, 242, 0.25) !important;
  border: 1px solid rgba(88, 101, 242, 0.40) !important;
  color: rgba(255, 255, 255, 0.95) !important;
  background-image: none !important;  /* Kill PF gradient */
}
```

### Specificity Comparison

| Selector | Specificity | Wins? |
|---|---|---|
| `button[name="source-discord"]` | 0,1,1 | No |
| `.pf-c-button.pf-m-primary.pf-m-block` | 0,3,0 | (PF default) |
| `.pf-c-button.pf-m-primary.pf-m-block[name="source-discord"]` | 0,4,0 | Yes |

### Why `background-image: none`?

PatternFly's `.pf-c-button.pf-m-primary` applies a gradient via `background-image`.
The CSS `background` shorthand *can* reset `background-image`, but when both the
shorthand and longhand are set at the same specificity, the longhand wins in some
browsers. Adding an explicit `background-image: none` ensures the PF gradient is
killed in all browsers.

### Security Key Exception

The "Use a security key" link is rendered as an `<a>` tag, not a `<button>`:

```html
<a class="pf-c-button" name="passwordless">Use a security key</a>
```

Its specificity game is different because it does not have `.pf-m-primary.pf-m-block`.
The selector `a[name="passwordless"].pf-c-button` is sufficient.

---

## 8. CSS Variable Cascade

The variable cascade is the foundation of the glassmorphism effect. Understanding
how a single variable declaration on `:root, :host` reaches an element three shadow
roots deep is essential for troubleshooting.

### The Critical Variable

```css
:root, :host {
  --pf-c-login__main--BackgroundColor: transparent !important;
}
```

This single line is what makes the entire login glass effect possible. Here is what
happens step by step:

### Cascade Path

```
:root, :host
  declares --pf-c-login__main--BackgroundColor: transparent
    │
    ▼
ak-flow-executor's shadow root
  :host evaluates → variable is set on the host
    │
    ▼
.pf-c-login__main  (Level 1)
  PF stylesheet says:
    background-color: var(--pf-c-login__main--BackgroundColor)
  Resolved value: transparent
    │
    ▼
ak-stage-identification's shadow root
  :host evaluates → variable inherited from parent
    │
    ▼
ak-flow-card's shadow root (Level 3)
  .pf-c-login__main-body
    PF stylesheet says:
      background-color: var(--pf-c-login__main-body--BackgroundColor,
                            var(--pf-c-login__main--BackgroundColor))
    Resolved value: transparent (falls through to parent variable)
```

### Why Transparency Matters for Glassmorphism

`backdrop-filter` only works when the element's own background is partially or fully
transparent. If PatternFly sets `.pf-c-login__main` to an opaque `#151515`, the
`backdrop-filter: blur(20px)` has nothing to show through. By overriding the
background color variable to `transparent`, you allow the theme's semi-transparent
`rgba()` background and `backdrop-filter` to combine and create the glass effect.

### Glass Layer System

The theme defines a four-layer glass hierarchy. Each layer has increasing transparency
and decreasing blur, creating visual depth:

```
Layer   Usage                  Background              Blur               Border
──────  ─────────────────────  ──────────────────────  ─────────────────  ──────────────────
L1      Login container,       rgba(tint, 0.55)        blur(20px)         rgba(border, 0.15)
        Library header                                  saturate(1.5)

L2      Library cards,         rgba(tint-l2, 0.45)     blur(16px)         rgba(border, 0.12)
        PF cards                                        saturate(1.4)

L3      Inputs, secondary      rgba(tint-l3, 0.30)     blur(12px)         rgba(border, 0.18)
        buttons, search bar                             saturate(1.2)

L4      Primary CTA button     gradient 135deg          blur(10px)         rgba(border, 0.25)
        (Log in)               accent colors             saturate(1.3)
```

Each layer's values are derived from the 11 theme color variables. Change those
variables and every layer updates consistently.

### Variable Dependency Graph

```
--theme-tint ──────────┬──► --glass-l1-bg
                       └──► (used directly in ::part(card))

--theme-tint-l2 ───────┬──► --glass-l2-bg
                       └──► (used in card hover, ::part(card):hover)

--theme-tint-l3 ───────┬──► --glass-l3-bg
                       └──► (used in input focus background)

--theme-tint-dark ─────┬──► header background
                       └──► drawer/dropdown backgrounds

--theme-accent ────────┬──► --glass-accent
                       ├──► --glass-accent-gradient
                       ├──► focus ring glow
                       └──► primary button box-shadow

--theme-accent-dark ───┴──► gradient end color, hover states

--theme-border ────────┬──► --glass-border
                       ├──► --glass-border-hover
                       └──► input borders, card borders

--theme-border-bright ─┬──► --glass-border-focus
                       └──► hover border states

--theme-link ──────────┴──► --glass-text-link, footer links

--theme-text-tint ─────┴──► --glass-text-secondary, labels

--theme-text-muted-tint ──► --glass-text-muted, placeholders
```

---

## Appendix: Verifying the Theme

You can confirm that the CSS is working correctly by inspecting computed styles from
the browser console. These scripts traverse the shadow DOM to reach the target elements.

### Login Page

```javascript
// Run on: auth.example.com/if/flow/default-authentication-flow/
const fe = document.querySelector('ak-flow-executor');
const sr = fe.shadowRoot;
const main = sr.querySelector('.pf-c-login__main');

console.log('L1 bg:', getComputedStyle(main).backgroundColor);
// Expected: rgba(15, 23, 52, 0.55)

console.log('L1 blur:', getComputedStyle(main).backdropFilter);
// Expected: blur(20px) saturate(1.5)

console.log('L1 radius:', getComputedStyle(main).borderRadius);
// Expected: 20px

const stage = sr.querySelector('ak-stage-identification');
const ssr = stage.shadowRoot;
const uid = ssr.querySelector('input[name="uidField"]');

console.log('Input bg:', getComputedStyle(uid).backgroundColor);
// Expected: rgba(30, 45, 90, 0.3)

const discord = ssr.querySelector('button[name="source-discord"]');
if (discord) {
  console.log('Discord bg:', getComputedStyle(discord).backgroundColor);
  // Expected: rgba(88, 101, 242, 0.25)
}
```

### Library Page

```javascript
// Run on: auth.example.com/if/user/
const ui = document.querySelector('ak-interface-user');
const sr = ui.shadowRoot;
const header = sr.querySelector('.pf-c-page__header:not(.pf-c-content)');

console.log('Header bg:', getComputedStyle(header).backgroundColor);
// Expected: rgba(10, 15, 30, 0.6)

console.log('Header blur:', getComputedStyle(header).backdropFilter);
// Expected: blur(18px) saturate(1.4)
```

---

## Appendix: CSS File Section Map

The `glassmorphism.css` file is organized into 17 sections, each targeting a specific
scope in the shadow DOM tree:

| Section | Name                           | Shadow Root Target            | Strategy Used      |
|---------|--------------------------------|-------------------------------|--------------------|
| 1       | CSS Custom Properties          | All (`:root, :host`)          | Variables          |
| 2       | Login: Flow Executor (L1)      | `ak-flow-executor`            | Direct selectors   |
| 3       | Login: Stage (L2)              | `ak-stage-*`                  | Direct + `:host()` |
| 4       | SSO Colored Buttons            | `ak-stage-identification`     | High specificity   |
| 5       | Flow Card (L3)                 | `ak-flow-card`                | Direct (defensive) |
| 6       | Consent Buttons                | `ak-stage-consent`            | Direct selectors   |
| 7       | Library Cards                  | `ak-interface-user`           | `::part()`         |
| 8       | Header and Navigation          | `ak-interface-user`           | Direct selectors   |
| 9       | Drawers                        | `ak-notification-drawer` etc. | Direct selectors   |
| 10      | Dropdown Menus                 | Various                       | Direct selectors   |
| 11      | Global Text and Headings       | All                           | Direct selectors   |
| 12      | Generic Buttons                | All                           | Direct selectors   |
| 13      | Scrollbar                      | All                           | Pseudo-elements    |
| 14      | Admin Hotfix                   | `ak-interface-admin`          | `:host()` reset    |
| 15      | Animations                     | All                           | `@keyframes`       |
| 16      | Accessibility                  | All                           | Media queries      |
| 17      | Responsive                     | All                           | Media queries      |
