# Showpad App manifest.json Reference

Complete schema for `manifest.json` — the control center of every Showpad App v2.

Source: https://developer.showpad.com/docs/appsv2/concepts/app-manifest  
JSON Schema: `https://schema.showpad.com/app/manifest.v2.json`

---

## What is a Showpad App?

A Showpad App is a bundle that contains one or more web applications which are rendered at specific extension points of the platform.

## Root structure

```json
{
  "$schema": "https://schema.showpad.com/app/manifest.v2.json",
  "schema": { "type": "app-manifest", "version": 2 },

  "appKey": "com.your-org.your-app",
  "name": "Your App",
  "version": "1.0.0",
  "description": {
    "short": "One-liner shown in overview",
    "full": "Longer description shown in the details section"
  },
  "icon": "icon.png",
  "images": ["images/screenshot-1.jpg", "images/screenshot-2.jpg"],
  "developer": {
    "name": "Your Company",
    "websiteUrl": "https://your-company.com"
  },

  "experienceTypes": [],
  "adminSettings": [],
  "userSettings": [],
  "addContentExtensions": [],
  "assetViewerExtensions": [],
  "sharedSpaceExtensions": [],

  "sharedStores": []
}
```

---

## Field reference

### Schema / identity (all required)

| Field            | Type   | Notes                                                                                                                                                                                                                           |
| ---------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `$schema`        | string | Must be `https://schema.showpad.com/app/manifest.v2.json`                                                                                                                                                                       |
| `schema.type`    | string | Must be `"app-manifest"`                                                                                                                                                                                                        |
| `schema.version` | number | Must be `2`                                                                                                                                                                                                                     |
| `appKey`         | string | Globally unique; lowercase `^[a-z][a-z0-9.-]+$` (e.g. `com.acme.my-app`). A new app can't start with `com.showpad.` (case-insensitive). Used as bundle filename prefix.                                                         |
| `name`           | string | Display name in the Showpad UI                                                                                                                                                                                                  |
| `version`        | string | `major[.minor[.bugfix]][-suffix]`, max 50 chars: `1`, `1.0`, `1.0.0` and `1.0.0-rc1` are all valid. The suffix is letters and digits only (`1.0.0-rc.1` is rejected). Each upload needs a new version; duplicates are rejected. |

### Description (required)

| Field               | Type   | Notes                                      |
| ------------------- | ------ | ------------------------------------------ |
| `description.short` | string | Brief summary, shown in app overview       |
| `description.full`  | string | Extended description, shown in app details |

### Media (optional)

| Field    | Type     | Notes                                                            |
| -------- | -------- | ---------------------------------------------------------------- |
| `icon`   | string   | Path to icon file (relative to manifest). Shown in app overview. |
| `images` | string[] | Up to 3 cover image paths. Shown in app details.                 |

### Developer (required)

| Field                  | Type   | Notes                                            |
| ---------------------- | ------ | ------------------------------------------------ |
| `developer.name`       | string | Your company/org name. Cannot contain "Showpad". |
| `developer.websiteUrl` | string | Optional URL to developer website                |

### Extension arrays (at least one required)

`experienceTypes`, `adminSettings`, `userSettings`, `addContentExtensions`, `assetViewerExtensions`, `sharedSpaceExtensions`. Each is an array, so an app can have several extensions of the same type. Per-type fields are in `extension-types.md`.

### AppsDB stores (optional)

```json
"sharedStores": [
  { "id": "my-store" }
]
```

- `id`: Store identifier, lowercase `^[a-z][a-z0-9.-]+$`. **Duplicate store IDs prevent app installation.**
- Usage, scopes and limits: `appsdb.md` in the `showpad-app-sdk` skill's `references/` folder.

---

## Versioning

- Use pre-release suffixes during development: `1.0.0-1`, `1.0.0-beta`, `1.0.0-rc1`. Only letters and digits after the `-`: no dots (`1.0.0-rc.1` is invalid)
- `1` and `1.0` are also accepted, but prefer full `major.minor.bugfix` for clarity
- Increment version before every bundle/upload — the same version cannot be uploaded twice
- The CLI validates the version format and checks for duplicates before uploading

---

## Full example: app with all three common extension types

```json
{
  "$schema": "https://schema.showpad.com/app/manifest.v2.json",
  "schema": { "type": "app-manifest", "version": 2 },
  "appKey": "com.acme.team-activities",
  "name": "Team Activities",
  "version": "1.0.0",
  "description": {
    "short": "Team management and activity proposals",
    "full": "Enables admins to create teams, users to join teams, and sales reps to propose activities via a custom Experience."
  },
  "icon": "icon.png",
  "images": ["images/screenshot-1.jpg"],
  "developer": {
    "name": "Acme Corp",
    "websiteUrl": "https://acme.com"
  },
  "adminSettings": [
    {
      "extensionKey": "team-activities-admin-settings",
      "name": "Teams",
      "description": "Setup your teams",
      "resources": { "folder": "admin-settings" }
    }
  ],
  "userSettings": [
    {
      "extensionKey": "team-activities-user-settings",
      "name": "My Team",
      "description": "Assign yourself to a team",
      "resources": { "folder": "user-settings" }
    }
  ],
  "experienceTypes": [
    {
      "extensionKey": "team-activities-experience-type",
      "name": "Team Activities",
      "description": "Propose your team activities",
      "resources": { "folder": "experience" }
    }
  ],
  "sharedStores": [{ "id": "teams-store" }, { "id": "team-activities-store" }]
}
```

For an experience type with a Custom Editor, add `"editorFolder": "<folder>"` to its `resources` (see `extension-types.md`).
