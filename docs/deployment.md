# Deploying the CSS to Authentik

This guide covers every method for deploying the glassmorphism theme CSS to your Authentik instance, including the Admin UI, the REST API, and workarounds for large payloads.

## Prerequisites

- An Authentik instance running version 2026.x or later
- Admin access to the instance
- The `glassmorphism.css` file from this repository

## Method 1: Admin UI (Simplest)

This is the recommended approach for first-time setup.

1. Log in to your Authentik admin interface at `https://your-instance.com/if/admin/`
2. Navigate to **System** > **Brands**
3. Select the brand you want to theme (usually the default brand)
4. Scroll to the **Custom CSS** field under **Branding**
5. Paste the entire contents of `glassmorphism.css` into the field
6. Click **Update** at the bottom of the page

The CSS takes effect immediately. Hard-refresh the login page (`Ctrl+Shift+R` / `Cmd+Shift+R`) to see the changes.

## Method 2: REST API

The API is useful for automation, scripting, or when the CSS is too large to paste comfortably in the UI text field.

The endpoint is:

```
PATCH /api/v3/core/brands/{brand-uuid}/
```

The request body must include the `branding_custom_css` field with the full CSS string as its value.

### Finding Your Brand UUID

In the Admin UI, go to **System** > **Brands**, click your brand, and copy the UUID from the URL bar. It looks like `e56dafee-4270-4142-a867-21a3544b0368`.

Alternatively, list all brands via the API:

```bash
curl -s https://your-instance.com/api/v3/core/brands/ \
  -H "Authorization: Bearer {token}" | jq '.results[] | {pk, branding_title}'
```

### Authentication Option A: Session Auth (Browser Console)

If you are already logged in to Authentik in your browser, you can use session-based authentication. This requires two CSRF headers, both sourced from the `authentik_csrf` cookie.

```javascript
(async () => {
  // Extract the CSRF token from the cookie
  const csrfCookie = document.cookie
    .split(';')
    .map(c => c.trim())
    .find(c => c.startsWith('authentik_csrf='));
  const csrfToken = csrfCookie.split('=')[1];

  // Read the CSS file content (replace with your actual CSS string)
  const cssContent = `PASTE_YOUR_CSS_HERE`;

  const resp = await fetch('/api/v3/core/brands/{brand-uuid}/', {
    method: 'PATCH',
    credentials: 'include',
    headers: {
      'Content-Type': 'application/json',
      'X-authentik-CSRF': csrfToken,
      'X-CSRFToken': csrfToken
    },
    body: JSON.stringify({ branding_custom_css: cssContent })
  });

  console.log('Status:', resp.status);
  const data = await resp.json();
  console.log('Response:', data.branding_custom_css?.length, 'chars saved');
})();
```

Both `X-authentik-CSRF` and `X-CSRFToken` headers are required. They use the same value, extracted from the `authentik_csrf` cookie. Omitting either header results in a 403 Forbidden response.

### Authentication Option B: Bearer Token (curl / Scripts)

Create an API token in **Admin > System > Tokens and App passwords** with the **Intent** set to **API**. The token must belong to a superuser account.

```bash
curl -X PATCH "https://your-instance.com/api/v3/core/brands/{brand-uuid}/" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d @css_payload.json
```

Where `css_payload.json` contains:

```json
{
  "branding_custom_css": "/* your full CSS here, escaped for JSON */"
}
```

To generate the payload file from a CSS file:

```bash
jq -n --rawfile css glassmorphism.css '{"branding_custom_css": $css}' > css_payload.json
```

This uses `jq` to properly escape the CSS content for JSON (handling quotes, newlines, and special characters).

## Method 3: Large CSS Workaround

The glassmorphism theme can exceed 30,000 characters. If the CSS is too large to paste inline in a browser console command (some browsers truncate long strings), use a local HTTP server as an intermediary.

### Step 1: Generate the JSON Payload

```bash
jq -n --rawfile css glassmorphism.css '{"branding_custom_css": $css}' > /tmp/css_payload.json
```

### Step 2: Serve It Locally with CORS

Start a temporary HTTP server that includes CORS headers so the browser allows the cross-origin fetch:

```bash
cd /tmp && python3 -c "
from http.server import HTTPServer, SimpleHTTPRequestHandler

class CORSHandler(SimpleHTTPRequestHandler):
    def end_headers(self):
        self.send_header('Access-Control-Allow-Origin', '*')
        self.send_header('Access-Control-Allow-Methods', 'GET')
        super().end_headers()

HTTPServer(('127.0.0.1', 9876), CORSHandler).serve_forever()
"
```

### Step 3: Fetch and PATCH from the Browser

Open a tab authenticated to your Authentik instance and run:

```javascript
(async () => {
  // Fetch the payload from the local server
  const payloadResp = await fetch('http://localhost:9876/css_payload.json');
  const payload = await payloadResp.json();

  // Extract CSRF token
  const csrfCookie = document.cookie
    .split(';')
    .map(c => c.trim())
    .find(c => c.startsWith('authentik_csrf='));
  const csrfToken = csrfCookie.split('=')[1];

  // PATCH the brand
  const resp = await fetch('/api/v3/core/brands/{brand-uuid}/', {
    method: 'PATCH',
    credentials: 'include',
    headers: {
      'Content-Type': 'application/json',
      'X-authentik-CSRF': csrfToken,
      'X-CSRFToken': csrfToken
    },
    body: JSON.stringify(payload)
  });

  console.log('Status:', resp.status);
  console.log('CSS length:', payload.branding_custom_css.length, 'chars');
})();
```

After the PATCH succeeds, stop the local server with `Ctrl+C`.

## Verification

After deploying, verify the CSS is active by checking computed styles on the login page.

### Check the Glass Container

Open your login page (`https://your-instance.com/if/flow/default-authentication-flow/`) and run in the console:

```javascript
const fe = document.querySelector('ak-flow-executor');
const main = fe.shadowRoot.querySelector('.pf-c-login__main');
const styles = getComputedStyle(main);

console.log('background:', styles.backgroundColor);
console.log('backdrop-filter:', styles.backdropFilter);
console.log('border-radius:', styles.borderRadius);
```

Expected output (with default blue theme):

```
background: rgba(15, 23, 52, 0.55)
backdrop-filter: blur(20px) saturate(1.5)
border-radius: 20px
```

### Check SSO Button Colors

```javascript
const fe = document.querySelector('ak-flow-executor');
const stage = fe.shadowRoot.querySelector('ak-stage-identification');
const ssr = stage.shadowRoot;

// Check a specific button (replace with your source slug)
const btn = ssr.querySelector('button[name="source-discord"]');
console.log('bg:', getComputedStyle(btn).backgroundColor);
console.log('border:', getComputedStyle(btn).borderColor);
console.log('bg-image:', getComputedStyle(btn).backgroundImage);
```

If `backgroundColor` shows `rgba(0, 0, 0, 0)` or the default accent gradient appears in `backgroundImage`, the CSS is not being applied. Double-check that you saved the brand and hard-refreshed the page.

## Background Image

The background image is **not** set in the CSS. It is configured separately in the Brand's **Attributes** field as YAML.

### Setting the Background

In the Admin UI, go to **System** > **Brands** > your brand > **Other global settings** > **Attributes** and enter:

```yaml
settings:
  theme:
    base: dark
    background: "background: url('https://your-cdn.com/background.jpg'); background-size: cover; background-position: center; background-repeat: no-repeat; background-attachment: fixed;"
```

The `background` value is a raw CSS string that Authentik injects into the page body. You can include any valid CSS background properties.

### Background Filters

You can add CSS filters to the background for better contrast with the glass surfaces:

```yaml
settings:
  theme:
    base: dark
    background: "background: url('https://your-cdn.com/background.jpg'); background-size: cover; background-position: center; background-repeat: no-repeat; background-attachment: fixed; filter: blur(2px) brightness(0.85) saturate(1.3);"
```

| Filter | Effect | Recommended value |
|---|---|---|
| `blur()` | Softens the background so glass stands out | `1px` to `3px` |
| `brightness()` | Darkens the image for better text contrast | `0.75` to `0.90` |
| `saturate()` | Intensifies colors visible through the glass | `1.2` to `1.5` |
| `contrast()` | Adjusts tonal range | `0.9` to `1.1` |
| `hue-rotate()` | Shifts the color palette | `0deg` to `360deg` |

Combine multiple filters in a single `filter` declaration separated by spaces (as shown above). Stronger blur and lower brightness tend to produce the most polished glassmorphism look.

## Updating the CSS

To update the CSS after making changes (adding SSO colors, adjusting variables, etc.), use the same process:

1. Edit `glassmorphism.css` locally
2. Deploy using any of the methods above (Admin UI paste, API PATCH with bearer token, or the local HTTP server workaround)

The API `PATCH` method replaces the entire `branding_custom_css` field. There is no incremental update -- you always send the full CSS content.

### Quick Update via curl

If you have a bearer token configured, the fastest workflow is:

```bash
# Regenerate the payload
jq -n --rawfile css glassmorphism.css '{"branding_custom_css": $css}' > /tmp/css_payload.json

# Push to Authentik
curl -X PATCH "https://your-instance.com/api/v3/core/brands/{brand-uuid}/" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d @/tmp/css_payload.json

# Clean up
rm /tmp/css_payload.json
```

A successful response returns HTTP 200 with the full brand object. Verify by hard-refreshing the login page.
