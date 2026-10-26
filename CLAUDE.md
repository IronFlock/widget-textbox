# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `npm run build` — bundle `src/widget-textbox.ts` into `dist/` via Rollup.
- `npm run watch` — Rollup in watch mode.
- `npm start` — runs `watch` and `@web/dev-server` concurrently; serves `demo/index.html` (open `/demo/`).
- `npm run types` — regenerate `src/definition-schema.d.ts` from `src/definition-schema.json` via `json2ts`. Run after editing the schema.
- `just lint` — what the CI gate checks besides the build: regenerating the committed generated files (`npm run types`) changes nothing. `just build` = `npm run build`.
- `just merge` / `just release-patch` (`release-minor`, `release-major`) — pull requests into `main` from a feature branch with everything committed: a no-release one, or a release one whose merge publishes to npm in CI (`.github/workflows/build-publish.yml`), then the release GitHub App tags the bare version and creates the GitHub release. Never `npm publish`, tag or edit the version by hand.
- No tests or linter are configured.

Requires Node `>=24.9.0`, npm `>=10.0.2`.

## Architecture

This is a single LitElement web component (`@ironflock/widget-textbox`) used as a tile in the IronFlock/RESWARM dashboard platform. The repo follows the same pattern as the sibling `widget-*` packages in `ironflock-widgets/`.

**Entry point:** `src/widget-textbox.ts` defines `WidgetTextbox` and registers it as `widget-textbox-versionplaceholder`. The literal string `versionplaceholder` is replaced at build time by `@rollup/plugin-replace` with the `package.json` version, so each published build registers a uniquely versioned custom-element tag. The host application (RESWARM frontend) reads the version from the package and instantiates the matching tag — see `demo/index.html` for the pattern (`unsafeStatic` with the version-suffixed tag name).

**Platform integration contract:** the component receives two reactive properties from the host:
- `inputData: TextboxConfiguration` — typed by `src/definition-schema.d.ts`, generated from `src/definition-schema.json`. The JSON schema is the source of truth: the platform's tile editor renders form controls from it (`order`, `dataDrivenDisabled`, custom `type: "color"` / `"textarea"`), and fields can be bound to backend data sources unless `dataDrivenDisabled` is set.
- `theme: { theme_name, theme_object }` — applied in `registerTheme()`. The CSS custom property `--re-text-color` from the host takes precedence over `theme_object` values; user-configured colors in `inputData` then take precedence over the theme. These are not snapshotted: `registerTheme()` stores a `var(--re-…, <theme value>)` chain, so a change to the host property repaints the tile live without the widget being told. The widget paints no background of its own: studio's `widget-item` already paints `--re-tile-background-color` on the tile, and painting it again here composited a translucent tile colour twice.

**Build pipeline:** Rollup with `typescript`, `node-resolve`, `commonjs`, `babel` (bundled helpers), `replace` for the version placeholder. Output is a single ESM bundle at `dist/widget-textbox.js` with sourcemaps. `treeshake.moduleSideEffects: false` is set — be careful adding modules with side effects.

**Local demo:** `demo/index.html` fetches `package.json` to construct the versioned tag, loads `src/default-data.json` as `inputData`, and `demo/themes/light.json` as the theme object. Use this to reproduce platform behavior locally.

## README highlights

The component renders three styled sections — title (`h2`), subtitle (`h3`), body (`p`, `white-space: pre-wrap` so `\n` becomes a line break). Each section is hidden via the `paging` class when its `text` is empty.

## `aiSelection` in `src/definition-schema.json`

The schema root carries an `aiSelection` block next to `title` and `description`. It is **not** JSON Schema and describes no config field — it exists so the IronFlock AI's Widget Builder can pick the right widget for a given shape of data, using knowledge only the widget author has:

```jsonc
"aiSelection": {
  "dataShape": "…what columns this widget consumes and what each one means…",
  "useWhen":   ["…a situation, naming the properties that express it…"],
  "notFor":    ["…a situation this widget is wrong for, naming the widget to use instead…"]
}
```

It is inert everywhere else, and must stay that way: `json2ts` ignores it (the generated `.d.ts` is byte-identical with and without it), the dashboard config editor renders only `schema.properties`, and the AI service's `validate_widget` validates *configs* against the schema, skipping unknown Draft-7 keywords.

When maintaining it:

- `notFor` is the high-value half and the part plain descriptions always omit. Every entry must name the widget that *should* be used, or it rejects without routing.
- Write for an LLM with no other documentation: describe the visible result and the user's intent, not the implementation.
- Prefer entries that discriminate against a *neighbouring* widget. Generic rejections are cheap; the ones that pay are those an author could plausibly get wrong.
- The `notFor` lists are a set across all `widget-*` repos and are meant to be reciprocal — if this widget routes to another for some case, that widget should usually route back for the converse. Changing one side is a cue to check the other.
- Update it whenever a property changes what this widget can *do*, not just how it looks.
