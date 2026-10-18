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
- `npm run dev` + `npm run dev:ha` (`--restore`, `--status`, `--clean`): points a Lovelace resource at the local Vite server (needs `.env` from `.env.example`, HA reached over HTTP). The installed bundle loads first, but hands over to the dev entry it finds in `localStorage` (`owlnest_dev_entry`) and falls back to itself if the dev server is unreachable (console: `[Owlnest] dev server unreachable, using the bundled card`).
- Releases are made **upstream by the maintainer** with `scripts/release.mjs` (bumps versions, commits bundle + `docs/releases/vX.Y.Z.md`, tags). Never run it or bump versions in the fork or in PR branches.

## Conventions

- Code comments in **French**, sparse, explaining constraints/why — never narrating the change.
- Commit messages in **English**: imperative subject ending with a period, bullet-point body, trailer `Co-authored-by: Cursor <cursoragent@cursor.com>`.
- UI strings live in `src/i18n.ts`; always add both `en` and `fr`.
- User docs: the short `README.md` / `README-FR.md` and the guides `docs/guide.md` / `docs/guide-fr.md` (Openings section) must stay in sync EN/FR.
- Editor UI is built imperatively with inline `cssText` dark styling (see `src/card/edit-panel.ts`); match it.

## The user's real setup

- The real model is **outside the repo**: `g:\Mi unidad\Domotica\Modelado Casa Vallecas\Modelado en  Blender\export a glb\casa_vallecas_00N.glb` (note the **two spaces** in "Modelado en  Blender"; Google Drive mount; latest known version: `006`). **Never modify it.** For analysis, copy it into a temp folder inside the repo and delete the copy afterwards.
- What it looks like (from analysis of `004`/`006`): ~80 MB Blender glTF export, ~1,200 mostly flat nodes (Blender collections are not exported as parents), objects keep the user's names (e.g. `puerta terraza`, `puerta_lavaplatos`, awnings `motor` / `tela` / `extremo` + `001`/`002` copies). Multi-material objects load in three.js as a Group with one child mesh per material. Top-level nodes carry a **+90° X rotation and 0.01 scale** (Z-up centimetre geometry shown Y-up in metres). three.js sanitises names (`tela.001` → `tela001`, spaces → `_`).
- `Modele3D/flat-archi.glb` in the repo is a different model (Sweet Home 3D export, flat sibling meshes); `obsidian/Modele 3D.md` documents older models (`floorplan2.glb`, merged meshes). Don't assume they represent the user's house.
- Deployment/testing: the user manually uploads **only** `custom_components/owlnest/frontend/ha-3d-floorplan.js` to `/config/custom_components/owlnest/frontend/` and hard-refreshes (Ctrl+Shift+R). No HA restart unless Python files change. There is no Samba access from this PC.

## Openings module (animated doors, shutters, awnings)

- Released upstream in v1.4.0: `PARTS_ENABLED` in `src/parts.ts` is `true` upstream too, so the fork no longer overrides it.
- Key files: `src/parts.ts` (piece index, frames, pivots), `src/parts-runtime.ts` (`PartController`: mount, `configure`, `applyStates`, `update`), `src/parts-clips.ts` (GLB clips), `src/part-carry.ts` (carried pieces, e.g. handles), `src/model-outline.ts` (Object tree logic), `src/part-highlight.ts` (viewport highlight), `src/part-tint.ts` (state colours on cloned materials), `src/import/sh3d-parts.ts` (openings detected on Sweet Home 3D import), `src/card/edit-panel.ts` (`_openPartModal`), `src/ha-3d-floorplan.ts` (wiring, picking, `refreshParts`).
- Motions (`PartMotion` in `src/types.ts`): `swing` (vertical/horizontal axis), `slide`, `extend` (directional scale anchored at an edge, with rigid `followers`), `animation` (scrubs Blender NLA clips by the open fraction).
- All `OwlnestPart` fields beyond the original ones are optional and backward compatible; store only non-defaults.
- Always handle world vs. local frames (see `_localVertical`): the model's vertical is measured in world space, pivots live in mesh/node local space.
- Settings apply live through `configure()` without reloading the model; Cancel in the modal restores the original part. Never mutate shared materials.

## Git / GitHub

- `origin` = `cacharritos/ha-owlnest` (the user's fork); `upstream` = `MestrieEsteban/ha-owlnest` (original). Upstream PRs #4 (editor), #5 (extend) and #6 (GLB animations) were merged and released in v1.4.0; the maintainer rebases contributions onto `main` (linear history, the user stays author).
- Fork `main` = `upstream/main` + **one fork-only commit** (this file). Sync by rebasing it: `git fetch upstream; git rebase upstream/main` on `main`, then `git push --force-with-lease origin main`.
- New work: `git switch -c feature/x upstream/main`, never from fork `main`, so PR branches don't carry `AGENTS.md`. Check with `git log upstream/main..HEAD` before opening a PR.
- Merged branches: delete them (local + `origin`) and remove their worktrees (`git worktree remove`; on Windows a locked folder may need a second `Remove-Item`).
- `gh` is installed and authenticated as `cacharritos`. If it isn't found, refresh PATH: `$env:Path = [Environment]::GetEnvironmentVariable('Path','Machine') + ';' + [Environment]::GetEnvironmentVariable('Path','User')`.
- **Ask before pushing, opening PRs or merging.**
- Windows: CRLF/LF warnings and line-ending-only diffs show up; don't commit EOL-only changes (check with `git diff --ignore-cr-at-eol`).

## Cursor Cloud specific instructions

- `npm ci` is enough to install. The default image's Node 22 runs `npm run typecheck`, `npm test`, and `npm run build`. CI pins Node 20; the lockfile works on both.
- The app you can run here is the simulated demo: `npm run demo -- --host 0.0.0.0 --port 5180` (`vite.demo.config.js`). It serves the real card on `custom_components/owlnest/frontend/demo.glb` with a fake Home Assistant. Open `http://127.0.0.1:5180/`.
- `npm run dev` (port 5173) only serves the card module for a real dashboard. It has no page of its own.
- Python checks do not need Home Assistant installed: `python3 -m unittest discover checks`.
- `HA_URL` and `HA_TOKEN` (see `.env.example`) are only for `npm run dev:ha`, which talks to a Home Assistant on the user's LAN. That instance is not reachable from a Cloud Agent. Do not block on a token.
- Never run `npm run release` or bump versions.
