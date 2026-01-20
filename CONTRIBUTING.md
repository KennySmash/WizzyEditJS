# Contributing to WizzyEdit

Thanks for helping build WizzyEdit. This project is designed to be collaborative across engineering + design, so contributions can be code, docs, UX flows, visual assets, or test fixtures.

## Ground Rules (project invariants)

Before proposing changes, please keep these invariants in mind:

1) **Canonical source is truth**
- The canonical text is the only authoritative representation.
- Saving returns canonical text exactly as edited.

2) **No silent rewriting**
- No implicit formatting.
- No implicit conversion.
- Any normalization/conversion must be an explicit user action and undoable.

3) **Everything is a transaction**
- All edits resolve to canonical text transactions.
- Undo/redo operates on transactions.

4) **Rendered views must be safe**
- No automatic code execution.
- Exfiltration-first: media same-origin, no frames/embeds, defuse scripts and unsafe URLs.
- Defusal never modifies canonical source.

5) **Validation never blocks saving**
- Diagnostics are debounced and visible.
- Saving may prompt/warn, but it’s always allowed.

If a proposal breaks an invariant, it needs a really strong justification and should be opened as a design discussion issue first.

---

## Ways to Contribute

### Design contributions
- UI layout for Source/Rich/Split views
- iconography for diagnostics/markers/defusal
- interaction patterns for undo/redo, template overwrite, conversions
- a11y improvements and keyboard-first flows

### Engineering contributions
- format registry implementations
- prefilter modules
- defusal policy rules
- content converters
- working copy recovery logic
- tests and fixtures

### Documentation contributions
- examples of supported formats
- usage guides for registries
- “how to write a plugin/prefilter/policy”
- screenshots and UX walkthroughs

---

## Issue Workflow

- If you’re unsure what to work on: open an issue describing what you want to improve.
- Prefer small, focused PRs.
- If your change affects UX or invariants: open a “Design discussion” issue first.

---

## Branch Naming

Use one of:
- `feature/<short-name>`
- `fix/<short-name>`
- `docs/<short-name>`
- `design/<short-name>`
- `chore/<short-name>`

Examples:
- `feature/xml-tree-editor`
- `design/diagnostics-panel`
- `fix/preview-defusal-url-check`

---

## Pull Request Checklist

Please ensure:
- [ ] The change respects project invariants (see above)
- [ ] The change has a clear description and screenshots (for UI changes)
- [ ] New behavior is covered by tests or fixtures where appropriate
- [ ] No silent formatting/conversion was introduced
- [ ] Defusal rules do not weaken safety guarantees
- [ ] Documentation updated if behavior/UX changes

---

## Coding Style

- Keep modules small and composable.
- Prefer explicit data contracts over implicit coupling.
- Avoid hidden magic: the goal is for this project to be readable by a mixed design/engineering guild.
- Any non-trivial algorithm (mapping, conversion, defusal) should have:
  - a short doc comment explaining intent
  - at least one test fixture

If the repo includes lint/format tooling, follow it. If not yet present, keep changes consistent with existing code.

---

## Commit Messages

Use a simple convention (pick one and stay consistent):
- `feat: ...`
- `fix: ...`
- `docs: ...`
- `chore: ...`
- `refactor: ...`

---

## Proposing Registry Modules

WizzyEdit is registry-driven. If you add a module, please include:

### Formats
- `id`, `extensions`, parse/validate/render contracts
- at least one sample fixture doc

### Prefilters
- marker extraction + validation
- render transformation behavior
- example markers + expected output

### Policies
- validation rules
- defusal behavior (include examples of blocked content and expected output)

### Converters
- document what is lossy
- warnings emitted
- example inputs/outputs

---

## Security Notes (Defusal)

Preview/rich rendering is a common exfiltration path. If you touch rendering/defusal:
- assume content can be hostile
- do not allow scripts, frames, embeds
- enforce same-origin for media
- block unsafe URLs (`javascript:` etc.)
- neutralize external CSS fetching (`@import`, non-same-origin `url(...)`)

If you’re unsure, open an issue before merging a change that loosens safety constraints.

---

## Getting Help

Open an issue if:
- you’re unsure how a change fits the invariants
- you need a design decision (e.g. interaction patterns)
- you want feedback on architecture direction

Thanks again for contributing ✨

---

## Testing

WizzyEdit is designed to be safe, predictable, and auditable. That only works if we keep the test suite healthy.

### Requirements
- **All existing tests must pass** before a PR can be merged.
- **New behavior must include tests**:
  - bug fixes → add a regression test that fails before the fix and passes after
  - new features → add coverage for the new behavior and key edge cases
  - refactors → keep or improve existing coverage; add tests if behavior becomes more complex
- If you change anything related to:
  - canonical text transactions
  - conversions
  - render mapping (rich ↔ source)
  - defusal/safety rules
  - prefilter marker extraction/substitution  
  …please add tests and fixtures that demonstrate correct behavior.

### What to test
At minimum, include tests/fixtures for:
- canonical text remains unchanged unless a user action/transaction modifies it
- undo/redo correctness (single transaction boundaries for “big” operations)
- defusal behavior (unsafe content is neutralized, no external loads/exfiltration paths)
- format parsing/validation diagnostics (ranges point to the right spot)
- prefilter marker extraction + substitution (and broken marker rendering with `<code>` errors)
- conversion output + warnings (lossy conversions must warn)

### CI
CI is treated as the source of truth. If CI fails, the PR should not be merged until it’s green.
If your change requires updating fixtures/snapshots, explain why in the PR description.

---

### Positive + Negative Tests (required)

For most changes, tests should include **both**:
- **Positive (“green path”) tests**: confirm allowed/safe content works normally.
- **Negative (“pass on failure”) tests**: confirm unsafe content is refused/defused and the test PASSES when the system blocks it.

This is particularly important in security and defusal code, where a “failure” is often the correct outcome.

#### Positive tests (examples)
Write tests that assert allowed behavior works and remains stable:

- Inline styles are allowed (but sanitized if needed)
- Normal links render
- Same-origin images render
- No diagnostics emitted for safe content

Example pattern:
```js
it('allows same-origin images', () => {
  const html = '<img src="/assets/a.png">'
  const { html: out, diagnostics } = defuse(html, { origin: 'https://example.test' })
  expect(out).toContain('<img')
  expect(diagnostics).toHaveLength(0)
})



