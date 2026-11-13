# Showpad App Error Reference

Errors encountered during upload, bundling, and validation.

Source: https://developer.showpad.com/docs/appsv2/references/errors

---

## Upload errors (apps-api problem types)

Apps v2 uploads are processed by apps-api. When processing fails, the upload goes to status `FAILED` and its `error` is an RFC 7807 problem:

```json
{
  "type": "/problems/invalid-manifest",
  "title": "Bad Request.",
  "status": 400,
  "detail": "Invalid manifest",
  "meta": [{ "name": "...", "reason": "...", "category": "..." }]
}
```

`npx showpad apps upload` prints `Could not upload app. Reason: <detail>: <meta as JSON>`. To see the `type`, run `npx showpad apps status --upload-id <uploadId> --output json`. Read `meta[].reason` first: it names the exact file, field or key at fault.

> The legacy Apps v1 uploader codes (`invalid-version`, `no-index-found`, `manifest-not-found`, `config-invalid-format`, …) are **not** returned for Apps v2. Don't diagnose with them.

| `type`                                 | `detail` (as printed by the CLI)                                       | Cause and fix                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| -------------------------------------- | ---------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/problems/app-version-already-exists` | `App version already exists`                                           | This `appKey` already has this `version`. Bump `version` in `manifest.json` (e.g. `1.0.0-2`), rebuild and rebundle. The CLI usually catches it before uploading: "App with specified version is already uploaded."                                                                                                                                                                                                                            |
| `/problems/invalid-manifest`           | `Invalid manifest` / `Invalid manifest v2.` / `Invalid manifest json.` | The manifest fails the v2 JSON schema, isn't valid JSON (`meta[].category` `invalid-json`), or breaks a content rule. Check `meta[].category`: `empty-manifest` (no extensions declared), `not-unique` (duplicate `extensionKey` or store `id`), `ValidationError` (`appKey` starts with `com.showpad.` or `developer.name` contains "showpad"), or a schema keyword (`required`, `pattern`, `enum`, …) with the field path in `meta[].name`. |
| `/problems/invalid-bundle-content`     | `Invalid bundle content`                                               | A file the manifest points to is missing from the bundle (`meta[].category` `not-found`, reason `File not found in bundle: <path>`), or a `resources.folder` value is invalid (`invalid-manifest`). Every extension folder needs `<folder>/index.html`; `icon` and `images` paths must exist; experience types without an `editorFolder` are also checked for `<folder>/config.json`. Fix the build output or the prebundle copy step.                           |
| `/problems/unknown-manifest`           | `Could not determine manifest version.`                                | The JSON parses but is neither a v1 nor a v2 manifest. Check that `manifest.json` has `"schema": { "type": "app-manifest", "version": 2 }` and the v2 `$schema` URL (see `manifest-reference.md`).                                                                                                                                                                                                                                            |
| `/problems/invalid-app-key`            | `App key does not match.`                                              | You uploaded a new version of an existing app with a different `appKey`. Keep the `appKey` identical across versions; use a new key only for a new app.                                                                                                                                                                                                                                                                                       |
| `/problems/app-key-in-use`             | `App key is already in use.`                                           | Another app (possibly another developer org) owns this `appKey`. Pick a different, globally unique key.                                                                                                                                                                                                                                                                                                                                       |
| `/problems/unauthorized`               | `Authorization is either invalid or expired.`                          | Run `npx showpad login` again (or check the CI env vars).                                                                                                                                                                                                                                                                                                                                                                                     |
| `/problems/forbidden`                  | `Forbidden to execute this action`                                     | The logged-in user lacks the rights for this app or developer org. Log in as an admin of the developer org that owns the app.                                                                                                                                                                                                                                                                                                                 |
| `/problems/internal-server-error`      | —                                                                      | Server-side failure, e.g. a bundle that couldn't be read (re-create it with `npx showpad apps bundle` instead of zipping by hand). Retry; if it persists, report the `uploadId`.                                                                                                                                                                                                                                                              |

## CLI-side errors (before anything is uploaded)

| Message                                                               | Fix                                                                                                                    |
| --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `Manifest does not exist or is inaccessible in source directory "…"`  | `manifest.json` must be at the root of the bundled folder: add `cp manifest.json dist` to `prebundle`.                 |
| `Manifest is missing required properties: appKey or version`          | Add both to `manifest.json`.                                                                                           |
| `Manifest has no link to schema`                                      | Add `"$schema": "https://schema.showpad.com/app/manifest.v2.json"` to `manifest.json`.                                 |
| `App with specified version is already uploaded.`                     | Bump `version` in `manifest.json` and rebundle.                                                                        |
| `The bundle exceeds the max filesize` / `File size exceeds 1GB limit` | Hard limit (4 GB with the `showpad_apps_size_limit_exempt` flag). Shrink the bundle (drop source maps, unused assets). |
| `The App has not become active after N attempts.`                     | Processing is still running. Poll with `npx showpad apps status --upload-id <uploadId>`.                               |

---

## CLI schema validation errors

`showpad apps bundle` and `showpad apps upload` validate `manifest.json` against `https://schema.showpad.com/app/manifest.v2.json` and fail with "The manifest trying to bundle is invalid" / "Manifest of the bundle to upload is invalid". Common causes:

| Error detail                                                     | Fix                                                                                                                     |
| ---------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `supportedContexts` `must be equal to one of the allowed values` | Use `sharedspace`, `collection`, `myfiles`. The field is required (at least one value).                                 |
| `must match pattern "^[a-z][a-z0-9.-]+$"`                        | `appKey`, `extensionKey` and `sharedStores[].id` must be lowercase letters, digits, `.` or `-`, starting with a letter. |
| `must have required property 'icon'` / `'validDomains'`          | `assetViewerExtensions` require `icon`; `sharedSpaceExtensions` require `validDomains`.                                 |

## Common issues and fixes

### Extension builds land in wrong folder

The `resources.folder` in manifest.json must match the folder name inside `dist/`. Set each extension's Vite `build.outDir` to the root `dist/<folder>` (not `<folder>/dist`):

```typescript
// admin-settings/vite.config.ts
export default defineConfig({
  base: "./",
  build: { outDir: "../dist/admin-settings", emptyOutDir: true },
});
```

### Upload times out (status stays WAITING_FOR_UPLOAD or PROCESSING)

Normal for large bundles. The CLI will print:

```text
Upload ID: abc-123
Run: npx showpad apps status --upload-id abc-123
```

Poll manually until status is `ACTIVE`.

### App not appearing after assignment

Assignments can take several minutes to propagate. If the app still doesn't appear after 10 minutes, check:

1. The version was assigned (not just uploaded)
2. The admin has installed/enabled it in the Admin App
3. The user's organization matches the assignment
