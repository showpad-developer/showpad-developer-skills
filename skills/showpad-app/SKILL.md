---
name: showpad-app
description: This skill should be used when the user asks to "create a Showpad App", "scaffold a Showpad App", "build a Showpad App", "develop a Showpad App v2", "upload a Showpad App", "bundle a Showpad App", "add a Showpad extension", "create an experience type", "create a custom editor", "create admin settings extension", "create user settings extension", "create add content extension", "create asset viewer extension", "create shared space extension", or anything related to Showpad Apps v2 development, the Showpad CLI, or the Experience App SDK. For a zipped HTML5 web app uploaded to the Library as an asset, or when deciding between the two, use the showpad-html-app skill instead.
version: 1.2.0
---

# Showpad App v2 Developer Skill

You guide developers through the Showpad App v2 lifecycle: scaffold → develop → build → bundle → upload → assign.

**Load only what the current task needs.** The references hold the exact field names, schemas and constraints. Don't guess them.

| Resource                           | Load when                                                              |
| ---------------------------------- | ---------------------------------------------------------------------- |
| `references/manifest-reference.md` | writing or validating `manifest.json` (always when scaffolding)        |
| `references/extension-types.md`    | scaffolding or configuring an extension type (always when scaffolding) |
| `references/cli-commands.md`       | needing CLI flags beyond this file, or CI/CD                           |
| `references/error-reference.md`    | diagnosing bundle, upload or validation errors                         |
| `showpad-app-sdk` skill            | writing any `Showpad.*` call (signatures, types, AppsDB methods)       |
| `showpad-app-editor-sdk` skill     | building a Custom Editor (`editorFolder`, `ShowpadEditor.*`)           |
| `showpad-html-app` skill           | the user wants a static HTML5 web app (Library `.zip` asset) instead   |

**Check first that a Showpad App is the right fit.** If the user wants self-contained interactive content with no admin configuration, no `trackEvent`, no AppsDB, and it must be shareable in Shared Spaces, an HTML5 web app is simpler: switch to the `showpad-html-app` skill. "Showpad app" in casual speech can mean either, so ask if it's unclear.

Don't start scaffolding until the Phase 1 questions are answered. If the user's message already answers them, go straight to Phase 2.

---

## Phase 1 — Understand the use case

Ask the unanswered questions in a single message:

1. **What should the app do?** This decides the extension types. Most apps combine 2–3 of them.

   | Goal                                                  | Extension type          |
   | ----------------------------------------------------- | ----------------------- |
   | A new Showpad Experience for sales reps               | `experienceTypes`       |
   | A config panel for admins                             | `adminSettings`         |
   | A settings panel for individual users                 | `userSettings`          |
   | Add content to Shared Spaces / Collections / My Files | `addContentExtensions`  |
   | Enhance the asset viewer                              | `assetViewerExtensions` |
   | Quick Actions in Shared Spaces                        | `sharedSpaceExtensions` |

   For `experienceTypes`, also ask: should admins edit it with the **Default Editor** (a form generated from `config.json`) or a **Custom Editor** (your own webapp, set with `editorFolder`)? Custom Editors are **beta**: they rely on `Showpad.getConfig` / `getAssetsByIds` / `getAssetIdsByFilterId`, which only work on orgs with the `experience_app_custom_editor` feature flag. If it isn't enabled, they must ask their Showpad account manager.

2. **App key?** Format `com.your-company.your-app`. It must be globally unique, lowercase (`^[a-z][a-z0-9.-]+$`) and must not start with `com.showpad.`.
3. **Developer name?** Must not contain `showpad`.
4. **Frontend stack?** Vite + TypeScript is recommended. Plain HTML/JS also works, as does any bundler that outputs static files.
5. **Persistent or cross-user data?** If yes, use AppsDB.

---

## Phase 2 — Scaffold the project

Load `references/manifest-reference.md` and `references/extension-types.md` first. Build the manifest and extension folders from them.

```text
<app-name>/
├── manifest.json
├── icon.png               # optional
├── images/                # optional, up to 3 cover images
├── <extension-folder>/    # one per extension; name = resources.folder in the manifest
│   ├── index.html         # required iframe entry point
│   ├── public/config.json # experience type with the Default Editor only
│   ├── src/main.ts
│   └── vite.config.ts
├── <editor-folder>/       # Custom Editor only (resources.editorFolder), same layout
├── tsconfig.json
└── package.json           # single root package, shared node_modules
```

### Install

The CLI needs Node.js 20+.

```bash
mkdir <app-name> && cd <app-name> && npm init -y
npm install @showpad/experience-app-sdk@^3.5.0
npm install @showpad/experience-editor-sdk@^0.4.0 # only for a Custom Editor
npm install -D @showpad/cli vite typescript @types/node
```

### Config files

`tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noEmit": true,
    "skipLibCheck": true
  },
  "include": ["**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

`<extension-folder>/index.html` is the iframe entry point:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>My Extension</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="./src/main.ts"></script>
  </body>
</html>
```

`<extension-folder>/vite.config.ts` needs one per extension. `outDir` must be the **root** `dist/<extension-folder>` so that `showpad apps bundle -s dist` finds every extension:

```typescript
import { defineConfig } from "vite";

export default defineConfig({
  base: "./", // required: absolute paths break inside iframes
  build: {
    outDir: "../dist/admin-settings", // relative to the extension folder
    emptyOutDir: true,
  },
});
```

`package.json` scripts. List one `vite build <folder>` per extension. A Custom Editor's `editorFolder` counts as its own folder (with its own `vite.config.ts`) unless it's the same as `folder`. Drop the `icon`/`images` copies if the app has none:

```json
{
  "type": "module",
  "scripts": {
    "dev": "vite --force",
    "build": "tsc && vite build admin-settings && vite build experience",
    "prebundle": "cp manifest.json dist && cp icon.png dist && cp -r images dist",
    "bundle": "showpad apps bundle -s dist"
  }
}
```

There's a working 3-extension reference at `https://github.com/showpad-developer/apps-v2-quick-start`.

---

## Phase 3 — Develop extensions

Invoke the `showpad-app-sdk` skill before writing `Showpad.*` calls. Always await `onShowpadLibLoaded()` first:

```typescript
import { Showpad } from "@showpad/experience-app-sdk";

const main = async (): Promise<void> => {
  await Showpad.onShowpadLibLoaded();
  // SDK calls are safe from here
};
main();
```

### Experience types

- **Default Editor** (no `editorFolder`): add `config.json` to the experience folder's `public/` directory, so that Vite copies it into `dist/<folder>/`. With no build step, put it at the folder root. The schema and how to read it (`Showpad.parseConfig()` / `parseEnrichedConfig()`) are in `references/extension-types.md`.
- **Custom Editor** (`editorFolder` set, beta): no `config.json`. Invoke the `showpad-app-editor-sdk` skill. `Showpad.getConfig()` resolves `undefined` until the admin saves, so always merge it over a default config. It covers both the editor webapp and how the experience reads and resolves the config. `editorFolder` can be a separate folder or the same as `folder` (WYSIWYG mode: the editor is loaded with `?edit=true`).

### AppsDB

Declare the stores in the manifest under `sharedStores`, then invoke the `showpad-app-sdk` skill and read `appsdb.md` in its `references/` folder. It covers scopes, limits and the no-credentials/no-PII rule.

### Asset cards

Enrich the assets to get `thumbnailUrl` (`Showpad.enrichAssets(assets)`), then link with the `showpad://` scheme. Identify assets by `slug`: `id` isn't stable across platforms.

```html
<a href="showpad://file/{asset.slug}">
  <img src="{asset.thumbnailUrl}" alt="" />
  <span>{asset.name}</span>
</a>
```

To open the viewer from code, call `Showpad.openAssetViewer(asset.slug)`.

### Security

- Render user, config and asset data with `textContent`. Never use `innerHTML`. If you really need HTML, sanitize it first (e.g. `DOMPurify.sanitize`).
- Never put secrets or PII in source code, config or AppsDB.

---

## Phase 4 — Build & bundle

**Bump `version` in `manifest.json` first.** A version can't be uploaded twice (`/problems/app-version-already-exists`). Use `1.0.0-1`, `1.0.0-2`, … during development.

```bash
npm run build   # dist/<extension-folder>/ for each extension
npm run bundle  # prebundle copies manifest/icon/images → <appKey>.<version>.showpad (rejected over 1 GB)
```

---

## Phase 5 — Upload

```bash
npx showpad login                                    # subdomain + credentials; use env vars in CI
npx showpad apps upload ./<appKey>.<version>.showpad
```

The upload polls its status. If it times out, it prints an `uploadId` you can check with `npx showpad apps status --upload-id <uploadId>`.

---

## Phase 6 — Assign to organizations

`--partner-code` is **required** on both commands and must be a UUID (e.g. `91dd4290-369e-488e-80df-634d6cb090a6`). Ask the user for the partner code first. It's in the Showpad platform: click **Install App**, and the install modal links to it in the bottom-left corner.

Only `--app-version-id` (a ULID) is optional: leave it out and the CLI lets you pick the app and version interactively. Get the `appId` from `npx showpad apps list`, then the `versionId` from `npx showpad apps versions --app-id <appId>`.

```bash
npx showpad apps assign --partner-code <partnerCode> --app-version-id <versionId> # repeat -p <partnerCode> for several orgs
npx showpad apps assign --partner-code <partnerCode> # pick the version interactively
npx showpad apps unassign --partner-code <partnerCode> --app-version-id <versionId> # takes minutes to propagate
```

---

## Phase 7 — Install and test

Install the app in the Admin App (**Apps → your app → Install**), then test each extension where it appears:

| Extension               | Where to find it                                             |
| ----------------------- | ------------------------------------------------------------ |
| `adminSettings`         | Admin App → Apps → your app → Settings tab                   |
| `userSettings`          | Web App → Settings → your app section                        |
| `experienceTypes`       | Online Platform → Experiences → Create New → your type       |
| `addContentExtensions`  | Web App → Shared Space / Collection / My Files → Add Content |
| `assetViewerExtensions` | Web App → open an asset → your extension panel               |
| `sharedSpaceExtensions` | Web App → Shared Space → Quick Actions                       |

**Local development:** run `npm run dev <extension-folder>`. In the Admin App, go to **Apps → your app → the extension**, enable **Developer Mode** and enter `http://localhost:5173`. Append `?configUrl=<path-to-local-config>` to load a local config. For a Custom Editor, see the `showpad-app-editor-sdk` skill.
