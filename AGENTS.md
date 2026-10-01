# AGENTS.md

Context for coding agents working on this repo. Only what the code doesn't tell you.

## Project

- Home Assistant integration (`custom_components/owlnest/`, Python) + Lovelace card (`src/`, TypeScript + three.js) that renders a 3D model of the house.
- The card bundle is **embedded in the integration and versioned on purpose**: HACS reads the repo at the tag, and `custom_components/owlnest/frontend.py` serves and registers it itself (`add_extra_js_url`, `?v=<manifest version>`).
- `npm run build` = `vite build` + `scripts/sync-card.mjs`, which copies `dist/ha-3d-floorplan.js` to `custom_components/owlnest/frontend/ha-3d-floorplan.js`. **Rebuild and commit the bundle together with any `src/` change.** `dist/` is gitignored.

## Commands

- `npm run typecheck` — `tsc --noEmit`.
- `npm test` — `scripts/test.mjs` compiles the modules listed in `ENTRIES` with esbuild into `.test-build/`, copies `src/**/*.test.mjs`, and passes the files explicitly to `node --test` (Node 24 no longer scans a directory argument). **New testable modules must be added to `ENTRIES`.** Tests have no DOM; keep testable logic out of the editor UI.
- `npm run build` — see above.
- `npm run dev` + `npm run dev:ha` (`--restore`, `--status`, `--clean`): points a Lovelace resource at the local Vite server (needs `.env` from `.env.example`, HA reached over HTTP). Caveat: the integration also injects the installed bundle via `add_extra_js_url`, and the card guards `customElements.define`, so the dev resource can silently lose to the installed bundle. Check for the `[Owlnest] mode dev` console message.
- Releases go through `scripts/release.mjs` (bumps `package.json` and `manifest.json`, tags). **Don't bump versions by hand.**

## Conventions

- Code comments in **French**, sparse, explaining constraints/why — never narrating the change.
- Commit messages in **English**: imperative subject ending with a period, bullet-point body, trailer `Co-authored-by: Cursor <cursoragent@cursor.com>`.
- UI strings live in `src/i18n.ts`; always add both `en` and `fr`.
- `README.md` and `README-FR.md` must stay in sync (both have an Openings section).
- Editor UI is built imperatively with inline `cssText` dark styling (see `src/card/edit-panel.ts`); match it.

## The user's real setup

- The real model is **outside the repo**: `g:\Mi unidad\Domotica\Modelado Casa Vallecas\Modelado en  Blender\export a glb\casa_vallecas_00N.glb` (note the **two spaces** in "Modelado en  Blender"; Google Drive mount; latest known version: `006`). **Never modify it.** For analysis, copy it into a temp folder inside the repo and delete the copy afterwards.
- What it looks like (from analysis of `004`/`006`): ~80 MB Blender glTF export, ~1,200 mostly flat nodes (Blender collections are not exported as parents), objects keep the user's names (e.g. `puerta terraza`, `puerta_lavaplatos`, awnings `motor` / `tela` / `extremo` + `001`/`002` copies). Multi-material objects load in three.js as a Group with one child mesh per material. Top-level nodes carry a **+90° X rotation and 0.01 scale** (Z-up centimetre geometry shown Y-up in metres). three.js sanitises names (`tela.001` → `tela001`, spaces → `_`).
- `Modele3D/flat-archi.glb` in the repo is a different model (Sweet Home 3D export, flat sibling meshes); `obsidian/Modele 3D.md` documents older models (`floorplan2.glb`, merged meshes). Don't assume they represent the user's house.
- Deployment/testing: the user manually uploads **only** `custom_components/owlnest/frontend/ha-3d-floorplan.js` to `/config/custom_components/owlnest/frontend/` and hard-refreshes (Ctrl+Shift+R). No HA restart unless Python files change. There is no Samba access from this PC.

## Openings module (animated doors, shutters, awnings)

- Feature flag `PARTS_ENABLED` in `src/parts.ts` (`true` in this fork, `false` upstream).
- Key files: `src/parts.ts` (piece index, frames, pivots), `src/parts-runtime.ts` (`PartController`: mount, `configure`, `applyStates`, `update`), `src/model-outline.ts` (Object tree logic), `src/part-highlight.ts` (viewport highlight), `src/part-tint.ts` (state colours on cloned materials), `src/card/edit-panel.ts` (`_openPartModal`), `src/ha-3d-floorplan.ts` (wiring, picking, `refreshParts`).
- Motions: `swing` (vertical/horizontal axis), `slide`, `extend` (directional scale anchored at an edge, with rigid `followers`).
- All `OwlnestPart` fields beyond the original ones are optional and backward compatible; store only non-defaults.
- Always handle world vs. local frames (see `_localVertical`): the model's vertical is measured in world space, pivots live in mesh/node local space.
- Settings apply live through `configure()` without reloading the model; Cancel in the modal restores the original part. Never mutate shared materials.

## Git / GitHub

- `origin` = `cacharritos/ha-owlnest` (the user's fork); `upstream` = `MestrieEsteban/ha-owlnest` (original). Contributions go upstream via PRs from the fork (e.g. upstream PR #4).
- `gh` is installed and authenticated as `cacharritos`. If it isn't found, refresh PATH: `$env:Path = [Environment]::GetEnvironmentVariable('Path','Machine') + ';' + [Environment]::GetEnvironmentVariable('Path','User')`.
- **Ask before pushing, opening PRs or merging.**
- Windows: CRLF/LF warnings and line-ending-only diffs show up; don't commit EOL-only changes (check with `git diff --ignore-cr-at-eol`).
