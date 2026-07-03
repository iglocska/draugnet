# Operator UI end-to-end checklist — Report via Template

A short, manual script for an operator to validate the **Report via Template**
page (`template.html`) after a deploy or upgrade. There is no automated frontend
test suite; run this by hand. Do the full pass **once in the light theme and once
in the dark theme** — the theme toggle is in the top nav (sun/moon).

The backend API path (instantiation, all 9 element types, both template sources)
is covered by `abracadabra/tests/test_api.py` (T1–T9). This checklist covers the
**browser** half: render → fill → submit → token → retrieve/visualize.

## 0. Prerequisites

- [ ] Draugnet backend (Abracadabra) is running and reachable.
- [ ] `webroot/config/config.json` `baseurl` points at that backend (or the app is
      served from the same origin so the `window.location.origin` fallback works).
- [ ] The backend's `allowed_origins` includes the origin you load the UI from
      (a CORS rejection shows as an empty template picker + a console error).
- [ ] Templates are available: opening the page shows a non-empty picker. If empty,
      check the backend `template_config["source"]` — `filesystem` needs
      `definition.json` files under its `dir`; `misp` needs at least one template
      flagged **exposed** and readable by the service account.
- [ ] Bundled submodules are populated on the backend (`git submodule update --init
      --recursive`) — otherwise the tag/galaxy pickers come up empty.

## 1. Per-template loop (repeat for each shipped template, in each theme)

For each template below:

1. [ ] Open **Report via Template** from the nav. The picker lists the templates;
       the active nav link is highlighted.
2. [ ] Type in the picker to **search/filter**; keyboard up/down + Enter select;
       the clear (×) resets it.
3. [ ] Select the template → the guided form renders: **sections as cards**, each
       field showing its **label + help text** (Markdown rendered), mandatory fields
       marked, `text_block` prose shown with no input.
4. [ ] The **Submit button is disabled** until every mandatory element (and every
       mandatory object relation) is filled. Confirm it is disabled on an empty form.
5. [ ] Fill at least the **mandatory** fields (listed per template below) plus a
       couple of optional ones to exercise the widgets.
6. [ ] Exercise the pickers where present:
   - **tag_field** (e.g. `tlp`) — dropdown populated from the taxonomy; single vs
     multi per the field; selected tags show as chips.
   - **galaxy_field** (e.g. threat-actor) — typeahead; typing filters clusters;
     selecting adds a chip.
   - **object_field** — the object's relations render with their overrides; add/remove
     works on repeatable objects; `repeatable` attribute fields add/remove instances.
7. [ ] Watch the **info preview** update live as you type into `{{field:…}}` sources
       (it also substitutes `{{date}}`/`{{now}}`/`{{user}}`). The **defaults panel is
       read-only** (the template owns distribution/threat/analysis/tags) — locked
       items are marked as such.
8. [ ] In the **slim metadata** block set **PAP** (PAP:CLEAR/GREEN/AMBER/RED),
       **submitter**, and an extra **description**.
9. [ ] **Submit.** On success a **token** is saved to the token store (right sidebar)
       and a success toast appears.
10. [ ] Click the token / go to the **view page** and **retrieve** the event; toggle
        **raw JSON ↔ visual tree**. Confirm:
    - `info` matches the substituted `info_template` (e.g. `Spearphishing — <date> —
      <sender>`);
    - the values you entered are present as typed attributes / objects;
    - the `tlp:*` tag, any galaxy tag (`misp-galaxy:…="…"`), `source:draugnet`, and
      `submitter:<name>` are on the event;
    - the extra description appears as an event report.
11. [ ] Force a **validation failure**: clear a mandatory field and confirm Submit
        disables again; on a deliberate bad submit the invalid field gets an inline
        marker and a toast points at the first gap.

### Shipped templates — minimal fill

- **Spearphishing email triage** — mandatory: `sender`, `subject`, `tlp`.
  Also worth touching: the `email` object, the repeatable `attachment` object +
  the email→attachment reference, repeatable `payload_url`, multi `kill_chain`,
  `actor` (threat-actor galaxy).
- **Ransomware incident** — mandatory: `family_name`, `date_observed`, `tlp`.
  Also: the `incident_summary` event report, repeatable host/user/C2 fields, the
  repeatable sample object, `ransomware_family` + `threat_actor` galaxies.
- **Suspicious domain triage** — mandatory: `domain`, `date_observed`, `tlp`.
  Also: the `why_suspicious` event report, `domain|ip` + `whois` objects,
  repeatable `observed_url`, multi `kill_chain`, `threat_actor` galaxy.

> `file_field` is not exercised by any shipped template. To test it, point the
> filesystem source at a template that includes a `file_field` element (attachment
> and/or malware-sample) and confirm the file picker → base64 path and, for
> malware-sample, that the retrieved attribute is the encrypted (`filename|md5`) form.

## 2. Theme parity (do the full pass twice)

- [ ] **Light theme**: complete section 1 for all three templates.
- [ ] **Dark theme**: toggle to dark and complete section 1 again.
- [ ] In both themes confirm no unreadable low-contrast text — especially: section
      cards, help text, mandatory markers, picker dropdowns + chips, the read-only
      defaults panel, the info preview, inline validation markers, and toasts.
- [ ] The chosen theme persists across a reload (stored in `localStorage`).

## 3. Source switch (optional, backend-side — confirms both sources drive the UI)

The UI is source-agnostic (it only calls `/templates*`). To confirm both backend
sources render identically:

- [ ] With `template_config["source"] = "filesystem"` (default), run section 1.
- [ ] Set `template_config["source"] = "misp"`, restart the backend, and confirm the
      picker now lists the **exposed** MISP templates and a submit still round-trips
      to a token + retrievable event.
- [ ] Switch back to `filesystem` and restart.

## Sign-off

| Template | Light | Dark | Token minted | Retrieved OK |
|----------|:-----:|:----:|:------------:|:------------:|
| Spearphishing email triage | ☐ | ☐ | ☐ | ☐ |
| Ransomware incident        | ☐ | ☐ | ☐ | ☐ |
| Suspicious domain triage   | ☐ | ☐ | ☐ | ☐ |
