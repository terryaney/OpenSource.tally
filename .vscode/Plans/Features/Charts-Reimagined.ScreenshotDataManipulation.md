# Charts Reimagined Screenshot Data Manipulation (Playbook)

Purpose: temporary screenshot-only value tweaks for the generated smoke report, with a single on/off switch and easy cleanup.

Target file:
- C:/BTR/OpenSource/tally/build/charts-reimagined-smoke/spending_report.js

Rules:
- Keep all tweaks inside one `SCREENSHOT_PATCH` config block.
- Gate every manipulation with `if (SCREENSHOT_PATCH.enabled)`.
- Never commit changes to `src/tally/spending_report.js` for fake screenshot data.
- After screenshots, remove all references by searching for `SCREENSHOT_PATCH`.

## 1) Add a Central Switch

Place this near chart constants:

```js
const SCREENSHOT_PATCH = {
    enabled: true,
    recurringNameOverrides: {
        'Josh Aney': 'John Doe',
        'Tiffany Skaran': 'Jane Doe',
        'Conduent': 'Microsoft',
    },
    recurringMoreLabel: '+ 13 more',
    recurringFooterLabel: '23 recurring merchants',
    kpiTargets: {
        incomeBase: 160000,
        spendingBase: 145000,
        randomSpread: 900,
        seed: 'kpi-screenshot-v1',
    },
    kpiTrendPct: {
        income: 10.4,
        spending: 3.2,
        cashFlow: 11.1,
    },
    kpiTrendWindowMonths: 6,
    compareYearJitter: 0.18,
    compareYearMinValue: 40,
};
```

## 2) KPI Headline Overrides

Inside `renderKpis(...)`, override displayed totals only when enabled:

```js
if (SCREENSHOT_PATCH.enabled && SCREENSHOT_PATCH.kpiTargets) {
    const t = SCREENSHOT_PATCH.kpiTargets;
    const spread = Math.max(0, Number(t.randomSpread || 0));
    const seed = String(t.seed || 'kpi-screenshot');

    const rand = (key) => {
        let h = 2166136261;
        const s = `${seed}|${key}`;
        for (let i = 0; i < s.length; i += 1) {
            h ^= s.charCodeAt(i);
            h = Math.imul(h, 16777619);
        }
        return (h >>> 0) % 1000000 / 1000000;
    };

    let targetIncome = Math.round(Number(t.incomeBase || 160000) + ((rand('income') * 2 - 1) * spread));
    let targetSpending = Math.round(Number(t.spendingBase || 145000) + ((rand('spending') * 2 - 1) * spread));

    if (targetIncome % 1000 === 0) targetIncome += 137;
    if (targetSpending % 1000 === 0) targetSpending -= 173;

    const targetCashFlow = targetIncome - targetSpending;
    // Apply to displayed KPI values and sparkline series.
}
```

## 3) KPI Trend Text Overrides

Force realistic percentages in screenshot mode:

```js
function screenshotKpiTrend(metricKey, upIsGood) {
    if (!SCREENSHOT_PATCH.enabled) return null;
    const raw = Number((SCREENSHOT_PATCH.kpiTrendPct || {})[metricKey]);
    if (!Number.isFinite(raw)) return null;

    const isUp = raw >= 0;
    const windowMonths = Math.max(1, Math.round(Number(SCREENSHOT_PATCH.kpiTrendWindowMonths || 6)));
    return {
        arrow: isUp ? '↑' : '↓',
        value: `${isUp ? '+' : '-'}${Math.abs(raw).toFixed(1)}%`,
        sentence: windowMonths === 1
            ? ' this month vs last month'
            : ` this month vs prior ${windowMonths} mo`,
        cls: (isUp === upIsGood) ? 'pos' : 'neg',
    };
}
```

## 4) Recurring Table Labels

Use patch values for cosmetic replacements:

```js
merchant: SCREENSHOT_PATCH.enabled
    ? (SCREENSHOT_PATCH.recurringNameOverrides[row.merchant] || row.merchant)
    : row.merchant
```

```js
td.textContent = SCREENSHOT_PATCH.enabled
    ? SCREENSHOT_PATCH.recurringMoreLabel
    : `+ ${hidden.length} more`;
```

```js
label.textContent = SCREENSHOT_PATCH.enabled
    ? SCREENSHOT_PATCH.recurringFooterLabel
    : `${rows.length} recurring merchants`;
```

## 5) Compare-Years Fill (Optional)

If compare mode looks sparse for screenshots, add synthetic fill only in screenshot mode and keep it deterministic by seed.

## 6) Validate

From repo root:

```powershell
rg -n "SCREENSHOT_PATCH|kpiTargets|kpiTrendPct|recurringNameOverrides" build/charts-reimagined-smoke/spending_report.js
```

Open smoke page and verify KPI cards and recurring table labels.

## 7) Remove After Screenshots

1. Delete `SCREENSHOT_PATCH` block.
2. Delete helper functions used only by the patch.
3. Replace all conditional `SCREENSHOT_PATCH.enabled ? ... : ...` with the real branch behavior.
4. Confirm no refs remain:

```powershell
rg -n "SCREENSHOT_PATCH|kpiTargets|kpiTrendPct|recurringNameOverrides|applyScreenshotCompareFill" build/charts-reimagined-smoke/spending_report.js
```

Expected result: no matches.
