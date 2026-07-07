# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Central Operacional" — a single-page internal tool for a logistics cold-chain monitoring desk
(Cordenonsi / AngelLira Rastreamento Satelital). It replaces a set of Google Sheets tabs that
operators used to consult during a shift: temperature ranges per product, alert-response text
templates, shipper/carrier lists, blocked drivers, maintenance contacts, yard-maneuver contacts,
and an escalation flowchart. UI text and data are in Brazilian Portuguese.

## Project structure

The entire app is one file: `index.html` (~1400 lines: inline `<style>`, then two inline
`<script>` blocks). There is no build step, no package.json, no bundler, and no test suite.
To work on it, edit `index.html` directly and open it in a browser (or serve it with any static
file server, e.g. `npx serve` / `python -m http.server`, to avoid `file://` quirks with the
Supabase JS SDK).

External dependencies are all loaded via CDN `<script>`/`<link>` tags in `<head>`: Google Fonts,
Bootstrap Icons, and `@supabase/supabase-js@2`.

## Architecture

### Two script blocks with a hard separation of concerns

1. **`CAMADA DE DADOS`** (data layer, first `<script>`): Supabase client setup, auth, and all
   domain data as plain JS constants/arrays — `FAIXAS` (temperature ranges), `OPERACIONAIS`
   (alert-response text templates), `EMBARCADORES` (shippers), `CONDUTORES` (blocked/outdated
   drivers), `MANUTENCAO` (maintenance contacts), `MARFRIG` (yard contacts), plus the PML
   (Plano de Monitoramento Logístico) constants: `ESCALONAMENTO`, `GESTORES`, `COORDENADORES`,
   `CONTATOS_EMERGENCIA`, `DIRETRIZES_MODO`.
2. **`CAMADA DE APLICAÇÃO`** (app/UI layer, second `<script>`): view routing and all rendering.

This split is intentional per the comment at the top of the data layer: the arrays are meant to
be swappable for `fetch()` calls to a Google Apps Script Web App (see `APPS_SCRIPT_URL`,
`SHEET_ID`) that would return the same shapes read live from the source spreadsheet, without
touching the app layer at all. When adding data, keep it as plain objects matching the existing
shape (e.g. via the small factory helpers `F()`, `E()`, `C()`) rather than reaching for a
framework or restructuring.

### Rendering model

No framework: `currentView` is a string, `render()` is a switch that calls one `renderXxx()` per
view (`renderInicio`, `renderOperacionais`, `renderFaixas`, `renderEmbarcadores`,
`renderCondutores`, `renderManutencao`, `renderMarfrig`, `renderPML`, `renderPesquisa`,
`renderFavoritos`, `renderConfiguracoes`, `renderSobre`). Each `renderXxx()` fully replaces
`viewport.innerHTML` with a template string and then re-attaches its own event listeners —
there is no diffing or component state beyond module-level `let` variables (e.g. `opFilterCat`,
`opQuery`, `EDITED_TEXTOS`). When adding a view, follow this same pattern: add a nav item with
`data-view`, a `VIEW_TITLES` entry, a branch in `render()`, and a `renderXxx()` function.

### Auth and per-user data (Supabase)

Supabase is used only for login and a small amount of shared/synced state — not for the domain
data above. Config lives at the top of the data layer (`SUPABASE_URL`, `SUPABASE_ANON_KEY`).
Operators log in with a plain "usuario" (no email) that gets mapped to a fake address via
`usuarioParaEmail()` (`usuario@interno.local`) purely so Supabase Auth accepts it; this fake
address is never shown in the UI. Three tables back everything:
- `perfis` — last-login timestamp per user (`registrarLogin()`).
- `favoritos` — favorited item ids per user (`toggleFav()` / `isFav()`).
- `textos_editados` — per-user edited copies of `OPERACIONAIS` alert templates
  (`getOpText()` / `saveOpText()`), overlaid on the hardcoded defaults via `EDITED_TEXTOS`.

`EMAIL_MASTER` designates a single master account. When logged in as master, Configurações also
renders a "Painel Master" (`renderPainelMaster()`) listing every account's last login and every
operator's edited templates, with a reset-to-default action.

### Theming

All colors are CSS custom properties on `:root` (dark, the default variable values) with a
`[data-theme="light"]` override block. The active theme is toggled by `applyTheme()` and
persisted in `localStorage` under `co_theme`.

### Temperature classification

`classifyTemp(min)` buckets a range's minimum temperature into `lowest` / `ideal` / `warning` /
`critical`, which drives the `.temp-badge` styling in the Faixas de Temperatura view. When
adding/editing entries in `RAW_FAIXAS`, the human-readable range and thresholds are derived
automatically by `standardizeFaixa()` — don't hand-compute `faixa`/`min`/`classe`.
