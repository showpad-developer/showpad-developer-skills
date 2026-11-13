# Key Types Reference

TypeScript types used by `@showpad/experience-app-sdk` method signatures.

Source: https://developer.showpad.com/docs/appsv2/references/sdk/typedoc

```typescript
// User & device
interface UserInfo {
  id: string;
  email: string;
  userName: string;
  fullName: string;
}
interface DeviceInfo {
  locale: LanguageCode;
  app: "web" | "ios" | "android"; // the Windows desktop app also reports "web"
}

// Asset
type AssetType =
  "document" | "photo" | "url" | "page" | "video" | "raw" | "audio" | "icon" | "webapp";
interface AssetPermissions {
  annotate: boolean;
  download: boolean;
  share: boolean;
}
interface Asset {
  id: string;
  slug: string;
  name: string;
  displayName: string;
  description: string | null;
  type: AssetType;
  permissions: AssetPermissions;
  createdAt: Date;
  updatedAt: Date;
  expiresAt: Date | null;
}
interface EnrichedAsset extends Asset {
  fileUrl: string;
  previewUrl: string;
  thumbnailUrl: string;
}

// Feature flags
type Feature = "shareEmail" | "shareLink" | "collections" | "sharedSpaces";

// Collections & spaces
interface Collection {
  id: string;
  name: string;
}
interface SharedSpace {
  id: string;
}

// Sharing
type ShareType = "email" | "link";
type ShareResult = "success" | "partial" | "error";

// UI
interface Toast {
  type: "info" | "success" | "error";
  text: string;
  actionText?: string;
}
type ToastReason = "timeout" | "clicked";
interface Modal {
  title: string;
  text: string;
  buttons: ModalButton[];
}
interface ModalButton {
  reason: ModalReason;
  text: string;
  tinted: boolean;
}
type ModalReason = "cancel" | string;

// Upload
interface UploadData {
  file: File | Blob;
  filename: string;
}
interface UploadProgress {
  bytesTotal: number;
  bytesSent: number;
}
interface UploadError {
  reason: string;
}

// Config (Experience Types)
interface ConfigJSON {
  labels: Record<string, Labels>;
  contents: Record<string, Contents>;
  assets: Record<Asset["id"], Asset>;
}
interface EnrichedConfigJSON {
  labels: Record<string, Labels>;
  contents: Record<string, Contents>;
  assets: Record<EnrichedAsset["id"], EnrichedAsset>;
}
interface ParseConfigOptions {
  localLabels?: boolean;
}

// OAuth UI
interface PasswordModalConfig {
  description?: string;
  placeholder?: string;
  buttonText?: string;
  containerStyle?: string;
  descriptionStyle?: string;
  formStyle?: string;
  inputStyle?: string;
  buttonStyle?: string;
}

// API
type ApiConfig = {
  accessToken: string;
  url: string; // platform URL, e.g. https://acme.showpad.biz
  scope?: string[];
};

// AppsDB
type AppsDbScope = "user" | "global";
interface AppsDbEntry<T> {
  id: string;
  storeId: string;
  value: T;
}
type Scalar = JsonValue; // string | number | boolean | null | Scalar[] | { [key: string]: Scalar }
type ToScalar<O extends object> = {
  [P in keyof O]: O[P] extends object ? ToScalar<O[P]> : O[P];
};
// Use ToScalar<MyInterface> when TypeScript complains about index signatures

// Analytics
interface PageViewTouchpoint {
  id: string;
  name?: string;
  sourceId: string;
  sourceName?: string;
  destinationId: string;
  destinationName?: string;
  fields?: { pageviewDepth: string; pageviewType: string };
}
interface EventTouchpoint {
  id: string;
  name?: string;
  sourceId: string;
  sourceName?: string;
  fields?: {
    eventCategory: string;
    eventAction: string;
    eventLabel: string;
    eventValue: number;
  };
}
```
