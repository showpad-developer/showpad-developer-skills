# Showpad Extension Types Reference

Each extension type is an entry point in the Showpad UI. Apps must include at least one.

Sources:

- https://developer.showpad.com/docs/appsv2/concepts/extension-types/experience-type
- https://developer.showpad.com/docs/appsv2/concepts/extension-types/admin-settings
- https://developer.showpad.com/docs/appsv2/concepts/extension-types/user-settings
- https://developer.showpad.com/docs/appsv2/concepts/extension-types

---

## Common fields (all extension types)

| Field              | Required | Description                                                       |
| ------------------ | -------- | ----------------------------------------------------------------- |
| `extensionKey`     | Yes      | Unique within the app; lowercase `^[a-z][a-z0-9.-]+$`             |
| `name`             | Yes      | Display name shown in the Showpad UI (max 50 chars)               |
| `description`      | Yes      | Brief explanation of what the extension does                      |
| `resources.folder` | Yes      | Directory (relative to manifest) containing the extension's files |
| `icon`             | No       | Path to an icon image shown in the Showpad UI for this extension  |

Every extension folder **must** contain `index.html` as the iframe entry point and can have other files (js, css, ...).

---

## 1. Experience Type (`experienceTypes`)

**Purpose:** Creates a new type of Showpad Experience. Appears in the "Create New Experience" dialog in the Online Platform.

**When to use:** You want to build a custom interactive content experience for sales reps and want to be able to edit the content via the Experience App Builder.

### Manifest

```json
"experienceTypes": [
  {
    "extensionKey": "my-experience",
    "name": "My Experience",
    "description": "An excellent custom experience",
    "resources": {
      "folder": "experience",
      "editorFolder": "editor"
    }
  }
]
```

| Field                    | Required | Description                                                                                                                                       |
| ------------------------ | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `resources.editorFolder` | No       | Folder with a Custom Editor webapp (`index.html`). Omit it to use the Default Editor (`config.json`). Can be the same as `folder` (WYSIWYG mode). |

### Required files

```text
experience/
├── index.html          # Entry point — loaded in iframe with ?configUrl=... appended
├── public/
│   └── config.json     # Configuration schema for admins to customize. Required only when no "editorFolder" is defined.
└── src/
    └── main.ts
```

### Default Editor

When the `editorFolder` is not defined, the experience content will be edited via the default editor generated from the `config.json` file.

#### config.json schema

Defines editable labels and content slots that admins configure in the Default Editor.

```json
{
  "version": 1,
  "labels": {
    "homepage": {
      "title": {
        "value": "Your Experience Title",
        "description": "Main heading shown to visitors",
        "limit": 60
      },
      "subtitle": {
        "value": "Subtitle text"
      }
    }
  },
  "contents": {
    "hero": {
      "slide": {
        "type": "asset",
        "description": "Hero banner asset"
      }
    },
    "featured": {
      "collection": {
        "type": "channel",
        "description": "Channel to feature on the homepage"
      }
    }
  }
}
```

##### Labels fields

| Field         | Required | Description                                      |
| ------------- | -------- | ------------------------------------------------ |
| `value`       | Yes      | Default text shown when no admin override is set |
| `description` | No       | Helper text shown to admins in the editor        |
| `limit`       | No       | Maximum character count enforced in the editor   |

Underscores in label keys are converted to spaces in the editor UI.

##### Content types

| Type        | Editor UI                                           |
| ----------- | --------------------------------------------------- |
| `asset`     | Searchable asset/URL selector                       |
| `channel`   | Channel selection dropdown                          |
| `folder`    | Text field — requires `showpad://folder/123` format |
| `page`      | Searchable page selector                            |
| `page-tags` | Tag search and combination interface (page scope)   |
| `tags`      | Tag search and combination interface                |
| `url`       | Text field — requires a valid URL                   |

> **Reserved words:** `type` and `value` cannot be used as config keys outside their designated roles.

### Custom Editor

When `editorFolder` is set, the admin edits the experience in your own webapp instead of the Default Editor, and `config.json` is not needed.

The config can have **any** JSON shape; Showpad items are stored as `$refType`/`$refId` references. For the config shape, the editor itself (Experience Editor SDK, save strategies, WYSIWYG mode with a shared `folder`/`editorFolder`, local testing) and how the experience reads the config, invoke the `showpad-app-editor-sdk` skill.

### Retrieve config at runtime

Showpad appends `?configUrl={path_to_config_file}&divisionId={division-id}` to the iframe URL. After `onShowpadLibLoaded()`:

| Editor  | Read config with                                                                                                | Example                                                         |
| ------- | --------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| Default | `Showpad.parseConfig()` or `parseEnrichedConfig()` (adds asset URLs)                                            | `showpad-app-sdk` skill, "Reading config in an Experience Type" |
| Custom  | `Showpad.getConfig()` (resolves `undefined` when unset, so merge over defaults) + resolve `$refType` references | `showpad-app-editor-sdk` skill                                  |

---

## 2. Admin Settings (`adminSettings`)

**Purpose:** Adds a section in the Showpad Apps area of the Admin Settings menu (Online Platform). Rendered as an iframe.

**When to use:** Admins need to configure the app (e.g. set up teams, manage app-level settings).

### Manifest

```json
"adminSettings": [
  {
    "extensionKey": "my-admin-settings",
    "name": "My App Admin",
    "description": "Configure settings for My App",
    "resources": { "folder": "admin-settings" }
  }
]
```

---

## 3. User Settings (`userSettings`)

**Purpose:** Adds a new section in the user's Settings menu (Web App). Rendered as an iframe.

**When to use:** Individual users need to configure personal preferences (e.g. select their team, connect personal accounts, set notification preferences).

### Manifest

```json
"userSettings": [
  {
    "extensionKey": "my-user-settings",
    "name": "My App",
    "description": "Personalize your My App experience",
    "resources": { "folder": "user-settings" }
  }
]
```

---

## 4. Add Content (`addContentExtensions`)

**Purpose:** Allows users to contribute content into Shared Spaces, Collections, and My Files.

**When to use:** You want users to be able to add custom or external content into Showpad content areas.

### Manifest

```json
"addContentExtensions": [
  {
    "extensionKey": "my-add-content",
    "name": "Add Custom Content",
    "description": "Add content from our system",
    "icon": "icon.png",
    "supportedContexts": ["sharedspace", "collection", "myfiles"],
    "resources": { "folder": "add-content" }
  }
]
```

**`supportedContexts`** (required, 1–3 unique values): where the extension appears. Valid values: `"sharedspace"`, `"collection"`, `"myfiles"`.

Add the chosen assets to wherever the user opened the extension with `Showpad.addAssetsToCurrentContext(assetSlugs)`, then `Showpad.closeExtension()`. To add a generated file, `Showpad.upload()` it first and use the slug of the uploaded asset.

---

## 5. Asset Viewer (`assetViewerExtensions`)

**Purpose:** Augments and enhances the asset viewing experience with additional information or interactivity.

**When to use:** You want to add custom panels, annotations, or metadata alongside asset viewing.

### Manifest

```json
"assetViewerExtensions": [
  {
    "extensionKey": "my-asset-viewer",
    "name": "Enhanced Viewer",
    "description": "View additional asset context",
    "icon": "icon.png",
    "resources": { "folder": "asset-viewer" }
  }
]
```

**`icon`** is required for this extension type.

---

## 6. Shared Space Extension (`sharedSpaceExtensions`)

**Purpose:** Adds Quick Actions capabilities within Shared Spaces (the secure collaborative spaces for sharing content with external contacts).

**When to use:** You want to add custom actions that users can trigger from within a Shared Space.

### Manifest

```json
"sharedSpaceExtensions": [
  {
    "extensionKey": "my-shared-space",
    "name": "My Quick Actions",
    "description": "Custom actions for Shared Spaces",
    "icon": "icon.png",
    "validDomains": ["your-domain.com"],
    "resources": { "folder": "shared-space" }
  }
]
```

**`validDomains`** (required, 1–10 hostnames, `*` allowed for subdomains e.g. `*.your-domain.com`): allowlist of external domains the extension is permitted to load or communicate with.
