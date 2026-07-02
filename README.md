# Pardot Form Handler Selector — Kontent.ai Custom Element

A [Kontent.ai custom element](https://kontent.ai/learn/docs/custom-elements) that lets content
editors pick a Salesforce Pardot form handler from a dropdown, instead of pasting a raw ID or URL
by hand. The selected form handler is stored on the content item as JSON (`id`, `name`, `url`).

It's a single self-contained `index.html` — no build step, no dependencies.

## How it works

```
┌──────────────┐      GET /api/pardot-forms      ┌──────────────┐      Pardot API      ┌────────┐
│ index.html   │ ───────────────────────────────▶│ your proxy   │ ────────────────────▶│ Pardot │
│ (in Kontent) │◀─────────────────────────────────│ server       │◀──────────────────────│        │
└──────────────┘   [{ id, name, url }, ...]        └──────────────┘                     └────────┘
```

Pardot's API requires OAuth credentials and doesn't allow direct browser (CORS) requests, and
Kontent.ai custom elements explicitly warn against storing secrets in their JSON parameters. So
this element never talks to Pardot directly — it calls a small proxy endpoint you host, which
holds the Pardot credentials server-side and returns a plain JSON list of form handlers.

## 1. Deploy the proxy endpoint

You need one HTTP endpoint that:

- Accepts `GET` requests (no auth required from the browser side — it runs inside a sandboxed
  iframe with no way to attach secrets).
- Authenticates to Pardot/Salesforce with your own OAuth credentials.
- Returns JSON in one of these shapes:

  ```json
  [{ "id": "1001", "name": "Contact Us", "url": "https://go.pardot.com/l/xxx/contact" }]
  ```

  or

  ```json
  { "formHandlers": [{ "id": "1001", "name": "Contact Us", "url": "..." }] }
  ```

- Sends CORS headers that allow the Kontent.ai app origin to read the response
  (`Access-Control-Allow-Origin: https://app.kontent.ai` at minimum).

This repo doesn't include the proxy itself — build it with whatever serverless platform you
prefer (e.g. a Vercel/Netlify function). Keep your Pardot Business Unit ID and OAuth token out of
version control and out of the custom element's JSON parameters — those parameters are visible to
anyone with access to the content type.

## 2. Host `index.html`

Custom elements are loaded in an `<iframe>` and must be served over HTTPS. Host `index.html`
yourself or on a static host such as [Netlify](https://www.netlify.com/). If you host it on your
own server, make sure it's served with a header that allows Kontent.ai to frame it, e.g.:

```
Content-Security-Policy: frame-ancestors https://app.kontent.ai
```

See [Host your custom element securely](https://kontent.ai/learn/docs/custom-elements#a-host-your-custom-element-securely)
for details.

## 3. Add the element to a content type

1. In **Content model**, open the content type you want to add the selector to.
2. Add a new **Custom element**.
3. Set **Hosted code URL (HTTPS)** to the URL where you deployed `index.html`.
4. In **Parameters {JSON}**, point it at your proxy:

   ```json
   { "proxyUrl": "https://your-proxy.example.com/api/pardot-forms" }
   ```

5. Save changes.

## Stored value

When an editor picks a form handler, the element calls `CustomElement.setValue` with a
`ValueObject` so the form name is searchable in the content items list. The element's `value` on
the content item ends up as a JSON string:

```json
{ "id": "1001", "name": "Contact Us", "url": "https://go.pardot.com/l/xxx/contact" }
```

Retrieve it via the Delivery/Management API like any other custom element — parse the `value`
string to get `id`, `name`, and `url`.

> Creating a content item via the **Management API** does not run the element's code, so no
> default value gets set automatically — insert the JSON value yourself if you need one.

## Local development

Open `index.html` directly in a browser (or serve it with any static file server). Outside of the
Kontent.ai iframe, `CustomElement.init` throws, and the element falls back to demo data (`Contact
Us`, `Newsletter Signup`, etc.) so you can check the UI without a live proxy or Pardot connection.

To test inside an actual Kontent.ai content type while developing locally, tunnel your local
server with a service like [ngrok](https://ngrok.com/) and use the resulting HTTPS URL as the
**Hosted code URL**.

## Troubleshooting

| Status message | Cause |
| --- | --- |
| `"proxyUrl" is missing from the element's JSON parameters` | Add `{"proxyUrl": "..."}` to the content type's custom element configuration. |
| `Proxy returned 4xx/5xx ...` | The proxy endpoint errored — check its logs and Pardot OAuth credentials. |
| `The previously selected form handler was not found in Pardot` | The form handler stored on this content item was deleted or renamed in Pardot. Pick a new one. |
| `No form handlers found in your Pardot account` | The proxy returned an empty list. |
