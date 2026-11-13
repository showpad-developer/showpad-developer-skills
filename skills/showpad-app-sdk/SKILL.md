---
name: showpad-app-sdk
description: "Complete SDK reference for @showpad/experience-app-sdk — method signatures, types, AppsDB, and usage patterns. Use when developing Showpad App v2 extensions, answering questions about SDK methods, or generating code that calls the Experience App SDK."
version: 1.1.0
---

# Showpad Experience App SDK Reference

Package: `@showpad/experience-app-sdk` (v3.5.0)  
Types package: `@showpad/experience-app-types` (v0.3.0)

Source: https://developer.showpad.com/docs/appsv2/references/sdk/typedoc

---

## Installation

```bash
npm install @showpad/experience-app-sdk@^3.5.0
```

## Import & initialization

**Always await `onShowpadLibLoaded()` before calling any other SDK method.**

```typescript
import { Showpad } from "@showpad/experience-app-sdk";

const main = async (): Promise<void> => {
  await Showpad.onShowpadLibLoaded();
  // safe to call SDK methods now
};
main();
```

---

## Method signatures

Methods are grouped by category. All are called as `Showpad.<method>(...)`.

### Initialization

```typescript
// Must complete before any other SDK call.
onShowpadLibLoaded(): Promise<boolean>

// SDK internal version (integer).
getVersion(): Promise<number>

// Where the app runs. Synchronous.
getAppContext(): "production" | "external"
```

### User & device info — Offline

```typescript
getUserInfo(): Promise<UserInfo>
getDeviceInfo(): Promise<DeviceInfo>
isAdmin(): Promise<boolean>
hasFeature(feature: Feature): Promise<boolean>
```

### Asset retrieval — Offline

```typescript
getAssetsByTags(tags: string[]): Promise<Asset[]>
getAssetsInFolder(folder: string): Promise<Asset[]>
getAssetsByQuery(query: string): Promise<Asset[]>     // matches displayName LIKE "%query%"
```

### Asset URLs — Offline

These return unauthenticated URLs valid within Showpad apps.

```typescript
getAssetFileUrl(assetId: Asset['id'], assetSlug: Asset['slug']): Promise<string>
getAssetPreviewUrl(assetId: Asset['id'], assetSlug: Asset['slug'], size: number): Promise<string>
getAssetThumbnailUrl(assetId: Asset['id'], assetSlug: Asset['slug'], size: number): Promise<string>
```

### Asset Blobs — Offline

Returns raw binary data as `Blob`.

```typescript
getAssetFile(assetId: Asset['id'], assetSlug: Asset['slug']): Promise<Blob>
getAssetPreview(assetId: Asset['id'], assetSlug: Asset['slug'], size: number): Promise<Blob>
getAssetThumbnail(assetId: Asset['id'], assetSlug: Asset['slug'], size: number): Promise<Blob>
```

### Asset enrichment — Offline

Enriches assets with `fileUrl`, `previewUrl`, and `thumbnailUrl` fields.

```typescript
enrichAssets(assets: Asset[], size?: number): Promise<EnrichedAsset[]>
```

### Collections — Offline

```typescript
getCollections(): Promise<Collection[]>
createCollection(name: Collection['name']): Promise<Collection['id']>
openCollection(collectionId: Collection['id']): Promise<void>           // opens modal
clearCollection(collectionId: Collection['id']): Promise<void>

addAssetsToCollection(assetSlugs: Asset['slug'][]): Promise<Collection['id']>
addAssetsToCollections(assetSlugs: Asset['slug'][]): Promise<Collection['id'][]>
addAssetsToCollectionWithId(collectionId: Collection['id'], assetSlugs: Asset['slug'][]): Promise<Collection['id']>
```

### Sharing — Offline

```typescript
share(type: ShareType, assetSlugs: Asset['slug'][]): Promise<ShareResult>
// type: 'email' | 'link'
// result: 'success' | 'partial' | 'error'
```

### Shared Spaces — Online

```typescript
addAssetsToSharedSpaces(assetSlugs: Asset['slug'][]): Promise<SharedSpace['id'][]>
```

### Add to current context — Online

```typescript
addAssetsToCurrentContext(assetSlugs: Asset['slug'][]): Promise<Asset['slug'][]>
```

### Upload — Offline

```typescript
upload(uploadData: UploadData): Promise<StatusEmitter>

// StatusEmitter events: queued | uploading | processing | success | failed
emitter.on('queued', () => void)
emitter.on('uploading', (data: UploadProgress) => void)
emitter.on('processing', () => void)
emitter.on('success', ({ asset }: { asset: Asset }) => void)
emitter.on('failed', (error: UploadError) => void)
```

### UI — Offline

```typescript
displayModal(modal: Modal): Promise<ModalReason>
displayToast(toast: Toast): Promise<ToastReason>
handleErrorWithToast(error: unknown): Promise<ToastReason>  // displays error message as toast
```

### Navigation — Offline

```typescript
openShowpadUrl(showpadUrl: string): void
openAssetViewer(assetSlug: Asset['slug'], useModal?: boolean): void
closeExtension(): void
```

### Analytics & tracking — Offline

```typescript
trackEvent(event: AnyEvent | AnyEvent[]): void
trackExperiencePageview(touchpoint: PageViewTouchpoint): void
trackExperienceEvent(touchpoint: EventTouchpoint): void
```

### Config (Experience Types) — Offline

```typescript
// Default editor (config.json-based)
parseConfig<T = ConfigJSON>(options?: ParseConfigOptions): Promise<T>
parseEnrichedConfig<T extends EnrichedConfigJSON = EnrichedConfigJSON>(options?: ParseConfigOptions): Promise<T>
// options.localLabels: boolean — apply local labels in production (default: false in v3+)

// Custom editor only — call after onShowpadLibLoaded()
getConfig<T = unknown>(): Promise<T | undefined> // undefined when no configUrl is set; no fallback to ./config.json
getAssetsByIds(ids: string[]): Promise<Asset[]>
getAssetIdsByFilterId(filterId: string): Promise<string[]>
```

### Showpad API access — Online

```typescript
getShowpadApi(): Promise<ApiConfig>
refreshShowpadApi(): Promise<ApiConfig>
requestOnShowpadInstance<T = any, R = AxiosResponse<T>, D = any>(config: AxiosRequestConfig<D>): Promise<R>
```

### Salesforce API access — Online

```typescript
getSalesforceApi(): Promise<ApiConfig>
refreshSalesforceApi(): Promise<ApiConfig>
// Note: may open login popup on web; some browsers block popups → promise never resolves
```

### OAuth API access — Online

Used to authenticate as an integration user, e.g. to write GLOBAL AppsDB entries when the current user isn't an admin.

```typescript
getShowpadOAuthApi(
  clientId: string,
  clientSecret: string,
  allowedEmails: string[],
  password: string
): Promise<ApiConfig>

getShowpadOAuthApiInteractive(
  clientId: string,
  clientSecret: string,
  allowedEmails: string[],
  passwordModalConfig?: PasswordModalConfig
): Promise<ApiConfig>

refreshShowpadOAuthApi(clientId: string, clientSecret: string): Promise<ApiConfig>
```

### Axios instance helpers — Online

Pre-configured Axios instances with automatic token refresh.

```typescript
type ShowpadApi = "v3" | "v4" | "coach";

getShowpadInstance(api?: ShowpadApi, customToken?: string): Promise<AxiosInstance>

getShowpadOAuthInstance(
  clientId: string,
  clientSecret: string,
  allowedEmails: string[],
  password: string,
  api?: ShowpadApi
): Promise<AxiosInstance>

getShowpadOAuthInstanceInteractive(
  clientId: string,
  clientSecret: string,
  allowedEmails: string[],
  api?: ShowpadApi,                        // pass undefined to reach passwordModalConfig
  passwordModalConfig?: PasswordModalConfig
): Promise<AxiosInstance>

getSalesforceInstance(proxyUrl?: string): Promise<AxiosInstance>

getApiInstance(
  name: string,
  getApi: () => Promise<ApiConfig>,
  refreshApi: () => Promise<ApiConfig>,
  proxyUrl?: string
): Promise<AxiosInstance>
```

On Android, asset file URLs use the `showpad://` scheme, which Axios can't fetch. Use `getAssetFile()` instead.

---

## AppsDB methods

AppsDB methods are direct exports on `Showpad` — there is no intermediate store object. For scopes, value handling, limits, security rules and examples, read `references/appsdb.md`.

### User-scoped (USER scope)

```typescript
getStoreEntries<T extends JsonValue = JsonValue>(store: string): Promise<AppsDbEntry<T>[]>
getStoreEntryValue<T extends JsonValue = JsonValue>(store: string, entryId: string): Promise<T>
setStoreEntryValue(store: string, entryId: string, entryValue: Scalar): Promise<Scalar>
deleteStoreEntry(store: string, entryId: string): Promise<void>
```

### Global-scoped (GLOBAL scope — omit `authentication` only when the current user is an admin)

```typescript
getGlobalStoreEntries<T extends JsonValue = JsonValue>(store: string): Promise<AppsDbEntry<T>[]>
getGlobalStoreEntryValue<T extends JsonValue = JsonValue>(store: string, entryId: string): Promise<T>
setGlobalStoreEntryValue(store: string, entryId: string, entryValue: Scalar, authentication?: AxiosInstance | string): Promise<Scalar>
deleteGlobalStoreEntry(store: string, entryId: string, authentication?: AxiosInstance | string): Promise<void>
```

### Store provisioning (admin use)

```typescript
createStore(name: string, authentication?: AxiosInstance | string): Promise<void>
```

---

## Key types

For `UserInfo`, `Asset`, `Toast`/`Modal`, `ConfigJSON`, `AppsDbEntry`, touchpoints and other SDK types, read `references/types.md`.

---

## Common patterns

### Guard pattern (check feature before using it)

```typescript
const hasCollections = await Showpad.hasFeature("collections");
if (hasCollections) {
  await Showpad.addAssetsToCollection(slugs);
}
```

### Reading config in an Experience Type (Default Editor)

The admin's values replace the defaults from `config.json`. Label leaves are `{ value }`. Asset, page and tag content slots get a `result` array of asset IDs, which you look up in `config.assets`:

```typescript
// Mirror your config.json shape
type MyConfig = Showpad.EnrichedConfigJSON & {
  labels: { homepage: { title: { value: string } } };
  contents: { hero: { slide: { result?: string[] } } };
};

const config = await Showpad.parseEnrichedConfig<MyConfig>(); // parseConfig() if you don't need asset URLs

const title = config.labels.homepage.title.value;
const heroAssets = (config.contents.hero.slide.result ?? []).map(
  (id: string) => config.assets[id], // EnrichedAsset: .fileUrl / .previewUrl / .thumbnailUrl
);
```

For a Custom Editor, read the config with `getConfig()` instead (resolves `undefined` when no config is set). The `showpad-app-editor-sdk` skill shows how to default it and resolve its `$refType` references.

### Calling the Showpad API directly

```typescript
const api = await Showpad.getShowpadInstance("v3"); // baseURL https://<sub>.showpad.<tld>/api/v3
const { data } = await api.get("users/me.json"); // relative path, no leading /api
```

### Upload a generated file

```typescript
const blob = new Blob([content], { type: "application/pdf" });
const emitter = await Showpad.upload({ file: blob, filename: "report.pdf" });

emitter.on("success", ({ asset }) => console.log("uploaded", asset.slug));
emitter.on("failed", ({ reason }) => console.error(reason));
```

### Error handling with toast

```typescript
try {
  await Showpad.share("email", slugs);
} catch (err) {
  await Showpad.handleErrorWithToast(err);
}
```

---

## Memoization

Several methods are memoized (results cached for the lifetime of the page):
`getAppContext`, `getVersion`, `getUserInfo`, `getDeviceInfo`, `hasFeature`, `isAdmin`,
`parseConfig`, `parseEnrichedConfig`.

`parseConfig` and `parseEnrichedConfig` return the config loaded at startup. A config saved later is only seen after the iframe reloads. `getConfig` is not memoized: each call fetches the config again.

Call the corresponding refresh method (`refreshShowpadApi`, `refreshShowpadOAuthApi`, `refreshSalesforceApi`) to invalidate tokens, not these memoized methods.

---

## Platform notes

The Windows desktop app hosts the web app: `getDeviceInfo()` reports `app: "web"` there, so you can't tell Windows from a browser. Treat it as Web.

| Method               | Web (incl. Windows app)             | iOS                                    | Android                             |
| -------------------- | ----------------------------------- | -------------------------------------- | ----------------------------------- |
| `upload`             | online only                         | queued offline, syncs when back online | queued offline                      |
| `getSalesforceApi`   | stored token or rejects `'expired'` | ✓                                      | ✓                                   |
| `getShowpadInstance` | ✓                                   | ✓                                      | can't fetch `showpad://` asset URLs |
