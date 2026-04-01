# Contributing

Thanks for your interest in contributing to the Authentik Glassmorphism Theme!

## How to contribute

### Bug reports

If something looks broken on your Authentik instance:

1. Open an issue with a screenshot
2. Include your Authentik version (e.g., 2026.2.1)
3. Specify which page is affected (login, library, admin, consent)
4. Mention your browser and OS

### Feature requests

Open an issue describing what you'd like to see. Examples:
- New color preset
- Support for a specific Authentik page/component
- New SSO provider color

### Pull requests

1. Fork the repo
2. Make your changes to `glassmorphism.css`
3. Test on a real Authentik 2026.x instance
4. Submit a PR with screenshots of before/after

## Testing checklist

Before submitting, verify:

- [ ] **Login page** — glass container, inputs, buttons, SSO colors, footer visible
- [ ] **Library page** — cards, search bar, group headers, hover effects
- [ ] **Admin interface** — no glass effects leaking (forms, modals, tables should look normal)
- [ ] **Consent flow** — green/red buttons render correctly
- [ ] **Accessibility** — enable `prefers-reduced-motion` in browser settings, verify no animations
- [ ] **Mobile** — resize to 375px width, check login and library layouts

## Code style

- Use `!important` on all properties (Authentik convention for custom CSS)
- Comment in English
- Use CSS variables from the theme config where possible
- Scope selectors to avoid admin interference (use `:host(ak-stage-*)` for login inputs)
- Keep the SSO provider colors hardcoded (brand colors don't change with the theme)

## Architecture notes

Read [docs/architecture.md](docs/architecture.md) before making changes to understand:
- How Shadow DOM affects CSS selectors
- Why certain selectors need high specificity
- How the admin hotfix works
