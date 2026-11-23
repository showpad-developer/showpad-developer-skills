---
name: showpad-app-editor-sdk
description: "Complete reference for the Showpad Experience Editor SDK (ShowpadEditor.* namespace) — setup, method signatures, config reference types, and a full editor example. Use when building a Custom Editor (editorFolder) for a Showpad App v2 experience type, or when code runs inside the Experience App Builder."
version: 1.1.0
---

# Showpad Experience Editor SDK Reference

Package: `@showpad/experience-editor-sdk`

A **Custom Editor** replaces the form that Showpad generates from `config.json` with your own webapp. It runs in an iframe inside the Experience App Builder and talks to it through `ShowpadEditor.*`. The config it produces can have any JSON shape.

- `ShowpadEditor.*` is available **only inside the `editorFolder` iframe**. It needs no initialization call.
- `Showpad.*` calls (`getConfig`, `getAssetsByIds`, `getAssetIdsByFilterId`) still require `await Showpad.onShowpadLibLoaded()` first. Invoke the `showpad-app-sdk` skill for their signatures.

> **Beta — feature flag required.** Custom Editors depend on `Showpad.getConfig`, `Showpad.getAssetsByIds` and `Showpad.getAssetIdsByFilterId`, which are beta and only work on organizations with the **`experience_app_custom_editors`** feature flag enabled by Showpad. Before building a Custom Editor, tell the user and ask them to confirm that the flag is enabled on every org the app will be assigned to. If not enabled, they must ask their Showpad account manager to enable it or use the Default Editor (`config.json` + `parseConfig`).

---

## Setup

Set `editorFolder` on the experience type in `manifest.json`:

```json
"experienceTypes": [
  {
    "extensionKey": "my-experience",
    "name": "My Experience",
    "description": "A curated content experience",
    "resources": { "folder": "experience", "editorFolder": "editor" }
  }
]
```

### Installation

```bash
npm install @showpad/experience-editor-sdk@^0.4.0
```

### Import & initialization

```typescript
import { Showpad } from "@showpad/experience-app-sdk";
import { ShowpadEditor } from "@showpad/experience-editor-sdk";
```

The editor folder is scaffolded like any other extension folder (`index.html`, `src/main.ts`, `vite.config.ts`, and its own `vite build` in the build script). The `showpad-app` skill has the project layout, build and bundle steps.

---

## Method signatures

All 9 `ShowpadEditor` methods are **beta** (`@beta` in the SDK): `emitConfig`, `registerEmitConfig`, `onSaved`, `onPublished`, `onError`, `selectAssets`, `createAssetFilter`, `editAssetFilter`, `getAssetFilterShowQl`. Their signatures may change in minor releases.

```typescript
// Config ↔ Experience App Builder
registerEmitConfig(getConfig: () => Record<string, unknown>): () => void; // returns a cleanup function
emitConfig(config: Record<string, unknown>): void;

// Experience App Builder events (return a cleanup function)
onSaved(callback: () => void): () => void;     // after each draft save (Save draft button, or auto-save when switching to Preview)
onPublished(callback: () => void): () => void; // after a successful publish
onError(callback: (error: ExperienceEditorError) => void): () => void; // a save or publish failed

interface ExperienceEditorError {
  action: "save" | "publish";
  message: string; // localized, user-facing
}

// Pickers and dialogs — resolve `undefined` when the admin cancels.
selectAssets(options?: SelectAssetsOptions): Promise<string[] | undefined>; // asset IDs, not Asset objects
createAssetFilter(): Promise<string | undefined>;                          // new filter ID
editAssetFilter(filterId: string): Promise<string | undefined>;            // possibly new filter ID — always store the returned one

// Asset filter as ShowQL (since 0.4.0) — no dialog opens
getAssetFilterShowQl(assetFilterId: string): Promise<string | undefined>;  // filter criteria as a ShowQL query, incl. ORDER BY if the filter is sorted

interface SelectAssetsOptions {
  limit?: number;            // max selectable; default unlimited; 1 = single-select
  assetTypes?: AssetType[];  // default: all types
}

type AssetType =
  | "3d-model" | "ai-roleplay" | "audio" | "brainshark-presentation" | "course"
  | "document" | "homepage" | "page" | "photo" | "raw" | "scorm" | "url" | "video" | "webapp";
```

### Save strategies

Pick one:

| Strategy                       | How it works                                                                                                  |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| `registerEmitConfig` (default) | The builder pulls the config when the admin saves. The callback must return the current config synchronously. |
| `emitConfig`                   | You push the config after every change (WYSIWYG or live preview).                                             |

```typescript
// On-demand (standard)
ShowpadEditor.registerEmitConfig(() => config);

// Eager push (only if you don't use `registerEmitConfig`)
inputEl.addEventListener("input", () => {
  config.title = inputEl.value;
  ShowpadEditor.emitConfig(config);
});
```

### Selecting assets

```typescript
// Any number of assets of any type
const assetIds = await ShowpadEditor.selectAssets();

// Single-select, restricted to some types
const [assetId] =
  (await ShowpadEditor.selectAssets({
    limit: 1,
    assetTypes: ["video", "photo", "document"],
  })) ?? [];
```

### Reading an asset filter as ShowQL

`getAssetFilterShowQl` returns the criteria of an existing asset filter as a ShowQL query string. When the filter is sorted, the query ends with an `ORDER BY` clause. It opens no dialog, so you can call it any time you have a filter ID, for example right after `createAssetFilter` / `editAssetFilter` or when restoring a saved config.

```typescript
const filterId = await ShowpadEditor.createAssetFilter();
if (filterId) {
  try {
    const showQl = await ShowpadEditor.getAssetFilterShowQl(filterId);
    if (showQl) {
      queryPreviewEl.textContent = showQl; // textContent, never innerHTML
    }
  } catch {
    status.textContent = "Could not load the filter query";
  }
}
```

In the config, keep the `FilterRef` (`{ $refType: "asset-filter", $refId }`) as the source of truth, and resolve assets in the experience with `Showpad.getAssetIdsByFilterId`. Treat the ShowQL string as derived data, for example a preview or a value the admin can inspect.

### Reacting to save, publish and errors

```typescript
const stopSaved = ShowpadEditor.onSaved(() => showSavedIndicator());
const stopPublished = ShowpadEditor.onPublished(() =>
  clearUnsavedChangesWarning(),
);
const stopError = ShowpadEditor.onError(({ action, message }) =>
  showError(action, message),
);
// Call stopSaved() / stopPublished() / stopError() to unsubscribe
```

---

## Config reference types

Store Showpad items in the config as references, never as full objects:

```typescript
type AssetRef = { $refType: "asset"; $refId: string }; // resolve with Showpad.getAssetsByIds([...])
type FilterRef = { $refType: "asset-filter"; $refId: string }; // resolve with Showpad.getAssetIdsByFilterId(id), then getAssetsByIds

// Example config shape
type MyConfig = {
  title: string;
  selectedAssets: AssetRef[];
  assetFilter: FilterRef | null;
};
```

### Reading the config in the experience (viewer)

The experience reads the editor's config with `Showpad.getConfig()`. It does **not** use `parseConfig()`, which is only for the Default Editor.

`getConfig()` resolves `undefined` until the admin has saved a config, so merge it over defaults.

```typescript
// experience/src/main.ts
import { Showpad } from "@showpad/experience-app-sdk";

type MyConfig = {
  selectedAssets: { $refType: "asset"; $refId: string }[];
  assetFilter: { $refType: "asset-filter"; $refId: string } | null;
};

const defaultConfig: MyConfig = { selectedAssets: [], assetFilter: null };

const main = async (): Promise<void> => {
  await Showpad.onShowpadLibLoaded();
  const savedConfig = await Showpad.getConfig<Partial<MyConfig>>(); // undefined when nothing is saved yet
  const config: MyConfig = { ...defaultConfig, ...savedConfig };

  if (!config.selectedAssets?.length && !config.assetFilter) {
    renderEmptyState(); // e.g. "This experience has no content yet"
    return;
  }

  const assets = config.selectedAssets?.length
    ? await Showpad.getAssetsByIds(config.selectedAssets.map((r) => r.$refId))
    : [];
  const filterAssets = config.assetFilter
    ? await Showpad.getAssetsByIds(
        await Showpad.getAssetIdsByFilterId(config.assetFilter.$refId),
      )
    : [];

  // enrichAssets() adds thumbnailUrl/fileUrl for rendering (see the showpad-app skill, "Asset cards")
  render(await Showpad.enrichAssets([...assets, ...filterAssets]));
};
main();
```

---

## Complete editor example

```typescript
// editor/src/main.ts
// index.html needs: #add-assets-btn, #add-filter-btn, #asset-list (a <ul>), #filter-preview, #status
import { Showpad } from "@showpad/experience-app-sdk";
import { ShowpadEditor } from "@showpad/experience-editor-sdk";

type MyConfig = {
  selectedAssets: { $refType: "asset"; $refId: string }[];
  assetFilter: { $refType: "asset-filter"; $refId: string } | null;
};

const defaultConfig: MyConfig = { selectedAssets: [], assetFilter: null };
let config: MyConfig = structuredClone(defaultConfig);
const status = document.getElementById("status")!;

const main = async (): Promise<void> => {
  await Showpad.onShowpadLibLoaded();
  const savedConfig = await Showpad.getConfig<Partial<MyConfig>>(); // undefined when nothing is saved yet
  config = { ...defaultConfig, ...savedConfig };

  ShowpadEditor.registerEmitConfig(() => config);

  if (config.selectedAssets.length) {
    renderAssets(
      await Showpad.getAssetsByIds(config.selectedAssets.map((r) => r.$refId)),
    );
  }
  if (config.assetFilter) {
    const ids = await Showpad.getAssetIdsByFilterId(config.assetFilter.$refId);
    renderFilterPreview(ids.length);
  }
};

document
  .getElementById("add-assets-btn")
  ?.addEventListener("click", async () => {
    try {
      const ids = await ShowpadEditor.selectAssets(); // e.g. { limit: 1, assetTypes: ["video"] }
      if (!ids?.length) return;
      ids.forEach((id) =>
        config.selectedAssets.push({ $refType: "asset", $refId: id }),
      );
      renderAssets(
        await Showpad.getAssetsByIds(
          config.selectedAssets.map((r) => r.$refId),
        ),
      );
    } catch {
      status.textContent = "Asset selection failed, please try again";
    }
  });

document
  .getElementById("add-filter-btn")
  ?.addEventListener("click", async () => {
    try {
      const filterId = config.assetFilter
        ? await ShowpadEditor.editAssetFilter(config.assetFilter.$refId)
        : await ShowpadEditor.createAssetFilter();
      if (!filterId) return;
      config.assetFilter = { $refType: "asset-filter", $refId: filterId };
      renderFilterPreview(
        (await Showpad.getAssetIdsByFilterId(filterId)).length,
      );
    } catch {
      status.textContent = "Filter update failed, please try again";
    }
  });

// Render asset data with textContent, never innerHTML
function renderAssets(assets: { name: string }[]): void {
  const list = document.getElementById("asset-list")!;
  list.replaceChildren(
    ...assets.map((asset) => {
      const item = document.createElement("li");
      item.textContent = asset.name;
      return item;
    }),
  );
}

function renderFilterPreview(count: number): void {
  document.getElementById("filter-preview")!.textContent =
    `Filter matches ${count} asset(s)`;
}

ShowpadEditor.onSaved(() => {
  status.textContent = "Draft saved";
});
ShowpadEditor.onPublished(() => {
  status.textContent = "Published";
});
ShowpadEditor.onError(({ message }) => {
  status.textContent = message;
});

main();
```

---

## WYSIWYG mode

Point `resources.folder` and `resources.editorFolder` at the **same folder**:

```json
"resources": { "folder": "experience", "editorFolder": "experience" }
```

The builder appends `?edit=true` to the iframe URL in edit mode, so one webapp can switch between viewing and editing:

```typescript
// experience/src/main.ts — shared experience + editor webapp
import { Showpad } from "@showpad/experience-app-sdk";
import { ShowpadEditor } from "@showpad/experience-editor-sdk";

const isEditMode =
  new URLSearchParams(window.location.search).get("edit") === "true";

type MyConfig = {
  selectedAssets: { $refType: "asset"; $refId: string }[];
  assetFilter: { $refType: "asset-filter"; $refId: string } | null;
};

const defaultConfig: MyConfig = { selectedAssets: [], assetFilter: null };

const main = async (): Promise<void> => {
  await Showpad.onShowpadLibLoaded();
  const savedConfig = await Showpad.getConfig<Partial<MyConfig>>(); // undefined when nothing is saved yet
  const config: MyConfig = { ...defaultConfig, ...savedConfig };

  if (isEditMode) {
    // Editor mode: ShowpadEditor.* is available
    ShowpadEditor.registerEmitConfig(() => config);
    renderEditorUI(config);
  } else {
    // Viewer mode: ShowpadEditor.* is NOT available
    renderViewerUI(config);
  }
};
main();
```

> **Important:** never call `ShowpadEditor.*` from the viewer path. It only works when the iframe is loaded with `?edit=true`.

With a shared folder there's only one `vite build` for both the experience and the editor.

---

## Local development

Run the editor's dev server. In the Experience App Builder's **Edit** tab, enable **Developer Mode** and enter `http://localhost:<port>`. To start from a saved config, append `?configUrl=<path-to-local-config.json>`.
