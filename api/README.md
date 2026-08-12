# ER Score API — Deployment & Bubble Integration

This is a small, self-contained API that runs the full ER Score calculation
(ER-028 through ER-087 — synced directly from the working prototype and verified
against its previously-confirmed worked examples). It's built to deploy on Vercel
for free and be called from Bubble via the API Connector plugin.

**Keeping this in sync:** this engine is auto-extracted from er-score-prototype.jsx
rather than hand-maintained separately, specifically because the two drifted apart
before (this file previously stopped at ER-057 while the prototype moved on to
ER-087 without anyone noticing). Whenever the prototype's scoring logic changes,
re-extract this file from it rather than editing both independently.

---

## Part 1 — Deploy to Vercel

**You do not need to know how to code to do this.**

1. Create a free account at [vercel.com](https://vercel.com) if you don't have one.
2. Create a free account at [github.com](https://github.com) if you don't have one.
3. Create a new empty repository on GitHub (e.g. `er-score-api`).
4. Upload these three files/folders into that repository, keeping the exact structure:
   ```
   er-score-api/
     api/
       _engine.js
       score.js
     package.json
   ```
   (GitHub's web interface lets you drag-and-drop files directly — no command line needed.)
5. Go to [vercel.com/new](https://vercel.com/new), click "Import Project," and select the GitHub repo you just created.
6. Leave all settings at their defaults and click **Deploy**.
7. Vercel will give you a URL like `https://er-score-api-yourname.vercel.app`.

Your live endpoint is now: **`https://er-score-api-yourname.vercel.app/api/score`**

Test it's working by visiting that URL in a browser — you should see
`{"error":"Use POST"}`, which means it's live and correctly rejecting a
plain browser visit (it only accepts POST requests with data).

---

## Part 2 — Connect Bubble to it

1. In Bubble, go to the **Plugins** tab and install the official **API Connector** plugin (free).
2. Add a new API called `ER Score`.
3. Add a new API call, e.g. `Calculate Score`:
   - Method: `POST`
   - URL: `https://er-score-api-yourname.vercel.app/api/score`
   - Body type: `JSON`
   - In the body, add the fields listed below, using Bubble's `<field_name>` syntax so
     they map to your app's actual input fields (e.g. `<age>`, `<annualIncome>`).
4. Click "Initialize call" with some test values filled in — Bubble will
   auto-detect the response shape (total score, band, category breakdowns, etc.)
   so you can reference them in workflows and page elements afterward.
5. In your Calculator page's workflow (e.g. "when Calculate button is clicked"),
   add a step to trigger the `Calculate Score` API call, passing in whatever
   the user just entered into your form.
6. Display the results by referencing `Result of step X's total`,
   `Result of step X's band's label`, etc. on your Results page.

---

## Request Body Fields

Every field below is a number, unless noted otherwise. Only send the fields
relevant to the categoryId being scored — extras are ignored, but required
ones must be present or the math will be wrong (not an error — it'll just
silently treat missing numbers as 0).

### Always required
| Field | Type | Notes |
|---|---|---|
| `categoryId` | text | One of: `vehicle`, `boat`, `rv`, `motorcycle`, `trailer`, `electronics`, `furniture`, `vacation`, `wedding`, `education`, `primary_residence`, `investment_property`, `renting`, `other` |
| `age` | number | |
| `targetRetirementAge` | number | Defaults to 67 if omitted |
| `annualIncome` | number | |
| `takeHome` | number | Monthly take-home income |
| `livingExpenses` | number | Monthly, before this purchase |
| `savings` | number | |
| `taxableInvestments` | number | |
| `retirementBalance` | number | |
| `delinquent` | true/false | |
| `purchasePrice` | number | Not required for `renting` or `education` |
| `payMethod` | text | `cash`, `finance`, `lease`, `credit_full`, `credit_carry` |

### If financing a standard purchase (`payMethod: "finance"`)
`cashDown`, `hasTradeIn` (true/false), `tradeInEquity`, `termMonths`, `interestRate`, `loanType` (`fixed`/`adjustable`/`balloon`/`interest_only`)

### If `payMethod: "credit_carry"`
`minPayment`

### If `categoryId: "investment_property"`
`estRent`, `rentalExpenses`, `ipMortgage`

### If `categoryId: "primary_residence"` or `"renting"` (Housing Situation, ER-051)
`housingSituation` — one of `renting`, `own_selling`, `own_keeping`, `none`
Then, depending on that value: `currentMonthlyRent`, `currentHousingPayment`,
`saleSalePrice`, `saleMortgageBalance`, `realtorPct`, `sellingClosingPct`,
and (`primary_residence` only) `newPITI`

### If `categoryId: "renting"` (ER-047)
`monthlyRent`, `securityDeposit`, `lastMonthRequired` (true/false)

### If `categoryId: "education"` (ER-052)
`eduLoanAmount`, `eduInterestRate`, `eduTermMonths`, `hasDeferment` (true/false),
`defermentMonths`, `expectedIncomeAfterCompletion`

### If `payMethod: "lease"` (ER-048/049/050)
`negotiatedVehiclePrice`, `capCostReduction`, `monthlyLeasePayment`, `leaseTermMonths`,
`isSinglePayLease` (true/false), `totalLeaseCost`

---

## Response Shape

```json
{
  "total": 82,
  "band": { "label": "Strong Financial Position", "color": "#6FA37A" },
  "overridden": false,
  "overrides": [],
  "cashFlow": { "points": 35, "max": 40, "margin": 2100, "pct": 0.34 },
  "financialCushion": { "points": 30, "max": 35, "savings": 22, "investments": 5, "retirement": 3 },
  "financingQuality": { "points": 17, "max": 25, "breakdown": { "ls": 6, "dp": 4, "lt": 4, "ty": 3 } },
  "recommendations": [ { "label": "...", "detail": "...", "delta": 5 } ],
  "raw": { ... full internal detail, rarely needed ... }
}
```

---

## Updating the Math Later

If you make another scoring decision (ER-058 and beyond), only `_engine.js`
needs to change. Edit it, push the change to your GitHub repo, and Vercel
will automatically redeploy — Bubble doesn't need to be touched, since it's
just calling the same URL.
