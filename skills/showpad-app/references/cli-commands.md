# Showpad CLI Reference

Source: https://developer.showpad.com/docs/appsv2/developer-tools/cli  
Commands reference: https://developer.showpad.com/docs/appsv2/references/cli/cli-commands

---

## Installation

```bash
npm install -D @showpad/cli
# or
yarn add -D @showpad/cli
```

**Requires Node.js 20+**

Run via npx: `npx showpad <command>`  
Or add to package.json scripts: `"bundle": "showpad apps bundle"`

---

## Authentication

### `showpad login`

Generates credentials in `.showpad/` directory for authenticated commands.

```bash
npx showpad login
```

**Flags:**

- `-p, --password=<value>` — password (or use env var)
- `-c, --config-dir=<value>` — custom config directory
- `--no-input` — non-interactive mode
- `--verbose` — verbose output

**Environment variables (non-interactive / CI):**

| Variable                 | Description                                                 |
| ------------------------ | ----------------------------------------------------------- |
| `SHOWPAD_SUBDOMAIN`      | Your Showpad subdomain (e.g. `acme` for `acme.showpad.com`) |
| `SHOWPAD_ADMIN_USERNAME` | Admin username / email                                      |
| `SHOWPAD_ADMIN_PASSWORD` | Admin password                                              |
| `SHOWPAD_CLIENT_ID`      | OAuth client ID                                             |
| `SHOWPAD_CLIENT_SECRET`  | OAuth client secret                                         |

For CI/CD pipelines, set these env vars instead of running `login` interactively. Store the password and client secret in your CI secret store, never in the repo.

---

## App commands (`showpad apps <command>`)

All commands below are prefixed with `showpad apps`.

---

### `bundle`

Creates a `.showpad` bundle file from a source directory.

```bash
npx showpad apps bundle
npx showpad apps bundle -s dist -o .
```

**Flags:**

| Flag                      | Default | Description                                     |
| ------------------------- | ------- | ----------------------------------------------- |
| `-s, --source-dir=<path>` | `.`     | Directory to bundle (your prepared dist folder) |
| `-o, --output-dir=<path>` | `.`     | Where to write the `.showpad` file              |
| `--skip-validation`       | —       | Skip manifest validation                        |
| `-c, --config-dir=<path>` | —       | Custom config directory                         |
| `--no-input`              | —       | Non-interactive mode                            |
| `--verbose`               | —       | Verbose output                                  |

**What it does:**

1. Validates `manifest.json` format and contents
2. Checks bundle size: over 1 GB it **fails** and deletes the bundle.
3. Creates `<appKey>.<version>.showpad` file

**Output filename format:** `com.your-company.your-app.1.0.0.showpad`

---

### `upload`

Uploads a `.showpad` bundle to Showpad.

```bash
npx showpad apps upload ./com.your-company.your-app.1.0.0.showpad
```

**Arguments:** `FILEPATH` — path to the `.showpad` file

**Flags:**

| Flag                | Description                |
| ------------------- | -------------------------- |
| `--skip-validation` | Skip pre-upload validation |
| `-c, --config-dir`  | Custom config directory    |
| `--no-input`        | Non-interactive mode       |
| `--verbose`         | Verbose output             |

**Behavior:**

- Checks if `appKey` exists → uploads as new version if yes, new app if no
- Polls upload status up to 10 times
- On timeout: prints `uploadId` and the manual status command

**Updating an existing app:** Just upload a bundle with the same `appKey` and a new `version`.

---

### `status`

Check the processing status of an uploaded bundle.

```bash
npx showpad apps status --upload-id <uploadId>
npx showpad apps status --app-id <appId>
```

**Flags:**

| Flag                       | Description                      |
| -------------------------- | -------------------------------- |
| `--upload-id=<value>`      | Upload ID from a previous upload |
| `--app-id=<value>`         | App ID to check latest status    |
| `-x, --extended`           | Show extra columns               |
| `--output=csv\|json\|yaml` | Output format                    |
| `-c, --config-dir`         | Custom config directory          |

**Status values:**

- `WAITING_FOR_UPLOAD` — upload created, waiting for the bundle file
- `PROCESSING` — bundle received, being validated and processed
- `ACTIVE` — bundle processed successfully
- `FAILED` — processing error (the `error` property holds a problem; see `error-reference.md`)

---

### `list`

List all uploaded apps.

```bash
npx showpad apps list
npx showpad apps list --output json
```

**Flags:**

| Flag                       | Description                |
| -------------------------- | -------------------------- |
| `-x, --extended`           | Show extra columns         |
| `--columns=<cols>`         | Show specific columns only |
| `--csv`                    | Output as CSV              |
| `--filter=<string>`        | Filter rows                |
| `--no-header`              | Suppress header row        |
| `--output=csv\|json\|yaml` | Output format              |
| `--sort=<col>`             | Sort by column             |
| `-c, --config-dir`         | Custom config directory    |

---

### `versions`

List all versions of a specific app.

```bash
npx showpad apps versions
npx showpad apps versions --app-id <appId>
```

**Flags:** Same table/output flags as `list`, plus:

| Flag               | Description     |
| ------------------ | --------------- |
| `--app-id=<value>` | App ID to query |

Use this to get `versionId` for the `assign` command.

---

### `assign`

Assign an app version to one or more Showpad organizations.

```bash
# Pick the app and version interactively (partner code is still required)
npx showpad apps assign --partner-code 91dd4290-369e-488e-80df-634d6cb090a6

# Fully non-interactive
npx showpad apps assign --partner-code <partnerCode> --app-version-id <versionId>

# Several organizations at once
npx showpad apps assign -p <partnerCode1> -p <partnerCode2> --app-version-id <versionId>
```

**Flags:**

| Flag                        | Required | Description                                                                                         |
| --------------------------- | -------- | --------------------------------------------------------------------------------------------------- |
| `-p, --partner-code=<uuid>` | **Yes**  | Organization's Partner Code (UUID). Never prompted: the command fails without it. Repeatable.       |
| `--app-version-id=<ulid>`   | No       | Version to assign. If omitted, the CLI prompts for the app and version. Required with `--no-input`. |
| `-c, --config-dir`          | No       | Custom config directory                                                                             |
| `--no-input`                | No       | Non-interactive mode                                                                                |
| `--verbose`                 | No       | Verbose output                                                                                      |

**Partner Code:** a UUID. In the Showpad platform, click **Install App**: the install modal links to the partner code in its bottom-left corner. Also called `organisationId` in the API. If the user hasn't given it, ask for it. Don't emit an `assign` command with a placeholder and claim it will prompt.

---

### `unassign`

Remove an app version from a Showpad organization.

```bash
npx showpad apps unassign --partner-code <partnerCode> --app-version-id <versionId>
```

**Flags:** Same as `assign`, except `--partner-code` takes a **single** UUID (not repeatable). It is required; only `--app-version-id` can be picked interactively.

**Exit code 33** ("app version is not assigned to organisation"): the version was never assigned to that partner code.

**Note:** Propagation can take several minutes. The app stops functioning for that organization once propagated.

---

### `help`

Show help for any command.

```bash
npx showpad apps help
npx showpad apps help upload
npx showpad apps help bundle
```

**Flags:**

- `-n, --nested-commands` — include all nested commands in output

---

## Common CI/CD workflow

```bash
# Env vars come from the CI secret store (SHOWPAD_ADMIN_PASSWORD etc.)
export SHOWPAD_SUBDOMAIN=acme
export SHOWPAD_ADMIN_USERNAME=admin@acme.com

# Build
npm run build && npm run prebundle

# Bundle
npx showpad apps bundle -s dist -o . --no-input

# Upload
npx showpad apps upload ./com.acme.my-app.*.showpad --no-input

# Assign (optional — only if auto-assigning to a known org)
# Both flags are mandatory here: PARTNER_CODE is a UUID, VERSION_ID a ULID (from `apps versions --output json`)
npx showpad apps assign --partner-code $PARTNER_CODE --app-version-id $VERSION_ID --no-input
```
