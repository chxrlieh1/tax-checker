# Build Plan — UK Tax Overpayment Checker (First Project)

A step-by-step guide to building a tool that checks whether you've over- or under-paid income tax across your jobs in a tax year. Written to be followed in order, with the *why* explained at each stage so you actually learn how it's built rather than just copying code.

**What you're building (the MVP):** a single web page where you type in each of your jobs (gross pay, tax deducted, tax code), and it tells you whether you've likely overpaid or underpaid income tax for the year, with a plain-English reason.

**Important framing:** this is an *estimate and sanity-check*, not official tax advice or an authoritative figure. The real numbers come from HMRC. Build it to give yourself a "this looks off, worth checking" signal — not a number to send anyone. We'll bake that disclaimer into the tool itself.

---

## Phase 0 — Understand the tax logic before writing any code

You can't build a calculator for something you can't calculate by hand. Spend time here; it's the most important phase.

### The one idea everything hangs on
UK income tax is charged on your **total income across the whole tax year**, using **one** tax-free personal allowance. But each employer runs PAYE on **only the slice they pay you**, applying a tax code in isolation. Neither employer can see the others. The gap between *"correct tax on your combined income"* and *"what all your employers actually deducted between them"* is where a refund (or a bill) comes from.

That gap is precisely why your situation — placement + summer internship + zero-hours job — is a textbook overpayment case.

### The tax year
Runs **6 April to 5 April**. A "check" is cleanest done for a *completed* tax year, because the year has to be finished before you can total everything up. The just-ended year (2025/26, ended 5 April 2026) is the one you'd reconcile now.

### The income tax rules (England, Wales & NI — Scotland differs)
These figures are for **2025/26**, and income tax bands are **unchanged for 2026/27** (they've been frozen since 2021/22, currently until 2030/31):

| Band | Rate | Applies to |
|---|---|---|
| Personal allowance | 0% | first £12,570 |
| Basic rate | 20% | £12,570 – £50,270 (next £37,700) |
| Higher rate | 40% | £50,270 – £125,140 |
| Additional rate | 45% | above £125,140 |

(The allowance tapers away above £100,000, but as a student you can ignore that for the MVP — note it as a "doesn't handle" limitation.)

### Why multiple jobs cause overpayment — the three culprits
Your tool's plain-English explanations will be built around spotting these:

1. **The "BR" (Basic Rate) code on a second job.** HMRC usually puts your *full* personal allowance against one job (code `1257L`) and codes the others `BR` — meaning **20% from the very first pound**, no allowance. That's correct *if* your main job already uses up the whole allowance. But if your total income for the year is **below £12,570**, the BR job taxed you on money that should have been tax-free → refund due.

2. **Emergency / non-cumulative codes** (e.g. `1257 W1` or `M1`) when you start a new job. These tax each pay period in isolation as if it's 1/12 of the year, which over-deducts if you didn't actually work the whole year.

3. **Part-year working** (the internship one). PAYE assumes whatever you earn *this month* continues all 12 months and deducts as if you're on that annualised salary. When the internship ends after 3 months, you've often been taxed as a much higher earner than you really were.

### What this tool deliberately does NOT do (for now)
- **National Insurance** — NI is calculated *per pay period* and is **not** reconciled annually the way income tax is, so you generally can't reclaim NI for a part-year. Leave NI out of the MVP; it'd give a misleading "you're owed NI" result.
- **Student loan** — also per-period, and refundable but through the Student Loans Company separately, not the tax refund. Stretch goal only.
- **Scotland**, the £100k allowance taper, dividends/savings/self-employment. All "later" / "doesn't handle" notes.

### Do this before moving on
Work the example below **by hand on paper**. If you can reproduce these numbers without the tool, you understand the logic.

---

## Phase 1 — Worked example (your reference test case)

Use this as the case you'll test your code against. It mirrors a real student year.

**Jobs in the tax year:**
- Summer internship: **£7,500** gross over 3 months, taxed on an emergency code → about **£870** income tax deducted.
- Zero-hours job back home: **£1,800** gross for the year, on a `BR` code → **£360** deducted (20% of everything).
- *(Placement assumed in a different tax year for this example, to keep it clean.)*

**Step 1 — total income:** £7,500 + £1,800 = **£9,300**

**Step 2 — correct tax on £9,300:** £9,300 is **below** the £12,570 personal allowance, so the correct income tax for the year is **£0**.

**Step 3 — total actually deducted:** £870 + £360 = **£1,230**

**Step 4 — the gap:** £0 due − £1,230 paid = **£1,230 overpaid → likely refund.**

That's the whole engine, conceptually. Everything else is presentation and edge cases.

---

## Phase 2 — Decide scope and sketch it on paper

Before code, draw two things on paper:

1. **The screen.** A title, a short disclaimer, an "Add job" button, a row per job with three boxes (gross pay, tax deducted, tax code), and a result panel at the bottom. Keep it boring and clear.

2. **The data.** Your jobs are just a list. Each job is an object:
   ```
   { grossPay: 7500, taxDeducted: 870, taxCode: "1257 W1" }
   ```
   The whole app state is an **array of these**. Get comfortable with that idea — most of the code is "turn the form into this array, run a calculation on it, show the answer."

---

## Phase 3 — Set up the project

For a first project, keep the toolchain to almost nothing:

- **One file:** `index.html` containing the HTML, a `<style>` block, and a `<script>` block. No build step, no frameworks, no installs. You open it by double-clicking it into a browser.
- **Editor:** VS Code is the standard free choice. Install the "Live Server" extension so the page auto-refreshes when you save — that's the only nicety worth bothering with.
- **The browser console is your best friend.** Press F12 → Console. You'll test your maths there before any UI exists.

Create the skeleton:
```html
<!DOCTYPE html>
<html lang="en">
<head><meta charset="utf-8"><title>Tax Checker</title></head>
<body>
  <h1>Tax Overpayment Checker</h1>
  <script>
    // everything goes here for now
  </script>
</body>
</html>
```

---

## Phase 4 — Build the calculation engine FIRST (no UI yet)

This is the heart of it, and doing it before the interface is a genuinely good habit: you get the logic provably correct in isolation, then wrap a UI around something you trust.

### Step 4a — Put the tax rules in one place
A "config object" so that when rules change next year, you edit one spot:
```javascript
const TAX_YEAR = {
  personalAllowance: 12570,
  bands: [
    { upTo: 50270,   rate: 0.20 }, // basic
    { upTo: 125140,  rate: 0.40 }, // higher
    { upTo: Infinity, rate: 0.45 } // additional
  ]
};
```

### Step 4b — Write the "correct tax" function
Given a total income, return the income tax that *should* be due. Walk the bands, taxing only the slice that falls in each:
```javascript
function correctTax(totalIncome) {
  let taxable = Math.max(0, totalIncome - TAX_YEAR.personalAllowance);
  let tax = 0;
  let lowerBound = TAX_YEAR.personalAllowance;
  for (const band of TAX_YEAR.bands) {
    if (taxable <= 0) break;
    const bandWidth = band.upTo - lowerBound;
    const slice = Math.min(taxable, bandWidth);
    tax += slice * band.rate;
    taxable -= slice;
    lowerBound = band.upTo;
  }
  return tax;
}
```

### Step 4c — Test it in the console *before going further*
Type these into the browser console (F12) and check the answers:
- `correctTax(9300)` → should be `0`
- `correctTax(20000)` → `(20000 − 12570) × 0.20 = 1486`
- `correctTax(60000)` → `37700 × 0.20 + (60000 − 50270) × 0.40 = 7540 + 3892 = 11432`

If those three pass, your engine is sound. **Don't skip this** — debugging maths is ten times harder once it's tangled up with form fields.

### Step 4d — The comparison
```javascript
function checkTax(jobs) {
  const totalIncome  = jobs.reduce((sum, j) => sum + j.grossPay, 0);
  const totalDeducted = jobs.reduce((sum, j) => sum + j.taxDeducted, 0);
  const due = correctTax(totalIncome);
  const difference = totalDeducted - due; // positive = overpaid
  return { totalIncome, totalDeducted, due, difference };
}
```
Test it against your Phase 1 worked example. You should get `difference: 1230`.

---

## Phase 5 — Build the UI and wire it up

Now the part that turns the form into that `jobs` array and shows the result.

1. **Render job rows.** Keep your jobs in a JavaScript array. Write one function that loops the array and draws a row (three `<input>`s + a remove button) for each. "Add job" pushes a blank job and re-draws.
2. **Read inputs.** On a "Calculate" click, read each row's three boxes into the array (remember inputs are *text* — wrap numbers in `Number(...)` or `parseFloat(...)`).
3. **Run `checkTax(jobs)`** and write the result into a result panel.
4. **Don't use a `<form>` with submit** — just buttons with `onclick` handlers. Simpler and avoids page reloads.

Keep styling minimal at this stage. Working first, pretty later.

---

## Phase 6 — Make the output actually useful (the explanations)

A bare number ("you overpaid £1,230") is less helpful than a *reason*. Add simple checks that turn the tax codes and totals into plain English:

- If `totalIncome < personalAllowance` **and** any job's code isn't `1257L` → *"Your total income is below the tax-free allowance, but at least one job deducted tax anyway (a BR or emergency code). That tax is likely reclaimable."*
- If any tax code contains `W1`, `M1`, or `X` → *"An emergency tax code was used, which often over-deducts early in a job."*
- If any code is exactly `BR` → *"A 'BR' code taxes every pound at 20% with no allowance — correct only if another job already uses your full allowance."*
- If `difference` is negative → *"You may have underpaid — worth checking before HMRC does."*

This is where your tool earns its keep, and it's just `if` statements on data you already have.

---

## Phase 7 — Validate, caveat, and ship

- **Input validation:** ignore blank rows, handle non-numbers, don't let it crash on empty input.
- **Always-visible disclaimer** in the UI: *"Estimate only — not tax advice. Income tax only (no NI/student loan). England/Wales/NI rules. Always check against HMRC."*
- **Point to the real thing:** HMRC's own "check if you've paid the right tax" / P800 process and personal tax account are the authoritative source. Your tool is the prompt to go look.

---

## Stretch goals (version 2+, once the core is solid)

Tackle these one at a time, only after the MVP works end-to-end:

1. **Save/load** — let the page remember your jobs (you'll learn about in-memory state vs. persistence here).
2. **National Insurance & student loan** as clearly-labelled *separate* estimates (with the caveat that they reconcile differently).
3. **Multiple tax years** — turn `TAX_YEAR` into a lookup by year so you can check past years too.
4. **Scotland** — a different set of bands behind a toggle.
5. **PDF payslip upload** — the genuinely hard one. Save it for last: the maths must be trustworthy before you automate the data entry, and payslip formats vary so much you'll likely have the user confirm the extracted figures anyway.

---

## Suggested order of work (checklist)

- [ ] Phase 0: read and understand the tax logic
- [ ] Phase 1: reproduce the worked example by hand
- [ ] Phase 3: create `index.html` skeleton
- [ ] Phase 4: write + console-test `correctTax()` and `checkTax()`
- [ ] Phase 5: build the form and wire it to the engine
- [ ] Phase 6: add plain-English explanations
- [ ] Phase 7: validation, disclaimer, tidy styling
- [ ] Stretch goals, one at a time

---

*Figures as of the 2025/26 UK tax year (income tax bands unchanged for 2026/27). England, Wales & Northern Ireland. Verify current rates before relying on the output.*
