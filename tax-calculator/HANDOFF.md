# Brief: Singapore Income Tax & CPF Calculator — v1, 17 Aug 2026

> **How to use this file.** Paste it into a new chat as the first message. It contains
> everything needed to rebuild, extend, or restyle the tool without seeing the original
> conversation. Section 6 (Design & Tone) is written to be **portable** — lift it on its
> own to apply the same visual language to a different page.
>
> The built artefact is `tax-calculator/index.html` in `bensoon129/benPortfolio`, branch
> `claude/sg-income-tax-calculator-goa7xv`. One file, 1,941 lines, no dependencies.

---

## 0. THE ORIGINAL PROMPT (verbatim)

> generate me an income tax calculator app for sg. i want to be able to input my net income,
> see my gross income, see the employer contribution amount, then give me options to input any
> other deductibles that can affect my income tax. input into it a suggestion on how i can
> reduce income tax as well. find all possible ways to reduce tax, but state all pros and cons
> on doing that action. for example, putting into SRS. you can invest using SRS, but it's stuck
> in there till retirement. if you withdraw before that, there is a higher tax and if you
> withdraw after your retirement, you still have to pay tax on that SRS withdrawal amount.

Follow-up prompt (became section 4 of the app):

> i also want to be able to do projections. like example, if i invest x amount every year to SRS
> for example, how much savings from tax will i get and how much i will have saved

---

## 1. OBJECTIVE

A single self-contained HTML page that turns one number a Singaporean actually knows — monthly
take-home pay — into a complete picture of gross salary, CPF (both sides), income tax, and a
ranked list of legal ways to reduce that tax with the trade-offs stated, plus a multi-year
projection of what contributing to a scheme actually yields.

**Done when:**
- Entering a monthly take-home figure produces gross, employee CPF, employer CPF and annual tax
  with no further input.
- Every tax figure matches IRAS's published cumulative tax table exactly at all 12 band edges.
- Every reduction suggestion shows a dollar saving computed at the user's own marginal rate, and
  carries both pros and cons.
- The projection's compounding matches the closed-form annuity-due formula to within $1.
- The page renders correctly at 360 px and 1280 px, in light and dark, with zero console errors.

---

## 2. CONTEXT

### What exists
| Path | What it is |
|---|---|
| `tax-calculator/index.html` | The whole app. HTML + inline `<style>` + inline `<script>`. No build step, no external requests. |
| `index.html` (repo root) | Ben's existing Bootstrap 3 portfolio. **Untouched.** The calculator is deliberately standalone. |

### Prior decisions that must not be reversed without a reason
1. **Single file, zero dependencies.** No npm, no CDN, no fonts, no bundler. It must open from
   `file://` and work offline.
2. **The calculator is not linked from the portfolio nav.** Left alone deliberately — adding it
   touches the main page layout, which is Ben's call.
3. **Singapore has no monthly tax withholding.** Take-home = gross − employee CPF. Tax is a
   separate annual bill. The UI states this explicitly; do not model tax as a payslip deduction.
4. **`assess()` is the single assessment core.** Both the live calculator and the projection call
   it. Do not let a second copy of the tax logic appear anywhere.

### Verification harness (not in the repo — recreate if needed)
A Node script reads `index.html`, extracts the `<script>` body with a regex, runs it in a `vm`
context against a stub DOM (`getElementById` returns a generic object per id; `querySelector`
resolves radio groups by name; stub `window.addEventListener`), then appends
`globalThis.__m = model(); globalThis.__project = project; …` to reach the internals. 85
assertions currently pass. This is the cheapest way to test the file without a browser.

---

## 3. SCOPE

**IN**
- Singapore **tax-resident** individuals, employed, private sector.
- Citizens/PRs (CPF applies) and foreigners on EP/S Pass (no CPF, higher SRS cap).
- Income years 2025 (YA 2026, confirmed) and 2026 (YA 2027, provisional).
- All personal reliefs currently claimable, the $80,000 relief cap, and 250% IPC donations.
- Multi-year projection for SRS and CPF cash top-ups.

**OUT — explicit non-goals**
- Non-resident taxation (flat 15% on employment income or resident rates, whichever is higher;
  24% on most other income). Not modelled at all.
- Self-employed / trade income CPF and MediSave obligations.
- The Not Ordinarily Resident scheme, and any tax treaty relief.
- Course Fees Relief (withdrawn from YA 2026) and Foreign Domestic Worker Levy Relief (lapsed
  after YA 2024) — deliberately absent, and named in the UI as no longer available.
- Public sector CPF rates; the low-wage CPF phase-in below $750/month is approximated, not exact.
- Any saving, accounts, analytics, or network calls.

---

## 4. REQUIREMENTS

### Pay reconstruction
- **R1** Given monthly take-home, solve monthly gross by bisection over the CPF curve
  (200 iterations, bracket `[net, 2·net + 10000]`). Bisection is used deliberately: the function
  is monotonic, and this handles the Ordinary Wage ceiling kink without special-casing.
- **R2** A toggle MUST let the user enter gross instead of take-home; the rest of the page is identical.
- **R3** Employee CPF on Ordinary Wages: 0 at or below $500/month; linear phase-in
  `eeRate × 750 × (gross − 500) / 250` between $500 and $750; full rate on `min(gross, OW ceiling)`
  above $750. Employer pays the full rate on `min(gross, OW ceiling)` whenever gross exceeds $50.
- **R4** Additional Wage (bonus) ceiling = `102,000 − (OW subject to CPF for the year)`. Bonus is
  entered as a number of months of gross pay.
- **R5** Display monthly gross, employee CPF, monthly take-home, bonus, CPF on bonus, annual gross,
  annual employee CPF, annual employer CPF, both rates, total into CPF, and total employer cost.
- **R6** Foreigners: all CPF figures are zero, gross = take-home, no CPF Relief, SRS cap $35,700.

### Tax computation
- **R7** Resident bands (unchanged since YA 2024), as `[band width, rate]`:
  `[20000,0] [10000,.02] [10000,.035] [40000,.07] [40000,.115] [40000,.15] [40000,.18] [40000,.19]
  [40000,.195] [40000,.20] [180000,.22] [500000,.23] [∞,.24]`
- **R8** Marginal rate = the rate the **next** dollar pays. At an exact band edge this is the band
  above (`chargeable < cumulativeEdge`, strictly). The bracket ladder highlight MUST use the same
  band index as the marginal-rate chip.
- **R9** Order of operations: `Total income → less allowable expenses and 250% donations =
  Assessable income → less personal reliefs (capped at $80,000) = Chargeable income → tax →
  less rebates = Net tax payable`.
- **R10** Donations reduce **assessable** income and therefore sit **outside** the $80,000 cap.
  The Parenthood Tax Rebate is a rebate against tax payable, also outside the cap.

### Reliefs
- **R11** Auto-derived and shown read-only: Earned Income Relief (by age) and CPF Relief (= computed
  compulsory employee CPF).
- **R12** User-entered, grouped in `<details>` accordions with a live subtotal in each summary:
  retirement & savings, family, National Service, donations.
- **R13** Each input MUST be clamped to its own statutory cap before it enters the sum.
- **R14** The $80,000 cap applies to the **sum** of reliefs, after clamping. Surface the wasted
  amount when it binds.

### Suggestions ("ways to pay less tax")
- **R15** Each card computes `taxOn(chargeable) − taxOn(chargeable − min(headroom, roomLeft))`,
  where `headroom = max(0, 80000 − reliefRaw)`. A saving of $0 is shown as $0, never hidden.
- **R16** Cards are sorted by that saving, descending. Every card MUST carry a non-empty pros list
  **and** a non-empty cons list.
- **R17** Cards MUST render even when they save nothing, styled as `dead`/`info`, so the user learns
  why a lever doesn't apply to them.

### Projection
- **R18** Iterate year by year from the user's current age. Each year: grow salary by `g`,
  recompute CPF and Earned Income Relief at the new age and income, assess with and without the
  contribution, and record the difference as that year's tax saving.
- **R19** Strip the user's existing entry for the projected vehicle out of the relief base so the
  projection **replaces** it rather than stacking on top of it.
- **R20** Contributions occur at the **start** of each year: `balance = (balance + c) × (1 + r)`.
  This MUST equal `C(1+r)((1+r)^N − 1)/r` when `c` is constant.
- **R21** The comparison arm invests **the same net cost** — `contribution − taxSaved` — each year
  at the same return, and is never taxed (Singapore does not tax individual investment gains).
  This is the only honest counterfactual; compounding the gross contribution against nothing would
  flatter the scheme.
- **R22** SRS drawdown: 10 withdrawals from the withdrawal age, `w = balance / (10 − k)` with the
  remainder still compounding. 50% of each withdrawal is taxable, stacked on any other retirement
  income entered. CPF: identical schedule, zero tax.
- **R23** Flag every projected year where adding the contribution pushes total reliefs past
  $80,000, and shade those rows in the table.
- **R24** Report a **negative** edge when the scheme loses to investing outside it, with a notice
  explaining why. Do not assume the scheme wins.

---

## 5. CONSTRAINTS

- Vanilla HTML/CSS/JS in one file. No frameworks, no build, no external requests of any kind.
- Must work opened directly from disk (`file://`).
- All state lives in the DOM. `render()` is called on every `input` and `change` event on
  `document`, reads all inputs, and rewrites all outputs. There is no state object to keep in sync.
- Full recalculation on every keystroke MUST stay imperceptible (it is ~50 assessments plus a
  chart rebuild; currently no measurable lag).
- Accessibility floor, never negotiable: keyboard-focusable controls with visible `:focus-visible`,
  body text contrast ≥ 4.5:1 in both themes, `prefers-reduced-motion` honoured, no horizontal page
  scroll at 360 px.

---

## 6. DESIGN & TONE — *the portable section*

### 6.1 The brief that produced it
- **Subject / audience / job:** Singapore personal income tax and CPF · a salaried worker who only
  knows their take-home · turn one number into a full assessment plus a ranked list of levers.
- **Signature element:** the **bracket ladder** — all 12 tax bands drawn as rungs, each filling with
  the slice of income that lands in it, the marginal rung flagged, and a green ghost marker showing
  where the top suggestion would move you. Spend the boldness here; keep everything around it quiet.
- **Rejected as default** (say why, so it isn't reintroduced):
  - Cream background + high-contrast serif + terracotta accent — the most recognisable
    AI-default look. Replaced with cool paper + tabular mono.
  - A grid of equal-weight KPI tiles — hierarchy must encode what matters, so one dominant figure
    plus the ladder instead.

### 6.2 Colour tokens — light (`:root`)
```css
--bg:#F2F3F7;        --surface:#FFFFFF;   --surface-2:#F7F8FB;  --surface-3:#ECEFF6;
--ink:#16192B;       --ink-2:#565C75;     --ink-3:#858BA3;
--line:#DEE1EA;      --line-2:#C9CEDC;
--accent:#2B4EE6;    --accent-ink:#1B34A8; --accent-soft:#E7EBFF;
--keep:#0E7A57;      --keep-soft:#DDF3EA;   /* money you keep */
--tax:#C4353F;       --tax-soft:#FBE4E5;    /* money you lose */
--warn:#8A5209;      --warn-soft:#FBEEDA;
--radius:10px;       --radius-sm:6px;
--shadow:0 1px 2px rgba(22,25,43,.06), 0 6px 20px -12px rgba(22,25,43,.28);
```

### 6.3 Colour tokens — dark (`@media (prefers-color-scheme: dark)`)
Selected for the dark surface, **not** an automatic inversion. Never pure black; three elevated
surface levels; accents desaturated.
```css
--bg:#101220;        --surface:#191C2C;   --surface-2:#1F2334;  --surface-3:#272C40;
--ink:#EDEFF6;       --ink-2:#A7ADC2;     --ink-3:#7E859C;
--line:#2E3348;      --line-2:#3C4259;
--accent:#7D93FF;    --accent-ink:#A9B7FF; --accent-soft:#242A48;
--keep:#4ECFA1;      --keep-soft:#16302A;
--tax:#FF8A90;       --tax-soft:#361D22;
--warn:#E5B057;      --warn-soft:#33270F;
--shadow:0 1px 2px rgba(0,0,0,.3), 0 8px 24px -14px rgba(0,0,0,.8);
```

### 6.4 Typography — two roles, both offline-safe
```css
--sans: -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,"Helvetica Neue",Arial,sans-serif;
--mono: ui-monospace,"SF Mono",SFMono-Regular,"Cascadia Mono","Roboto Mono",Menlo,Consolas,monospace;
```
- **The rule that carries the personality: every money figure is monospace with
  `font-variant-numeric: tabular-nums` and `letter-spacing:-.03em`.** This is the "ledger voice",
  and it is why the page doesn't look like a generic form. Apply it via `.num` or per-component.
- Headings: `--sans`, weight 650, `letter-spacing:-.02em`, `line-height:1.2`.
- Body: `--sans`, 15px, `line-height:1.55`.
- Scale in use: 10.5 / 11 / 12 / 13 / 15 / 17 / 19 / 22 / 30.
- Eyebrows and stat keys: 10.5–11px, `letter-spacing:.08–.14em`, uppercase, weight 650–700.

### 6.5 Layout
- Shell: `.wrap { max-width:1220px; padding:0 20px }`.
- Two columns: `grid-template-columns: minmax(0,380px) minmax(0,1fr)`, gap 28px. Collapses to one
  column below 940px.
- Left rail is `position:sticky; top:76px` with its own `overflow-y:auto` — the payslip stub.
- Right column is the assessment — the Notice of Assessment.
- A sticky `.tape` bar under the masthead holds six headline figures and stays visible while
  scrolling (`z-index:30`).
- Spacing sits on a 4/8px rhythm.

### 6.6 Component conventions
- `.card` — surface + 1px `--line` border + `--radius` + `--shadow`. `.card-head` / `.card-body`.
- `.ledger` — two-column table; labels in `--ink-2`, figures right-aligned in mono; dotted row
  separators; `tr.total` gets a 2px solid `--ink` top border; `tr.sub` is indented and muted.
- `.seg` — segmented radio control. The `<input>` is visually hidden inside its `<label>`; the
  checked state is `background:var(--accent)` with the label text set to `var(--surface)` so it
  stays legible in **both** themes. (Hiding the input means automated clicks must target the
  `<label>`, not the input.)
- `details.group` — accordion with a live subtotal in the summary. **The disclosure marker is a
  rotating CSS chevron, never `+`/`−`** — a minus sign next to a dollar figure reads as negative.
- `.move` — a suggestion card: title, one-line explanation, big green saving, chips, then a
  two-column pros/cons split with `+` and `−` pseudo-element bullets in `--keep` and `--tax`.
- `.notice` — inline `info` (accent) or `warn` (amber) banner.
- `.stats` — bordered tile grid, `repeat(3, minmax(0,1fr))`, stepping to 2 then 1 column. Fixed at
  3 rather than `auto-fit` because six tiles must fill two clean rows with no dead cells.
- Motion: 150–250 ms, eased, and only on state that actually changes. No decorative animation.

### 6.7 Chart palette — validated, do not eyeball substitutions
Categorical slots 1 / 3 / 2 from a CVD-validated set, re-stepped for the dark surface:

| Role | Light | Dark |
|---|---|---|
| `--s-contrib` What you put in | `#2a78d6` | `#3987e5` |
| `--s-growth` Investment growth | `#1baf7a` | `#199e70` |
| `--s-alt` Same net cost, outside | `#eb6834` | `#d95926` |

Validated all-pairs in both modes: worst CVD ΔE 9.2 light / 9.4 dark (≥8 target); worst
normal-vision ΔE 24.0 light / 20.9 dark (≥15 floor). Light-mode `#1baf7a` sits at 2.82:1 against
white, which **obligates relief** — the page ships direct labels *and* a year-by-year table, so
identity is never colour-alone. If you change a hue, re-validate; do not reason about it.

Chart rules in force: stacked area for the part-to-whole balance, dashed 2px line for the
comparison arm, 2px surface-coloured stroke as the gap between stacked fills, recessive
`--line` gridlines, crosshair + tooltip on hover, legend always present, direct labels only at the
final year with 26px collision nudging. **Never a dual y-axis.**

### 6.8 Responsive chart
The SVG is built at the container's **true pixel width** (`chartGeom()` reads
`$('pChart').clientWidth`, clamps to 300–760) so one viewBox unit equals one CSS pixel and axis
text never shrinks below its stated size. Below 520px it switches to a shorter chart, tighter
margins, four x-ticks, and drops the direct labels. A debounced `resize` listener re-renders only
when the width actually changes.

### 6.9 Voice
- Plain, specific, and never salesy. "Locked until the statutory retirement age", not "long-term
  commitment".
- Name the cost in dollars: "Give $1,000 and you save $550 — a net cost of $450."
- Label controls by what the user does. Keep a name stable through its whole flow.
- State the uncomfortable thing rather than burying it: "Every additional dollar of SRS now saves
  you exactly $0 in tax while still being locked away."
- Never imply certainty about future policy. YA 2027 is labelled provisional everywhere it appears.

---

## 7. DATA — the domain rules, with the numbers

> Every figure below was checked against IRAS/CPF published material in Aug 2026. **Re-verify
> before relying on it** — the Budget moves these.

### 7.1 Resident tax bands (YA 2024 onward)
Cumulative tax at each band edge, used as the test oracle:

| Chargeable income | Cumulative tax | | Chargeable income | Cumulative tax |
|---|---|---|---|---|
| $20,000 | $0 | | $240,000 | $28,750 |
| $30,000 | $200 | | $280,000 | $36,550 |
| $40,000 | $550 | | $320,000 | $44,550 |
| $80,000 | $3,350 | | $500,000 | $84,150 |
| $120,000 | $7,950 | | $1,000,000 | $199,150 |
| $160,000 | $13,950 | | above $1,000,000 | 24% |
| $200,000 | $21,150 | | | |

### 7.2 CPF configuration
```js
2026 (→ YA 2027): owCeiling 8000,  annualWageCeiling 102000, annualLimit 37740
  ≤55: er .17  ee .20   |  >55–60: er .16   ee .18
  >60–65: er .125 ee .125 | >65–70: er .09  ee .075  |  >70: er .075 ee .05
2025 (→ YA 2026): owCeiling 7400,  annualWageCeiling 102000, annualLimit 37740
  ≤55: er .17  ee .20   |  >55–60: er .155 ee .17
  >60–65: er .12  ee .115 | >65–70: er .09  ee .075  |  >70: er .075 ee .05
```
Sanity check that must hold: a high earner's total CPF is exactly `37% × 102,000 = $37,740`, which
is the CPF Annual Limit. Senior rates rose on 1 Jan 2026 and are legislated to rise again in 2027.

### 7.3 Caps and thresholds
| Constant | Value |
|---|---|
| Personal relief cap | $80,000 per YA |
| SRS cap — Citizen/PR | $15,300 |
| SRS cap — foreigner | $35,700 |
| CPF cash top-up relief | $8,000 self **+** $8,000 family (separate caps) |
| Voluntary MediSave room | $37,740 − mandatory CPF for the year |
| Life Insurance Relief | only if employee CPF < $5,000; capped at 7% of sum assured |
| IPC donations | 250% deduction, **outside** the relief cap; 250% rate runs to 31 Dec 2026 |

### 7.4 Relief amounts
Earned Income $1,000 / $6,000 (55–59) / $8,000 (60+), capped at earned income · Spouse $2,000,
disability $5,500 · QCR $4,000, Child (Disability) $7,500 · Parent $9,000 living with / $5,500 not,
disability $14,000 / $10,000, max 2 dependants · Grandparent Caregiver $3,000 · Sibling
(Disability) $5,500 · NSman $3,000 active / $5,000 key appointment / $1,500 / $3,500 inactive;
wife $750; parent $750 each · Parenthood Tax Rebate $5,000 / $10,000 / $20,000 (a **rebate**).

**WMCR** — from YA 2025, children born or adopted on/after 1 Jan 2024 get fixed
$8,000 / $10,000 / $12,000 for the 1st / 2nd / 3rd-and-later child. Earlier births keep
15% / 20% / 25% of earned income. QCR + WMCR is capped at $50,000 **per child**, and WMCR cannot
exceed earned income.

### 7.5 Rules that trip people up — keep these in the UI
- **Course Fees Relief is withdrawn from YA 2026.** Do not reintroduce it.
- **FDWL Relief lapsed after YA 2024.**
- **No personal income tax rebate for YA 2026.** There was 50% (capped $200) for YA 2024 and
  60% (capped $200) for YA 2025.
- **MRSS-matched CPF top-ups get no relief from YA 2026** — take the matching grant or the relief.
- **SRS withdrawal age locks to the statutory retirement age when you make your *first*
  contribution** — 63 until 30 Jun 2026, 64 from 1 Jul 2026. A token $1 contribution locks the
  lower age.
- **Early SRS withdrawal: 100% taxable plus a 5% penalty.** At/after the retirement age, only 50%
  is taxable, spread over up to 10 years.
- **CPF SA closed at 55 from 2025**; top-ups go to the Retirement Account.
- No deduction exists for own-home mortgage interest, rent, medical bills, tuition, or a car.

---

## 8. EDGE CASES & ERRORS

| Input | Required behaviour |
|---|---|
| Empty / non-numeric / negative number field | Treated as 0 (`numOf` returns 0 unless finite and > 0). Never `NaN` on screen. |
| Take-home of $0 | Everything resolves to $0. Bisection converges; no divide-by-zero. |
| Chargeable income below $20,000 | Tax $0, marginal 0%, **zero** live suggestions, and an explicit notice that contributions would buy no tax saving. |
| Reliefs summing past $80,000 | Cap applied; wasted amount reported; every further-relief suggestion shows a $0 saving with a "stop contributing" warning. |
| Bonus above the AW ceiling | Only the ceiling attracts CPF; a warning states the ceiling and the amount actually charged. |
| Salary above the OW ceiling | CPF on the first $8,000/month only; notice explains the excess is fully take-home and fully taxable. |
| Contribution above the SRS/CPF cap | Silently clamped for the maths, **loudly** flagged in the hint and a notice. |
| Projection where the scheme loses | Negative edge shown in `--tax`, with a notice explaining that the relief going in is worth less than the tax coming out. |
| Withdrawal age ≤ current age | Clamped to `age + 1`. |
| Very long projections (50 years) | Table scrolls in a `max-height:340px` container; chart x-ticks thin out automatically. |

---

## 9. ACCEPTANCE CHECKLIST

- [ ] **R7** All 12 cumulative tax figures in §7.1 reproduce exactly.
- [ ] **R8** Chargeable income of exactly $280,000 reports a 20% marginal rate; $278,600 reports 19.5%. Ladder highlight matches the chip.
- [ ] **R1/R3** Take-home $4,000, age 30, 2-month bonus, 2026 → gross $5,000/month, employee CPF $14,000/yr, employer CPF $11,900/yr, chargeable $55,000, tax $1,600.
- [ ] **R4** Gross $20,000/month with a 3-month bonus → OW subject $96,000, AW ceiling $6,000, total CPF exactly $37,740.
- [ ] **R6** Foreigner: all CPF zero, SRS cap $35,700, life-insurance room $5,000.
- [ ] **R14/R15** Reliefs forced past $80,000 → `reliefUsed` is $80,000, headroom $0, and the SRS card shows a $0 saving.
- [ ] **R16** Every suggestion card has at least one pro **and** one con.
- [ ] **R20** $10,000/yr for 20 years at 6% → balance equals `10000 × 1.06 × (1.06²⁰ − 1)/0.06` within $1.
- [ ] **R21** With zero return and zero growth, the comparison arm equals `contributions − cumulative tax saved` exactly.
- [ ] **R24** An income below the tax threshold produces a **negative** edge (lock-in for no benefit).
- [ ] **R23** Salary growth that pushes reliefs past the cap flags the affected years.
- [ ] `assess()` output equals `model()` output for chargeable income, tax, relief total and employee CPF.
- [ ] Zero console errors at 1280 px and 420 px, light and dark.
- [ ] `document.documentElement.scrollWidth === clientWidth` at a 360 px viewport.
- [ ] Chart axis text renders at its stated pixel size at 420 px (SVG viewBox width equals container width).
- [ ] Hover anywhere on the plot shows a crosshair and a tooltip with all five figures.

---

## 10. ASSUMPTIONS & OPEN QUESTIONS

**ASSUMED**
1. **YA 2027 uses today's bands with no rebate.** Not announced as at Aug 2026. Labelled
   provisional throughout, with income year 2025 available as the confirmed basis.
2. **The user is a tax resident**, employed in the private sector. No residency test in the UI.
3. **Earned Income Relief is computed from employment income only**, ignoring trade income entered
   under "other taxable income". The relief is small and capped, so the error is at most a few
   hundred dollars.
4. **The projection holds CPF rates, tax bands and the relief cap constant** for its whole horizon.
   Stated in the assumptions panel; certain to be wrong over 30 years, and there is no better
   available assumption.
5. **Relief inputs are entered as the user's own share** where a relief is splittable with a spouse.
   The UI says so; it does not enforce it.
6. **Mixed-regime WMCR families** (some children born before 2024, some after) are not modelled —
   one regime applies to all children in the count.
7. **CPF projections use the same 10-year drawdown as SRS** for comparability, though CPF actually
   converts to lifelong CPF LIFE payouts. Flagged in the assumptions panel.

**OPEN QUESTIONS FOR BEN**
- Should the calculator be linked from the portfolio's nav, and under what label?
- Should it remember inputs between visits (`localStorage`), or stay stateless by design?
- Is a non-resident mode worth adding, or is the audience always resident?

---

### Riskiest assumptions — confirm these first

1. **YA 2027 rates are guesses.** The whole default view rests on the Budget not changing the bands
   or announcing a rebate; if it has, `YEARS[2026]` and the "no rebate" copy both need updating.
2. **Every tax and CPF figure was verified in Aug 2026 and has a shelf life.** The senior-worker CPF
   rates in particular are mid-ramp and change again in 2027 — treat §7.2 as perishable.
3. **The projection's flat-return model understates real risk.** It is arithmetically correct and
   economically naive; keep the "why it could be wrong" panel prominent rather than trimming it.
