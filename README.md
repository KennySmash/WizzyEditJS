# WizzyEdit

WizzyEdit is a general-purpose, registry-driven editor for **one canonical document at a time**.

It’s designed for document-heavy systems where **the source text is the truth**:
- What you edit is what gets saved.
- No silent conversions.
- No automatic reformatting.
- Everything is undoable.

WizzyEdit provides two editable views over the same canonical document:
- **Source View** — edit the canonical text directly.
- **Rich View** — edit an interactive rendered view that translates actions back into canonical text transactions.

It also supports:
- **Multiple formats** via a format registry (HTML/XHTML/XML, GFM Markdown, plaintext)
- **Prefilters** via a prefilter registrar (marker extraction + preview substitution without mutating canonical source)
- **Policies** via a policy registry (validation + defusal rules for safe rendering)
- **Content conversion** via registered converters (explicit, undoable, warnings supported)
- **Working copy recovery** in the browser (refresh/crash resilience)
- **Non-blocking validation** (debounced diagnostics; saving is always allowed)

> Design mantra: **Text is the truth. Everything else is derived.**

---

## Key Concepts

### Canonical Document
WizzyEdit edits exactly **one** canonical document at a time:
- `canon.text` — the canonical source string
- `canon.formatId` — `html | xhtml | xml | gfm | text` (extensible)
- `canon.encoding` — `utf-8 | utf-16`

Saving returns `canon.text` exactly as it exists in the editor.

### Transactions
All edits are represented as **transactions** over canonical text:
- typing
- paste
- rich edits
- template overwrite
- format actions
- conversions

Undo/redo operates on transactions.

### Registries
WizzyEdit stays general by using registries:

- **Format registry**: parse / validate / render / rich-edit mapping
- **Prefilter registry**: detect markers + substitute into render output (never edits canon)
- **Policy registry**: additional validation + render defusal rules
- **Plugin registry**: commands + UI contributions
- **Converter registry**: explicit content conversion between formats

---

## Safety / Defusal (exfiltration-first)
Rendered views must be safe by default:
- No automatically running code (e.g. scripts)
- No frames/embeds (iframe/object/embed)
- Media must be **same-origin** (img/audio/video)
- Inline CSS is allowed, but external calls (e.g. `@import`, non-same-origin `url(...)`) are blocked/neutralized
- Defusal never changes canonical source

---

## Repository Status
This repo is new and intentionally modular. Early contributions are welcome in:
- design system + UI layout
- editor interactions and ergonomics
- accessibility improvements
- icons and visual language for diagnostics/markers
- documentation, examples, and demo fixtures

---

## Planned Structure (high level)

- `src/core/` – canonical model, transactions, history, derived indices
- `src/registries/` – registries and base interfaces
- `src/formats/` – format implementations (html/xhtml/xml/gfm/text)
- `src/prefilters/` – optional prefilter modules (example: xhtml span markers)
- `src/policies/` – validation rules + defusal implementation
- `src/ui/` – Vue components (source view, rich view, split view, panels)
- `docs/` – specs, design notes, contribution references

(Exact folder layout may evolve; keep modules loosely coupled.)

---

## Contributing
We love PRs. See [CONTRIBUTING.md](./CONTRIBUTING.md) for:
- how to pick an issue
- branch naming
- PR checklist
- code style
- how to propose design changes
