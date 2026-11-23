---
name: showpad-html-app
description: Use when building an HTML5 web app as a Showpad asset, with no admin configuration, linking to Showpad files from HTML with showpad:// URLs, or deciding between an HTML5 web app and a Showpad App v2 for interactive Showpad content.
version: 1.0.0
---

# Showpad HTML5 Web Apps

An HTML5 web app is a static site (HTML, CSS, JS, images, fonts) zipped and uploaded to the Showpad Library. It behaves like any other asset across the whole platform. Showpad unzips it, makes a thumbnail from `index.html`, and renders it in the Asset Viewer on desktop, tablet and mobile (iOS, Android, Windows, Web). It's searchable and manageable in the content library, can be added to Experiences, and can go in Shared Spaces to share externally.

Source: https://developer.showpad.com/docs/html5-apps/intro/

## Decide first: web app or Showpad App?

|                                     |                              HTML5 web app                              |                                            Showpad App v2                                            |
| ----------------------------------- | :---------------------------------------------------------------------: | :--------------------------------------------------------------------------------------------------: |
| What it is                          |                        A Library asset (`.zip`)                         |                             Extensions installed in the org (`.showpad`)                             |
| Where it shows up                   | Anywhere an asset can: Library, Experiences, Collections, Shared Spaces | Where its extension types plug in (Experiences, settings, asset viewer, Shared Space Quick Actions…) |
| Shared externally as content        |                                    ✓                                    |                             ✗ (an experience can't go in a Shared Space)                             |
| Admin configuration                 |                                    ✗                                    |                                                  ✓                                                   |
| Interaction tracking (`trackEvent`) |                                    ✗                                    |                                                  ✓                                                   |
| AppsDB                              |                                    ✗                                    |                                                  ✓                                                   |
| Best for                            |               Static HTML app with no admin configuration               |                             Configurable tools, guided experiences, ...                              |

- **Web app** (this skill): build once and reuse everywhere, no admin customization, shareable externally via Shared Spaces.
- **Showpad App v2** (load the `showpad-app` skill): trackable interactions, admin configuration, persistent shared data, or extending Showpad's own UI.

A web app has **no runtime configuration** (no config API or editor; everything is fixed at build time), **no `trackEvent`** (Showpad logs only the asset open) and **no guaranteed cross-origin requests** (the viewer is sandboxed and may block external APIs). If the user needs any of these, propose a Showpad App instead of working around it, then hand over to the `showpad-app` skill.

## Hard packaging rules

| Rule                                                                                           | Why                                                    |
| ---------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `index.html` at the **root** of the zip, not inside a subfolder                                | It's the entry point                                   |
| `.zip` only, at most **2 GB**. The unzipped size must be ≤ 1.5 GB for Windows offline          | Upload limit                                           |
| Name pages `.html`, never `.htm`. No spaces in file names, no symlinks                         | Not supported                                          |
| All paths **relative** (`./js/app.js`, not `/js/app.js`)                                       | The app isn't served from the domain root              |
| Every library, font and image goes **inside the zip**. No CDNs and no runtime API dependencies | It must work offline. Downloaded apps run from the zip |
| Static files only. No PHP, Ruby, .NET or other server-side code                                | No server runtime                                      |

Bundler settings that keep paths relative:

- **Vite**: `base: './'` in `vite.config.*`
- **Angular**: `<base href="">` in `index.html` (the default `/` breaks loading)
- **CRA / webpack**: `"homepage": "."` / `output.publicPath: './'`

## Package and verify

Zip the **contents** of the build folder, not the folder itself:

```bash
npm run build                                   # if there is a build step; output in dist/
cd dist && rm -f ../app.zip && zip -r -X ../app.zip . -x '.*' -x '__MACOSX/*' && cd ..
zipinfo -1 app.zip | grep -qx 'index.html' && echo "OK: index.html at root" || echo "FAIL: index.html not at root"
zipinfo -1 app.zip | grep -E '\.htm$| ' && echo "FAIL: .htm or spaces in names above"
grep -rnE '(src|href)="/[^/]|https?://(cdn|unpkg|cdnjs|fonts\.googleapis)' dist --include='*.html' --include='*.css' --include='*.js' | head
```

The last command should print nothing. Each hit is an absolute path or a CDN dependency to fix.

Upload: Admin App → Library → upload the `.zip`, like any file. There's no CLI for web apps. Don't use `npx showpad apps upload`, which is for Showpad Apps only.

## Minimal `index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>My Web App</title>
    <link rel="stylesheet" href="./css/style.css" />
  </head>
  <body>
    <!-- Static image so the generated thumbnail isn't blank (Showpad renders index.html before JS runs) -->
    <img
      id="thumbnail"
      src="./img/thumbnail.png"
      alt=""
      style="display:block;width:100%"
    />
    <div id="app" style="display:none;width:100%;height:100%"></div>
    <script src="./js/app.js"></script>
  </body>
</html>
```

`app.js` hides `#thumbnail` and shows `#app` once the app is ready.

## Talking to Showpad (optional)

Use `@showpad/experience-app-sdk`. **Never call `window.ShowpadLib` directly**, even though the platform injects it. Load the `showpad-app-sdk` skill for the exact signatures.

- Install the same version as Showpad Apps: `npm install @showpad/experience-app-sdk@^3.5.0`.
- Documented for web apps: user info (`getUserInfo`), Collections (create, clear, add assets), the current Share, sharing and opening assets. `trackEvent` is **not supported**.
- Don't use the Showpad App–only methods: config (`getConfig`, `parseConfig`, `parseEnrichedConfig`), AppsDB, `upload` and `addAssetsToCurrentContext`. They depend on an installed app. If one of them is needed, that's the signal to switch to a Showpad App.
- Bundle the SDK into the zip with a bundler. Don't load it from a CDN. If the app has no build step yet, add Vite (`npm create vite@latest -- --template vanilla-ts`, then set `base: './'`) instead of calling `ShowpadLib` directly.
- Await `Showpad.onShowpadLibLoaded()` before any other call.
- SDK calls don't work in the Admin App preview or in a plain browser. Race the readiness check against a timeout so the app still renders there:

```ts
import { Showpad } from "@showpad/experience-app-sdk";

const inShowpad = await Promise.race([
  Showpad.onShowpadLibLoaded().then(() => true),
  new Promise<boolean>((r) => setTimeout(() => r(false), 5000)),
]);
if (inShowpad) {
  const user = await Showpad.getUserInfo();
  // ...
}
```

## Links

| Target                    | Markup                                                                                                                |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Showpad file              | `<a href="showpad://file/{assetSlug}">`. Add `?modal=1` to open in an overlay, `&page=5` to open a PDF page (1-based) |
| File in a specific folder | `<a href="showpad://folder/{folderId}/file/{assetSlug}">`                                                             |
| Experience                | `<a href="showpad://profile/{experienceId}">`                                                                         |
| Record a Showpad Video    | `<a href="showpad://video">`                                                                                          |
| External site             | `<a href="https://...">`. HTTPS only                                                                                  |

- Use `modal=1` inside Advanced Experiences, so closing returns to the web app.
- `showpad://file/` links take the asset **slug**, not the asset ID. Get the slug from the file's **App link** in Library → file details → Overview, or from `asset.slug` in the SDK. The Experience ID is the **App link** in the Experiences details.
- Never use iframes for internal navigation. They don't work on iPad.
- Don't hard-code slugs or IDs without asking the user for them. They differ per organisation.

## Prefill a share (email or link)

```
https://{org}.showpad.biz/webapp2/share?action=email&subject=...&body=...&attachments={slug1},{slug2}&recipients={urlencoded JSON}
```

`action` is required: `email` or `link`. `recipients` is a URL-encoded JSON array of `{ "Email", "FirstName"?, "LastName"? }`. URL-encode every value with `encodeURIComponent`.

## Runtime constraints

- Webviews: iOS WKWebView, Android Chrome, Windows WebView2, and Edge/Chrome/Safari/Firefox on Web. Use only features all of them support.
- Make the layout responsive. The Collection Viewer is an iframe, so set the viewport and `overflow` correctly. On iOS, keep the **top-left corner** clear for Showpad's navigation button.
- Prefer CSS animations or `requestAnimationFrame` over JS timers. Watch memory on mobile.
- `localStorage` works. Use it to keep app state between sessions.
- Analytics on interactions: Showpad can't track them. The only documented route is GA4 `gtag` in `index.html`, which is third-party, needs network access and should use SRI. Ask the user before adding it.
- To test offline, open the app in a Showpad mobile app in airplane mode.

## Outgrowing a web app

If the requirements grow (admin configuration, tracking, AppsDB), keep the build: a Vite project with `base: './'` can become the folder of an `experienceTypes` extension unchanged. Load the `showpad-app` skill and start at its Phase 1. That adds a manifest, a `config.json` or Custom Editor, and the CLI bundle/upload. The asset keeps working until it's replaced. Note that an experience can't be shared through Shared Spaces.

## Security

- Render user and asset data with `textContent`, not `innerHTML`. Sanitize any HTML you must inject (vendored DOMPurify).
- Validate every input, including URL query parameters.
- No secrets in the zip. Anyone with the asset can unzip it. Put API keys behind a server-side proxy.
- External calls go over HTTPS only.
