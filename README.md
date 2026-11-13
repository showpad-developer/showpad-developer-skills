# Showpad Developer Skills

AI skills for building Showpad Apps v2 and working with the Showpad SDKs, without technical knowledge. They give Claude Code and other coding agents the exact CLI commands, manifest fields, extension types, SDK method signatures and types, so the agent doesn't have to guess them.

## Skills

| Plugin         | Skills                                                                                                                   | Use it for                                                                                                                                                     |
| -------------- | ------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `showpad-apps` | [`showpad-app`](skills/showpad-app/SKILL.md)                                                                             | The whole Showpad App v2 lifecycle: scaffolding, the Showpad CLI, manifest, extension types, bundling, uploading and fixing errors. Depends on `showpad-sdks`. |
| `showpad-sdks` | [`showpad-app-sdk`](skills/showpad-app-sdk/SKILL.md), [`showpad-app-editor-sdk`](skills/showpad-app-editor-sdk/SKILL.md) | Reference for `@showpad/experience-app-sdk` (`Showpad.*`) and `@showpad/experience-editor-sdk` (`ShowpadEditor.*`, used to build Custom Editors).              |

## Installation

### Claude Code plugin marketplace

```text
/plugin marketplace add showpad-developer/showpad-developer-skills
/plugin install showpad-apps@showpad-developer-skills
```

You can pass a full git URL or a local path to `/plugin marketplace add` instead of `owner/repo`.

### `npx skills`

```bash
# Install all skills
npx skills add showpad-developer/showpad-developer-skills

# Install specific skills
npx skills add showpad-developer/showpad-developer-skills --skill showpad-app showpad-app-sdk showpad-app-editor-sdk
```

## Repository layout

```text
.claude-plugin/
  marketplace.json   # marketplace catalog; each plugin entry lists its skill folders
skills/
  <skill-name>/
    SKILL.md         # frontmatter (name, description) and instructions
    references/      # detailed docs, loaded only when needed
```

## Adding a skill

1. Create `skills/<skill-name>/SKILL.md` with `name` and `description` frontmatter. `name` must match the folder name.
2. Put long reference material in `skills/<skill-name>/references/` and link to it from `SKILL.md`.
3. Add the skill folder to the `skills` array of a plugin entry in `.claude-plugin/marketplace.json`, or add a new plugin entry for it.
4. Bump `version` on that plugin entry.
5. Run `claude plugin validate .` before you push.

`npx skills` finds every skill under `skills/` on its own, so it needs no changes.
