# AppsDB Reference

AppsDB is Showpad's built-in key/value store for Showpad Apps. It provides offline storage, cross-platform support, and data protection.

Source: https://developer.showpad.com/docs/appsv2/developer-tools/appsdb

---

## Core concepts

### Store

The top-level container that groups related entries. Declare every store in `manifest.json` under `sharedStores` before using it (see `manifest-reference.md` in the `showpad-app` skill's `references/` folder).

### Store Entry

Individual key/value objects within a Store:

- Identified by an `id`
- Holds a value (stored as a string; the SDK serializes JSON for you, see below)
- Has a `scope` (USER or GLOBAL)

### Scopes

| Scope    | Who can access                            | Who can create/write                                                                                            |
| -------- | ----------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `USER`   | The specific user (data is user-isolated) | Any authenticated user                                                                                          |
| `GLOBAL` | All users in the organization             | Admin users, or any user via an OAuth instance/token of a requester with the `appsdb_online_integrations` scope |

Use `USER` scope for personal preferences (e.g. "which team am I on").  
Use `GLOBAL` scope for org-wide data (e.g. "list of all teams").

---

## SDK usage

Access AppsDB via the Experience App SDK — do not call the REST API directly from frontend code. Exact signatures are in the AppsDB section of this skill's `SKILL.md`.

### User-scoped entries

```typescript
import { Showpad } from "@showpad/experience-app-sdk";

// Write (create or update)
await Showpad.setStoreEntryValue("teams-store", "entry-id", {
  name: "Alpha Team",
});

// Read a specific entry
const value = await Showpad.getStoreEntryValue<{ name: string }>(
  "teams-store",
  "entry-id",
);

// List all entries — each has { id, storeId, value }
const entries = await Showpad.getStoreEntries<{ name: string }>("teams-store");

// Delete (irreversible)
await Showpad.deleteStoreEntry("teams-store", "entry-id");
```

> **Security warning**
>
> - **Do NOT store credentials or PII** (passwords, API keys, OAuth tokens, secrets of any kind, names, email addresses, identifiers, or any data covered by GDPR/CCPA) in AppsDB. Use a dedicated secrets manager or backend service instead.

### Global-scoped entries

Reads are open to all users. Writes and deletes take an optional `authentication` argument (OAuth `AxiosInstance` or access token). Omit it only when the current user is an admin. That is typical in `adminSettings`, but don't assume it in an extension non-admins can open.

```typescript
// Read a specific global entry
const value = await Showpad.getGlobalStoreEntryValue<{ label: string }>(
  "global-store",
  "entry-id",
);

// List all global entries
const entries = await Showpad.getGlobalStoreEntries<{ label: string }>(
  "global-store",
);

// Write as a non-admin user (requires an OAuth instance).
// clientSecret ships in the bundle: prefer admin-only writes or a backend when possible.
const instance = await Showpad.getShowpadOAuthInstanceInteractive(
  clientId,
  clientSecret,
  allowedEmails,
);
await Showpad.setGlobalStoreEntryValue(
  "global-store",
  "entry-id",
  { label: "Foo" },
  instance,
);

// Delete (requires OAuth instance, irreversible)
await Showpad.deleteGlobalStoreEntry("global-store", "entry-id", instance);
```

Values are `Scalar` (= any JSON value). Objects and arrays are stringified automatically on write, and parsed automatically on read: pass the expected type as the generic (`getStoreEntryValue<Team>`) and **don't call `JSON.parse`** on the result. Declare stored shapes with `type`, not `interface`, because interfaces aren't assignable to `JsonValue`.

> **Strings that look like JSON come back converted.** Every value is stored as a string, and on read the SDK JSON-parses any string it can. So storing the string `"123"` reads back as the number `123`, `"true"`/`"false"` as booleans, and `"null"` as `null`. The generic is only a type hint and won't stop this. To keep such values as strings, wrap them in an object (`{ code: "123" }`), or convert on read (`String(value)`).

Global-scope **writes** (`setGlobalStoreEntryValue`, `deleteGlobalStoreEntry`) and `createStore` are Online: they need a connection. USER-scope reads/writes and global reads work offline.

---

## Offline use

For offline scenarios, consolidate related data into a **single store entry** (as a serialized JSON object) rather than many separate entries. This reduces sync overhead and improves offline reliability.

```typescript
// Instead of: one entry per team member
// Do: one entry for all teams
await Showpad.setStoreEntryValue("teams-store", "all-teams", {
  teams: allTeams,
});
```

---

## Known limitations

| Constraint                     | Value                                                                                |
| ------------------------------ | ------------------------------------------------------------------------------------ |
| Max data per entry (direct)    | 250,000 bytes (250KB)                                                                |
| Max data per entry (via SDK)   | 10MB — SDK auto-chunks into 250KB segments                                           |
| Sync — web (incl. Windows app) | Local writes are pushed after ~1 s; changes from other devices are pulled every 60 s |
| Sync — iOS / Android           | Background sync about every 5 minutes                                                |
| Concurrent writes              | Not supported — last write wins; offline edits may overwrite online versions         |
| Performance                    | Optimized for small objects — large entries degrade performance                      |

---

## Full example: team management

Teams are shared with everyone, so they live in GLOBAL scope (written by admins from `adminSettings`). Each user's choice is private, so it lives in USER scope. That way no user identifier needs to be stored.

```typescript
import { Showpad } from "@showpad/experience-app-sdk";

type Team = { id: string; name: string };

// adminSettings: create a team. The user is an admin, so no OAuth instance is needed.
async function createTeam(name: string): Promise<void> {
  const team: Team = { id: crypto.randomUUID(), name };
  await Showpad.setGlobalStoreEntryValue("teams-store", team.id, team);
}

// Any extension: list all teams
async function listTeams(): Promise<Team[]> {
  const entries = await Showpad.getGlobalStoreEntries<Team>("teams-store");
  return entries.map((entry) => entry.value);
}

// userSettings: remember the current user's team (USER scope is isolated per user)
async function joinTeam(teamId: string): Promise<void> {
  await Showpad.setStoreEntryValue("teams-store", "my-team", teamId);
}
```
