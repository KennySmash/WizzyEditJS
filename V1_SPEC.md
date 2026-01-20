# WizzyEdit V1 Spec Sheet

## 1) Summary

WizzyEdit is a general-purpose, registry-driven editor for **one canonical document at a time**.

Core rule: **Text is the truth. Everything else is derived.**

WizzyEdit provides:
- An **editable Source view** (direct canonical text editing)
- An **editable Rich view** (interactive rendered editing mapped back to canonical text transactions)
- Debounced **validation + diagnostics** (never blocks save)
- **Content conversion** where converters exist (explicit, undoable, warning-capable)
- Pluggable **prefilters** (marker extraction + render substitution without mutating canonical text)
- Pluggable **policies** (validation + render safety/defusal)
- A **working copy** for crash/refresh recovery (localStorage when appropriate)

---

## 2) Non-Negotiable Guarantees

### Canonical source truth
- WizzyEdit edits exactly **one canonical source string** at a time.
- Saving returns **exactly** the canonical source (post-edit), plus format/encoding metadata.
- No implicit normalization, conversion, or formatting.

### Dirty tracking
- Any change (including whitespace) marks the document dirty.

### Explicit-only formatting/conversion
- Any normalization/formatting/conversion is **user-driven** and **undoable**.

### Validation is non-blocking
- Validation can warn/prompt, but **never blocks save**.

### Render safety is exfiltration-first
Rendered output must:
- run **no automatically executing code**
- block common exfil paths
- preserve content visibility by neutralizing/wrapping instead of silently deleting
- never mutate canonical source during defusal

### Recovery
- WizzyEdit maintains a working copy sufficient to recover after refresh/crash.

---

## 3) Views (Both Editable)

### Source view (canonical editor)
- Direct editing of canonical text.
- Paste inserts the pasted string literally.
- If paste includes rich formatting and the format supports it, represent formatting in source-level constructs (e.g. inline `style=""`).

### Rich view (interactive rendered editor)
Rich view is fully interactive:
- users type directly into the rendered view
- edits are translated into canonical **text transactions**
- selection and cursor movement are mapped between views

Rich view behavior per format:
- **HTML/XHTML:** page-like rendering + direct editing, mapped back to canonical HTML/XHTML text.
- **GFM Markdown:** rendered markdown editing mapped back to canonical markdown text.
- **XML:** **tree editor** (nodes/attributes/values) mapped back to canonical XML text.

### View configuration
Host app may configure:
- Source only / Rich only / Split view
- default view + toggle availability

---

## 4) Intermediary Document Model (IDM)

WizzyEdit is **text-first**.

### Canonical buffer
- `canon.text: string`
- `canon.formatId: string`
- `canon.encoding: 'utf-8' | 'utf-16'` (default `utf-8`)
- `canon.selection: { from: number, to: number }` (UTF-16 offsets)

### Transactions (all mutations)
All document changes are applied as transactions:

- `Edit { from: number, to: number, insert: string }`
- `Transaction { edits: Edit[], selection?: Selection, meta?: object }`

Everything must emit transactions:
- typing
- paste
- rich edits
- template overwrite
- formatting actions
- conversions

### Undo/Redo
- Undo/redo operates over transactions.
- Multi-edit operations (formatting/conversion) must be a single undoable transaction.

### Derived indices (caches)
Recomputed on debounce; never authoritative:
- `ParseIndex` (best-effort parse output; errors become diagnostics)
- `MarkerIndex` (prefilter markers extracted from canonical text; optional)
- `DiagnosticsIndex` (aggregated diagnostics from format/prefilters/policies/plugins/converters)
- `RenderProjection` (render output + mapping hints)

---

## 5) Registries

WizzyEdit is generalized by registries.

### Format registry

**FormatDefinition**
- `id: string`
- `extensions: string[]` (host-defined allowed extensions)
- `parse(canon): ParseIndex` (best-effort; must not throw)
- `validate(canon, parse?): Diagnostic[]`
- `renderRich(canon, parse?, context): RichProjection`
- `applyRichEdits(richEdits, canon, parse?, context): Transaction[]`
- `renderPreview(canon, parse?, context): PreviewOutput`

RichProjection requirements:
- sufficient mapping metadata to translate user edits back into canonical ranges
- selection mapping between rich ↔ canonical

Supported format IDs (baseline):
- `html`, `xhtml`, `xml`, `gfm`, `text`
(Host can extend.)

### Prefilter registrar

Prefilters are optional modules that:
- extract and validate markers from canonical source
- transform render output by substituting resolved values
- never mutate canonical source

**PrefilterDefinition**
- `id: string`
- `appliesToFormats: string[]`
- `extractMarkers(canon, parse?): Marker[]`
- `validateMarkers(canon, markers): Diagnostic[]`
- `transformRender(renderOutput, canon, markers, runtimeContext): RenderOutputWithMapping`

### Policy registry

Policies contribute:
- additional diagnostics
- defusal rules (render safety)

**PolicyDefinition**
- `id: string`
- `appliesToFormats: string[]`
- `validate(canon, parse?, markers?, context): Diagnostic[]`
- `defuse(renderHtml, context): { html: string, diagnostics: Diagnostic[] }`

### Plugin registry

Plugins add:
- commands (emit transactions)
- UI contributions (toolbars/panels/inspectors)
- optional validators

Constraint: plugins may only mutate canonical text by emitting transactions.

### Converter registry

Converters enable explicit content conversion where available.

**ConverterDefinition**
- `id: string`
- `fromFormatId: string`
- `toFormatId: string`
- `convert(canon, parse?, context): { text: string, warnings: Diagnostic[] }`

Conversion rules:
- explicit user action
- warnings shown
- conversion replaces entire canonical text
- single undoable transaction

---

## 6) Templates, Switching, Conversion

### Templates
Host provides templates:
- `title`, `description`, `extension`, `content: string`
Templates are literal content.

### Template overwrite
- If canonical is blank: overwrite without prompt.
- If canonical is non-blank: confirmation required.
- Overwrite replaces entire canonical text and is undoable.

### Content conversion
- Available only when a converter exists for a (from,to) pair.
- Conversion replaces entire canonical text and is undoable.
- Lossiness allowed; warnings required.

---

## 7) Example Prefilter Module: XHTML `<span>` markers (optional)

This module is registered via the prefilter registrar.

### Applicability
Typically applies to `xml` and `xhtml` (host decides), optionally `html`.

### Marker recognition
Markers are `<span>` elements containing any of:

1) Preferred: namespaced attribute `*:type="KEY"` where the namespace URI is `urn:cognidox:xhtml`
2) `rel="cognidox:type:KEY"` (case-insensitive prefix match)
3) `data-prefilter="KEY"`

### Namespace strictness
- Namespaced markers are validated by **namespace URI match**, not prefix name.

### Conflicts / precedence
If multiple marker attributes exist on the same span:
- `*:type` is authoritative
- emit warning describing ignored attributes

### Marker validity
Invalid if:
- unknown key, or
- malformed marker syntax (missing values, malformed prefix, etc.)

### Allowed key set (baseline)
- `partnum`, `title`, `version`, `author`, `issuer`, `issuername`, `date`, `versiontype`
(Host may extend.)

### Rendering substitution
- Substitute marker inner content with resolved values from runtime context/meta.
- If invalid: render a visible `<code>` error block in place of the broken marker and emit a diagnostic tied to canonical range.

### Mapping
Rendered marker output carries mapping metadata for click-through to source.

---

## 8) Render Safety / Defusal (Exfiltration-first)

Rendered HTML must be defused to prevent automatic execution and exfiltration.

### Block entirely (neutralize/wrap as code)
- `<script>`
- `<iframe>`, `<frame>`, `<frameset>`
- `<object>`, `<embed>`

### Neutralize attributes
- strip/neutralize inline event handlers (`on*`)
- sanitize URL attributes:
  - block `javascript:`, `vbscript:`
  - block `data:` unless explicitly allowed by policy
  - block protocol-relative `//...` unless same-origin is guaranteed

### Media restrictions (same-origin only)
Allowed only if same-origin:
- `<img src>`
- `<audio src>`
- `<video src>`
If not same-origin:
- replace element with visible `<code>` explanation
- emit diagnostic

### CSS rules
- inline `style=""` is allowed
- block external-loading CSS constructs:
  - `@import`
  - `url(...)` that is not same-origin
Emit diagnostics for blocked CSS constructs.

### Defusal behavior constraints
- preserve content visibility where possible
- defusal never mutates canonical source

---

## 9) Validation & Diagnostics

### Debounced validation
- parse + validate run on debounce after edits.

### Diagnostics contract
Diagnostics must include:
- severity: `info | warning | error`
- message
- canonical range `{ from, to }` when possible
- source: `format:<id>` | `prefilter:<id>` | `policy:<id>` | `plugin:<id>` | `converter:<id>`
- optional quick-fix actions (emit transactions)

### Save behavior
- save is always allowed
- if errors exist, prompt/warn but proceed

---

## 10) Working Copy Recovery

### Stored data
Working copy stores:
- canonical text
- formatId
- encoding
- timestamp
- optional meta fingerprint

### Keying
Key format:
- `wizzy:{hostNamespace}:{docKey}:{formatId}`

Where:
- `hostNamespace` provided by host (default `default`)
- `docKey` stable identity (e.g., partnum+version, or UUID)

### Restore flow
On load:
- if working copy exists and differs from initial content, offer Restore/Discard.

### Write policy
- debounced writes
- clear on successful save (configurable)

---

## 11) Testing Expectations (stack-agnostic)

### All tests must pass
- All existing tests must pass for merges.

### Positive + Negative tests required
For security/defusal and other critical paths, include:
- **Positive (“green path”) tests**: allowed behavior remains allowed.
- **Negative (“pass on failure”) tests**: unsafe content is blocked/defused and the test passes when refusal occurs.

Security tests must verify:
1) unsafe content is not present/active in output
2) diagnostics are emitted and/or visible `<code>` explanation appears
3) canonical source remains unchanged when testing render/defusal paths

---

## 12) Acceptance Criteria

WizzyEdit V1 is complete when:
1) one canonical source text is edited and saved exactly
2) dirty triggers on any change including whitespace
3) Source view and Rich view are both editable and remain consistent via transactions
4) rendering behaves per format (HTML page-like, Markdown rendered, XML tree)
5) prefilters can be registered and transform rendered output without mutating canonical text
6) invalid markers render with visible `<code>` error blocks and produce diagnostics
7) defusal prevents execution + blocks exfiltration (no scripts, no frames/embeds, media same-origin, CSS external calls blocked)
8) validation is debounced and non-blocking
9) template overwrite prompts when canon non-blank and is undoable
10) content conversion exists where converters are registered; warnings emitted; undoable
11) working copy recovery restores after refresh using docKey
