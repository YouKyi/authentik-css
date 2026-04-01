# Adding SSO Provider Brand Colors

This guide explains how to add brand-specific colors to SSO login buttons in the glassmorphism theme. Each SSO provider gets a tinted glass button that matches its official brand color, while keeping the overall glassmorphism aesthetic.

## How SSO Buttons Work

When you add a federation source in Authentik (**Admin > Directory > Federation & Social login > Sources**), a button appears automatically on the login page. Authentik generates an HTML attribute `name="source-{slug}"` from the source name.

The slug is derived from the source name by lowercasing it and replacing spaces with hyphens:

| Source name | Generated attribute |
|---|---|
| Discord | `name="source-discord"` |
| Google Workspace | `name="source-google-workspace"` |
| Microsoft Azure AD | `name="source-microsoft-azure-ad"` |

By default, any SSO button without a dedicated color rule uses the **generic glass fallback** -- a neutral translucent style that matches the theme but does not reflect the provider's brand.

## Finding the Slug

The slug must match exactly. Open your Authentik login page in a browser and run this in the JavaScript console (F12 > Console):

```javascript
const fe = document.querySelector('ak-flow-executor');
const stage = fe.shadowRoot.querySelector('ak-stage-identification');
stage.shadowRoot.querySelectorAll('button.source-button, a.pf-c-button[name]')
  .forEach(b => console.log(b.getAttribute('name'), '->', b.textContent.trim()));
```

Example output:

```
source-discord -> Continue with Discord
source-plex -> Continue with Plex
source-google-workspace -> Continue with Google
passwordless -> Use a security key
```

The value before the `->` is the exact string you need for the CSS selector.

## CSS Template

Copy this block into `glassmorphism.css` in **Section 4 (SSO Colored Buttons)**, before the `/* Generic fallback */` comment. Replace the placeholders with your values.

```css
/* {PROVIDER_NAME} */
.pf-c-button.pf-m-primary.pf-m-block[name="source-{SLUG}"] {
  background: rgba({R}, {G}, {B}, 0.25) !important;
  border: 1px solid rgba({R}, {G}, {B}, 0.40) !important;
  color: rgba(255, 255, 255, 0.95) !important;
  background-image: none !important;
}

.pf-c-button.pf-m-primary.pf-m-block[name="source-{SLUG}"]:hover {
  background: rgba({R}, {G}, {B}, 0.40) !important;
  border-color: rgba({R}, {G}, {B}, 0.55) !important;
  box-shadow: 0 0 24px rgba({R}, {G}, {B}, 0.25) !important;
  transform: translateY(-1px) !important;
  background-image: none !important;
}
```

| Placeholder | Replace with | Example |
|---|---|---|
| `{PROVIDER_NAME}` | Provider name (comment only) | `Google` |
| `{SLUG}` | Exact slug from the discovery step | `google-workspace` |
| `{R}, {G}, {B}` | Brand color as RGB integers | `66, 133, 244` |

### Concrete Example: Google

```css
/* Google */
.pf-c-button.pf-m-primary.pf-m-block[name="source-google-workspace"] {
  background: rgba(66, 133, 244, 0.25) !important;
  border: 1px solid rgba(66, 133, 244, 0.40) !important;
  color: rgba(255, 255, 255, 0.95) !important;
  background-image: none !important;
}

.pf-c-button.pf-m-primary.pf-m-block[name="source-google-workspace"]:hover {
  background: rgba(66, 133, 244, 0.40) !important;
  border-color: rgba(66, 133, 244, 0.55) !important;
  box-shadow: 0 0 24px rgba(66, 133, 244, 0.25) !important;
  transform: translateY(-1px) !important;
  background-image: none !important;
}
```

## Why High Specificity Is Needed

The "Log in" submit button uses the selector `.pf-c-button.pf-m-primary.pf-m-block`, which has a CSS specificity of **0,3,0** (three class selectors). SSO buttons share those same three classes because Authentik renders them identically.

A simpler selector like `button[name="source-discord"]` has a specificity of only **0,1,1** (one element + one attribute), which is too weak to override the primary button styles.

By appending the attribute selector to the same class chain -- `.pf-c-button.pf-m-primary.pf-m-block[name="source-discord"]` -- the specificity becomes **0,4,0** (three classes + one attribute). This reliably beats the Log in button rule regardless of source order.

| Selector | Specificity | Wins? |
|---|---|---|
| `.pf-c-button.pf-m-primary.pf-m-block` (Log in) | 0,3,0 | -- |
| `button[name="source-discord"]` | 0,1,1 | No |
| `.pf-c-button.pf-m-primary.pf-m-block[name="source-discord"]` | 0,4,0 | Yes |

## Why `background-image: none`

The Log in button uses a `background: linear-gradient(...)` for its accent gradient effect. In CSS, `linear-gradient()` is a `background-image` value. When you set a new `background` color on the SSO button, the `background-image` from the Log in rule can still apply if it has equal or higher specificity in a shorthand conflict.

Adding `background-image: none !important` explicitly cancels the gradient and ensures only the flat `rgba()` brand color is visible. Without it, the SSO buttons display the Log in gradient on top of the brand color.

Both the normal state and the `:hover` state require `background-image: none` because the hover rule for the Log in button also sets a gradient.

## Provider Color Reference

Official brand colors for common SSO providers:

| Provider | Hex | RGB | Notes |
|---|---|---|---|
| Google | `#4285F4` | `66, 133, 244` | Google Blue |
| Microsoft | `#00A4EF` | `0, 164, 239` | Microsoft Blue |
| GitHub | `#6E40C9` | `110, 64, 201` | GitHub Purple |
| GitLab | `#FC6D26` | `252, 109, 38` | GitLab Orange |
| Apple | `#A2AAAD` | `162, 170, 173` | Apple Gray |
| Facebook | `#1877F2` | `24, 119, 242` | Facebook Blue |
| Twitter/X | `#1DA1F2` | `29, 161, 242` | Twitter Blue (legacy) |
| Slack | `#4A154B` | `74, 21, 75` | Slack Aubergine |
| Okta | `#007DC1` | `0, 125, 193` | Okta Blue |
| SAML (generic) | `#FF6600` | `255, 102, 0` | Orange convention |
| LDAP | `#5B9BD5` | `91, 155, 213` | Soft blue convention |
| OpenID Connect | `#F78C40` | `247, 140, 64` | OpenID Orange |

All RGB values use the 0-255 integer format expected by `rgba()`.

## Verification

After deploying your updated CSS, verify the button styles from the browser console on the login page:

```javascript
const fe = document.querySelector('ak-flow-executor');
const stage = fe.shadowRoot.querySelector('ak-stage-identification');
const btn = stage.shadowRoot.querySelector('button[name="source-{SLUG}"]');
console.log('background:', getComputedStyle(btn).backgroundColor);
console.log('border:', getComputedStyle(btn).borderColor);
console.log('bg-image:', getComputedStyle(btn).backgroundImage);
```

Expected results for a correctly configured button:

- `backgroundColor` should contain the provider's RGB values at 0.25 opacity (e.g., `rgba(66, 133, 244, 0.25)` for Google)
- `borderColor` should contain the provider's RGB values at 0.40 opacity
- `backgroundImage` should be `none`

If `backgroundImage` shows a `linear-gradient(...)`, the `background-image: none` declaration is missing or being overridden.

## Already Configured

The following providers are included in the theme by default. You do not need to add them manually.

| Provider | Slug / Selector | Brand Color | Hex |
|---|---|---|---|
| Discord | `source-discord` | Purple | `#5865F2` (88, 101, 242) |
| Plex | `source-plex` | Amber | `#E5A00D` (229, 160, 13) |
| Security Key | `passwordless` (uses `a[name="passwordless"]`) | Cyan | `#00B4D8` (0, 180, 216) |

Note that the Security Key button uses a slightly different selector (`a[name="passwordless"].pf-c-button`) because it is an anchor element (`<a>`), not a `<button>`, and does not carry the `.pf-m-primary.pf-m-block` classes.

## Optional: Update the Fallback Exclusion

The generic fallback rule uses `:not()` selectors to exclude providers that have dedicated colors. If you add a new provider, you can add it to the exclusion list:

```css
.source-button:not([name="source-discord"]):not([name="source-plex"]):not([name="source-{SLUG}"]),
.source-button-promoted:not([name="source-discord"]):not([name="source-plex"]):not([name="source-{SLUG}"]) {
  /* generic fallback styles */
}
```

This step is optional. The high-specificity selector (0,4,0) on your provider rule always wins over the fallback (which uses lower-specificity `.source-button` class), so the fallback will not bleed through even without updating the exclusion list.
