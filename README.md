# Authentik Glassmorphism Theme

A premium glassmorphism CSS theme for [Authentik](https://goauthentik.io/) 2026.x+ — the open-source identity provider.

Translucent glass surfaces, dynamic blur, and subtle light effects that make your SSO portal look stunning.

## Preview

| Login | Library |
|-------|---------|
| ![Login page](screenshots/login.png) | ![Library page](screenshots/library.png) |

## Features

- **Full glassmorphism** — blur, transparency, luminous borders, specular highlights
- **Login page** — glass container, styled inputs, animated entrance
- **Application library** — glass cards with hover effects, search bar, group headers
- **SSO buttons with brand colors** — Discord (purple), Plex (amber), Security Key (cyan), and a template to add more
- **Consent flows** — green/red glass buttons for grant/deny
- **Themeable** — change 11 CSS variables to match any background color
- **5 color presets** — Blue, Red, Green, Purple, Neutral
- **Admin-safe** — glass effects are scoped to user-facing pages, admin interface stays untouched
- **Accessible** — `prefers-reduced-motion`, `prefers-contrast`, `prefers-reduced-transparency` support
- **Responsive** — mobile-optimized with touch-friendly targets

## Quick Start

### 1. Copy the CSS

Copy the contents of [`glassmorphism.css`](glassmorphism.css) to your clipboard.

### 2. Paste into Authentik

Go to **Admin** > **System** > **Brands** > select your brand > **Branding** > **Custom CSS** field.

Paste the entire CSS content and click **Update**.

### 3. Set a background image

In the same brand settings, go to **Other global settings** > **Attributes** and set:

```yaml
settings:
  theme:
    base: dark
    background: "background: url('https://your-cdn.com/background.jpg'); background-size: cover; background-position: center; background-repeat: no-repeat; background-attachment: fixed;"
```

### 4. Reload

Hard-refresh your login page (`Ctrl+Shift+R` / `Cmd+Shift+R`). The glassmorphism effect should be visible immediately.

## Customization

### Change the color scheme

The theme is controlled by **11 CSS variables** at the top of the file. Edit them to match your background:

```css
/* THEME COLOR CONFIG — Edit these values to change the theme */
--theme-tint: 15, 23, 52;        /* Glass surface tint (RGB) */
--theme-accent: 59, 130, 246;    /* Buttons, focus rings (RGB) */
--theme-border: 100, 140, 255;   /* Glass edge color (RGB) */
```

### Color presets

Uncomment one of the preset blocks in the CSS:

| Preset | `--theme-tint` | `--theme-accent` | Best for |
|--------|---------------|-----------------|----------|
| **Blue** (default) | `15, 23, 52` | `59, 130, 246` | Blue/tech backgrounds |
| **Red** | `52, 15, 20` | `239, 68, 68` | Warm/red backgrounds |
| **Green** | `15, 42, 30` | `16, 185, 129` | Nature/green backgrounds |
| **Purple** | `35, 15, 52` | `139, 92, 246` | Cyberpunk/violet backgrounds |
| **Neutral** | `20, 20, 25` | `139, 92, 246` | Any background (gray glass) |

### Add SSO provider colors

The theme includes brand-colored buttons for Discord, Plex, and Security Key. To add more providers, find the **SSO BUTTON TEMPLATE** comment in the CSS and copy the block:

```css
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

Replace `{SLUG}` with your source's slug (e.g., `source-google`) and `{R}, {G}, {B}` with the provider's brand color.

**Finding the slug:** Open your login page, press F12, and run:
```javascript
document.querySelector('ak-flow-executor').shadowRoot
  .querySelector('ak-stage-identification').shadowRoot
  .querySelectorAll('button.source-button, a.pf-c-button[name]')
  .forEach(b => console.log(b.getAttribute('name'), b.textContent.trim()));
```

**Common provider colors:**

| Provider | R, G, B |
|----------|---------|
| Google | `66, 133, 244` |
| Microsoft | `0, 164, 239` |
| GitHub | `110, 64, 201` |
| GitLab | `252, 109, 38` |
| Apple | `162, 170, 173` |
| Facebook | `24, 119, 242` |
| Slack | `74, 21, 75` |
| Okta | `0, 125, 193` |

### Background filter tips

You can add CSS filters to your background image for better glass effects:

```yaml
background: "background: url('...'); background-size: cover; background-position: center; background-repeat: no-repeat; background-attachment: fixed; filter: blur(2px) brightness(0.85) saturate(1.3);"
```

| Filter | Effect |
|--------|--------|
| `blur(2px)` | Softens the background, glass stands out more |
| `brightness(0.85)` | Darkens slightly for better text contrast |
| `saturate(1.3)` | Richer colors visible through the glass |

## How it works

Authentik uses **Web Components with Shadow DOM** (Lit + PatternFly 4). Custom CSS from the brand settings is adopted into each component's shadow root. This theme works by:

1. **CSS Variables** on `:root, :host` — cascade through all shadow boundaries
2. **Direct selectors** without `.pf-c-login` prefix — work inside each shadow root where CSS is adopted
3. **`::part()` selectors** — style library cards from the parent shadow root
4. **`:host()` scoping** — login stage selectors prevent glass from leaking into admin

### Shadow DOM structure (login)

```
ak-flow-executor [.pf-c-login host]
  └─ shadowRoot
      ├─ .pf-c-login__main        ← glass container
      └─ ak-stage-identification
          └─ shadowRoot
              ├─ inputs, buttons   ← glass form controls
              └─ ak-flow-card
                  └─ shadowRoot    ← styled via CSS variable cascade
```

### Shadow DOM structure (library)

```
ak-interface-user
  └─ shadowRoot
      ├─ .pf-c-page__header       ← glass header
      └─ ak-library-impl
          └─ shadowRoot            ← styled via ::part()
              ├─ cards, search, groups
```

## Compatibility

| Authentik | Status |
|-----------|--------|
| 2026.2.x | Tested |
| 2026.1.x | Should work |
| 2025.x | Not tested, may need adjustments |
| < 2025 | Likely incompatible (different Shadow DOM structure) |

**Browser support:** Chrome 105+, Firefox 103+, Safari 15.4+, Edge 105+ (requires `backdrop-filter` support).

## Files

| File | Description |
|------|-------------|
| `glassmorphism.css` | Main theme file — copy this into your brand CSS |
| `docs/architecture.md` | Deep dive into Authentik's Shadow DOM and CSS strategy |
| `docs/sso-colors.md` | Complete guide for adding SSO provider colors |
| `docs/deployment.md` | Deployment methods (UI, API, automation) |

## Contributing

Contributions are welcome!

Before submitting a PR:
- Test on a real Authentik 2026.x instance
- Verify admin interface is not affected
- Check login, library, and consent pages
- Test with `prefers-reduced-motion` enabled

## License

[MIT](LICENSE)

## Credits

Created by [YouKyi](https://youkyi.net) with assistance from Claude Code.
