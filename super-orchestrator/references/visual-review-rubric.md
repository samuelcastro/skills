# Visual review rubric — super-orchestrator validate-visual stage

**Purpose:** Deterministic 5-criterion checklist used by a validate-visual worker to grade screenshots against acceptance criteria. Without this rubric, two iters can grade the same screenshot opposite ways and trigger false oscillation that burns the iteration budget.

**Pass = 5/5.** Any single FAIL fails the AC for that screenshot.

The rubric assumes you have a design source of truth (`DESIGN.md`, design tokens file, Figma export, or wireframe) and at least one rendered screenshot per acceptance criterion.

---

## The 5 criteria

For every `<AC, screenshot>` pair the worker grades, check ALL FIVE:

### 1. Typography matches design tokens

- Font family is the one your design source specifies for that surface (body / heading / mono).
- Font size matches a documented scale token (no off-token "looks roughly right" sizes).
- Weight matches the spec.
- Line-height feels in-range (visual estimate within ±10%).

**FAIL example:** body text rendered in serif on a page that specifies sans-serif.

### 2. Colors only from the documented palette

- No legacy/off-brand colors leaking through. Compare against your design tokens.
- Background, foreground, accent, border, and state colors all visibly come from the palette.
- Status hues (success / warn / error) match documented state colors, not arbitrary CSS keywords.

**FAIL example:** primary CTA button rendered in a color that conflicts with the documented primary. Even one clear violation = FAIL.

### 3. Spacing on the spec'd grid

- Vertical rhythm between sections is a multiple of your spacing token (e.g. 4 or 8 px).
- Form field padding (input height, label-to-field gap) matches the documented form spec.
- Card / panel internal padding consistent across the page.

**FAIL example:** a 13-px gap between heading and first form field where the spec calls for 16 or 24.

### 4. Interactive states (hover / active / disabled / error) visible per the AC's flow

If the AC narrates an interaction (click, focus, error path), the screenshots in that flow must SHOW the resulting state correctly:

- Focused inputs show the focus ring.
- Disabled buttons render with the disabled palette + cursor.
- Error states render the inline error message with the error color.
- Hover/active states only need verification if the AC explicitly mentions them.

**FAIL example:** AC says "submit button disabled while form invalid"; the screenshot shows the button at full opacity with a normal cursor.

### 5. Layout matches the wireframe / reference

- Compare the screenshot against the relevant frame in your wireframe or reference design.
- Allow translation tolerance: the wireframe is a sketch — exact pixel alignment is NOT required, but the **structural relationships** must hold (column counts, primary-vs-secondary order, what's above-the-fold, presence/absence of nav rails, etc.).

**FAIL example:** wireframe shows a left-rail nav + main panel; screenshot has no left rail.

---

## Verdict-block format

Every AC gets a YAML block in your phase visual-review file (e.g. `docs/phase-loop/phase-N-visual-review.md`). Worker writes one per AC:

```yaml visual-verdict
ac: AC-SET-02
spec_quote: "RMB payment method triggers exchange-rate input"
screenshot: QA-report/payment-methods/tc-2/en/02-ac-set-02-toggle-rmb-checkbox.jpg
criteria:
  typography: PASS
  palette: PASS
  spacing: PASS
  interactive_states: FAIL — checkbox is shown checked but no exchange-rate input is rendered
  layout: PASS
visual_verdict: FAIL
fail_reason: "criteria #4 — RMB checkbox is checked but the exchange-rate input never appears in the captured frame; AC requires it to be visible after the toggle."
```

`visual_verdict: PASS` requires every criterion to be `PASS`. Any single FAIL → `visual_verdict: FAIL`.

---

## Worked example: PASS case

```yaml visual-verdict
ac: AC-SET-01
spec_quote: "Settings page renders with left-rail nav and main panel"
screenshot: QA-report/settings/tc-1/en/01-ac-set-01-settings-renders.jpg
criteria:
  typography: PASS
  palette: PASS
  spacing: PASS
  interactive_states: PASS
  layout: PASS
visual_verdict: PASS
fail_reason: null
```

## Worked example: FAIL case (palette)

```yaml visual-verdict
ac: AC-SET-04
spec_quote: "Save button uses the primary CTA color"
screenshot: QA-report/settings/tc-1/en/04-ac-set-04-save-clicked.jpg
criteria:
  typography: PASS
  palette: FAIL — Save button rendered in a legacy/off-brand color. The design tokens specify a different accent color.
  spacing: PASS
  interactive_states: PASS
  layout: PASS
visual_verdict: FAIL
fail_reason: "criteria #2 — primary CTA uses a legacy/off-brand color; the design tokens mandate the documented accent."
```

---

## Worker discipline

- Read your design source of truth ONCE at the start of the validate-visual iter, capture the palette / type-scale to a scratch buffer (do not re-read between ACs).
- Open the AC's section of the spec ONCE. Quote the relevant line into `spec_quote`.
- Open each screenshot via the multimodal `Read` tool and grade.
- Be terse on PASS. Be specific on FAIL — name the criterion number and what's wrong.
- A FAIL here triggers a FIX iter that pins the bug in code with a red test before fixing — do not be sloppy with FAIL reasoning, every FAIL costs a fix iteration.
