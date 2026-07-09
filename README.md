# Landing1 Plugin

Public landing page with tariff plan selection. Can be used standalone (`/landing1`) or embedded via iframe on external websites.

## Routes

| Route | Auth | Description |
|-------|------|-------------|
| `/landing1` | No | Standalone plan selection page |
| `/embed/landing1` | No | Embeddable version for iframes |

## Standalone Usage

Navigate directly to your VBWD instance:

```
http://your-vbwd-domain.com/landing1
```

Clicking a plan redirects to `/checkout?tarif_plan_id=<slug>`.

## Embedding on External Websites

The embed mode renders the plan selection inside an iframe. Plan clicks send a `postMessage` to the parent page instead of navigating.

The loader served by the host at `/embed/widget.js` finds its own `<script>` tag
(by `data-embed` or `data-origin`), builds the iframe URL `/embed/<embed>?...`,
and forwards the presentation attributes below as query params. `EmbedLanding1View`
reads those params; `Landing1View` renders the cards.

### Basic Embed

```html
<div id="pricing-root"></div>
<script
  src="https://your-vbwd-domain.com/embed/widget.js"
  data-embed="landing1"
  data-container="pricing-root"
></script>
```

### Full Example

```html
<div id="embed-live-preview"></div>
<script
  src="/embed/widget.js"
  data-embed="landing1"
  data-category="root"
  data-container="embed-live-preview"
  data-locale="en"
  data-theme="indigo"
  data-height="650"
  data-highlight="pro"
  data-features="All core platform features,Unlimited projects,Priority email support,Cancel anytime"
></script>
```

Plan clicks default to redirecting to `<origin>/checkout?tarif_plan_id=<slug>`;
call `event.preventDefault()` in a `vbwd:plan-selected` listener to override:

```html
<script>
  document.getElementById('embed-live-preview').addEventListener('vbwd:plan-selected', function(e) {
    e.preventDefault();
    window.location.href = 'https://your-shop.example.com/buy?plan=' + e.detail.planSlug;
  });
</script>
```

### Widget Attributes

| Attribute | Required | Default | Description |
|-----------|----------|---------|-------------|
| `data-embed` | No | `landing1` | Embed preset to render (the iframe path segment). |
| `data-origin` | No | current page origin | Full URL of your VBWD instance; omit when the script is served from that instance. |
| `data-category` | No | — | Tariff plan category slug (`root` shows all plans). |
| `data-container` | Yes | `vbwd-iframe` | ID of the host `<div>` the iframe is appended to. |
| `data-locale` | No | `en` | UI language (`en`, `de`, `fr`, `ru`, …). |
| `data-theme` | No | `light` | Card colour theme (see allowed values below). |
| `data-height` | No | `600` | Initial iframe height in pixels. |
| `data-highlight` | No | — | Plan slug rendered as the featured card. |
| `data-image` | No | — | Header image URL shown above the cards. |
| `data-features` | No | — | Feature bullets, as **one comma-separated list** (see caveat). |
| `data-heading` | No | — | Overrides the card heading. |
| `data-subtitle` | No | — | Overrides the card subtitle. |
| `data-cta` | No | — | Overrides the call-to-action button label. |
| `data-badge` | No | — | Overrides the featured-plan badge label. |
| `data-checkout-url` | No | `<origin>/checkout` | Base URL for the default plan-click redirect. |

**Allowed `data-theme` values:** `default`, `light`, `dark`, `teal`, `indigo`,
`emerald`. Any other value silently falls back to `default`.

**`data-features` caveat:** the loader forwards `data-features` as a single
comma-separated string and `EmbedLanding1View` splits it on `,` — so an
individual feature bullet must **not** contain a comma (it would split into two).

**i18n fallback:** leave `data-heading`, `data-subtitle`, `data-cta` and
`data-badge` **unset** to use the built-in localised text (`landing1.title`,
`landing1.subtitle`, `landing1.choosePlan`, `landing1.popular`) — this keeps the
card correct in every locale. Setting them pins one language.

### Events

The iframe communicates with the parent page via `postMessage`. The widget script converts these into DOM `CustomEvent`s on the container element.

#### `vbwd:plan-selected`

Fired when a user clicks "Choose Plan".

```js
document.getElementById('vbwd-iframe').addEventListener('vbwd:plan-selected', function(e) {
  console.log(e.detail);
  // {
  //   planSlug: "basic-monthly",
  //   planName: "Basic",
  //   price: 9.99,
  //   currency: "EUR"
  // }
});
```

#### `vbwd:resize`

Fired automatically when the iframe content height changes. The widget handles this internally — the iframe resizes itself. No action needed from the host page.

### Security

- The iframe uses `sandbox="allow-scripts allow-same-origin allow-forms allow-popups"`
- Only `/embed/*` paths allow framing (`Content-Security-Policy: frame-ancestors *`)
- All other routes respond with `X-Frame-Options: DENY`
- The widget validates `data-origin` as a valid URL before creating the iframe
- `postMessage` events are validated against the configured origin

## Files

| File | Description |
|------|-------------|
| `index.ts` | Plugin entry — registers routes and translations |
| `Landing1View.vue` | Standalone plan selection page |
| `EmbedLanding1View.vue` | Embed wrapper — handles postMessage, locale, theme, auto-resize |
| `embed-widget.js` | Legacy standalone loader snippet (data-origin only). The current embed is served by the host at `/embed/widget.js`, which also supports `data-embed` + the presentation attributes above. |
| `locales/en.json` | English translations |
| `locales/de.json` | German translations |
| `config.json` | Plugin config schema |
| `admin-config.json` | Admin panel UI config |

---

## Related

**Core:** [vbwd-fe-user](https://github.com/VBWD-platform/vbwd-fe-user) · [vbwd-fe-core](https://github.com/VBWD-platform/vbwd-fe-core)
