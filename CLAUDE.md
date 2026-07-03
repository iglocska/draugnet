# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**DraugnetUI** is the static web frontend for the **[Draugnet](https://github.com/draugnet/draugnet)** backend, a community threat intelligence submission tool. It communicates with the Abracadabra backend API and allows users to submit threat reports in multiple formats and view them by token.

**Stack:** Vanilla JavaScript, HTML5, Bootstrap 5, Font Awesome — no framework, no build step, no npm.

## Running / Serving

This is a purely static site. Serve the `webroot/` directory with any web server:

```bash
# Python (quick dev server)
python3 -m http.server --directory webroot 8080

# Docker (production)
docker build -t draugnetui .
docker run -p 80:80 draugnetui
```

No build, compile, or install step required.

## Configuration

```bash
cp webroot/config/config.default.json webroot/config/config.json
# Edit config.json:
# { "baseurl": "https://your-backend-url", "timezone": "Europe/Berlin" }
```

`webroot/config/config.json` is gitignored (it contains the backend URL). The app loads it at runtime via fetch and falls back to `window.location.origin` if absent.

## Architecture

### File Structure

```
webroot/
├── config/config.json        # Runtime config (backend URL, timezone)
├── js/
│   ├── main.js               # Config loading, report viewing, download formats
│   ├── menu.js               # Shared nav loading, light/dark theme toggle
│   ├── tokenstore.js         # localStorage token CRUD, export/purge
│   ├── metadata.js           # Metadata form handling (TLP, PAP, distribution)
│   └── visualizer.js         # MISP event tree renderer
├── css/style.css             # CSS custom properties, light/dark theme variables
├── *.html                    # One file per page (index, view, misp, object, freetext, stix, csv, template)
├── menu.html                 # Shared nav (loaded dynamically via fetch)
├── tokenstore.html           # Token sidebar (loaded dynamically via fetch)
└── metadata.html             # Metadata accordion (loaded dynamically via fetch)
```

### Page Layout Pattern

Every page dynamically fetches and injects three shared HTML fragments: `menu.html` (navbar + theme toggle), `tokenstore.html` (right sidebar), and `metadata.html` (submission metadata accordion). Pages signal their identity to `menu.js` via a `data-page` attribute for active-link highlighting.

### State

- **Tokens**: stored in `localStorage["tokens"]` as a JSON array of strings.
- **Theme**: stored in `localStorage["preferred-theme"]` (`"light"` or `"dark"`), applied as `data-bs-theme` on `<html>`.
- No server-side sessions; all state is browser-local.

### Key JS Responsibilities

| File | Responsibility |
|------|---------------|
| `main.js` | Loads `config.json`, drives the view page (raw JSON / visual tree toggle, download-as selector, copy-to-clipboard) |
| `visualizer.js` | Recursively renders a MISP event as a collapsible tree; sorts nodes by type (Tag → Galaxy → Attribute → Note → Object…); highlights nodes newer than the last submission timestamp |
| `metadata.js` | Fetches sharing groups from backend, builds metadata payload sent alongside every submission. Exports `getMetadata()` both as an ES module export and via `window.getMetadata` so both module and non-module scripts can call it. |
| `tokenstore.js` | Renders the token list, handles add/delete/export/purge actions |
| `menu.js` | Injects nav, wires theme toggle |

### Adding a New Submission Page

1. Create `webroot/<type>.html` — copy `freetext.html` as the template. Set `data-page="<type>"` on `<body>` and POST to `/share/<type>` with a payload of `{ <type>: data, optional: meta }`.
2. Add a nav link in `menu.html`: `<a class="nav-link" href="<type>.html" data-page="<type>">Report via …</a>`
3. Add the corresponding backend route in `main.py` and list it in `GET /share`.

### The Template Page (`template.html`)

`template.html` ("Report via Template") is the guided, event-template-driven
submission page. Unlike the other pages it renders a **dynamic form from a backend
template definition** rather than a fixed layout. Everything lives in one inline
`<script>` (matching the `object.html` / `csv.html` idiom — no new `js/` file); it
reuses `loadConfig`/`baseUrl`/`showToast`/`esc` (main.js) and `getTokens`/`saveTokens`
(tokenstore.js).

- **Flow:** `GET /templates` (searchable picker) → `GET /templates/{uuid}` (definition)
  → per-element renderer for all 9 element types (section, text_block, attribute_field,
  object_field, tag_field, galaxy_field, file_field, event_report, object_reference) →
  validate → `POST /share/template` → save token.
- **Pickers:** `tag_field` → `GET /taxonomies?ns=`; `galaxy_field` →
  `GET /galaxy_clusters?type=&q=`; `object_field` → `GET /object_templates?template=`.
  Help/text_block/event_report Markdown is rendered through a safe escape-first
  renderer (`safeMarkdown`; http/https/mailto links only).
- **Slim metadata** (PAP / submitter / description) is built **inline** on this page,
  not via `metadata.html` / `metadata.js` — the template owns info/distribution/threat/
  analysis/tags/galaxies through its `event_defaults`, so the page needs only those three.
- **Submit body:** `{template_uuid, values:{<element_id>: <value(s)>}, optional:{pap,
  submitter,description}}`. There is **no channel to override `event_defaults`**, so the
  defaults panel is **read-only/informational** with a live `{{date}}/{{now}}/{{user}}/
  {{field:id}}` info preview (best-effort client-side; the server is authoritative).
- **Values contract** (must match the backend): attribute/tag/galaxy = `str|list`
  (tag = full machine tag `tlp:amber`; galaxy = cluster *value* string `APT28`);
  `object_field` = `{relation:value}` or list-of-maps; `file_field` = `{filename,
  data(base64)}` (data-URI prefix stripped) or list; `event_report` = trimmed Markdown
  string. Empty widgets are omitted. Submit is disabled until all mandatory elements +
  mandatory object relations are filled.

### Backend API Calls

All calls use the `baseurl` from `config.json`:

| Endpoint | Method | Used by |
|----------|--------|---------|
| `/share/misp` | POST | misp.html |
| `/share/objects` | POST | object.html |
| `/share/raw` | POST | freetext.html |
| `/share/stix` | POST | stix.html |
| `/share/csv` | POST | csv.html |
| `/share/template` | POST | template.html |
| `/templates` | GET | template.html (picker) |
| `/templates/{uuid}` | GET | template.html (definition to render) |
| `/taxonomies?ns=` | GET | template.html (tag_field picker) |
| `/galaxy_clusters?type=&q=` | GET | template.html (galaxy_field picker) |
| `/retrieve?token=&format=` | GET | view page (main.js) |
| `/timestamp?token=` | GET | view page (main.js) |
| `/object_templates[?template=]` | GET | object.html, template.html (object_field) |
| `/sharing_groups` | GET | metadata.js |

Successful submissions return `{ status: "ok", token: "..." }`. The token is saved to localStorage automatically.
