---
# ── SETTINGS ─────────────────────────────────────────────
daily_limit: 500
weekly_limit: 2500
currency: "₱"
savings_start: 2026-03-30
savings_goal: 10000
# ─────────────────────────────────────────────────────────
tags: [expense-dashboard]
---

# Expense Dashboard

> [!tip] Quick Add
> To log an expense, create a new note in `Expenses/` using the [[Expense Template]] or run the **Expense Entry** template from Templater.

---

## Summary

```dataviewjs
// ── CONFIG ────────────────────────────────────────────────────────
const EXPENSE_FOLDER = "Expenses";
const cfg = dv.current();
const DAILY_LIMIT  = cfg.daily_limit  ?? 500;
const WEEKLY_LIMIT = cfg.weekly_limit ?? 2500;
const CURRENCY     = cfg.currency     ?? "₱";
const SAVINGS_GOAL = cfg.savings_goal ?? 0;

// Parse savings_start safely
const rawStart = cfg.savings_start;
let SAVINGS_START;
if (rawStart) {
  SAVINGS_START = rawStart.ts ? new Date(rawStart.ts) : new Date(String(rawStart) + "T00:00:00");
} else {
  SAVINGS_START = new Date();
}
SAVINGS_START.setHours(0, 0, 0, 0);

// ── TIMEZONE-SAFE DATE HELPERS ────────────────────────────────────
function localStr(d) {
  return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,"0")}-${String(d.getDate()).padStart(2,"0")}`;
}

const now      = new Date();
const todayStr = localStr(now);

const startOfWeek = new Date(now);
const day = now.getDay();
startOfWeek.setDate(now.getDate() - (day === 0 ? 6 : day - 1)); // Monday
startOfWeek.setHours(0, 0, 0, 0);

const startOfMonth = new Date(now.getFullYear(), now.getMonth(), 1);

function toDate(d) {
  if (!d) return null;
  if (d instanceof Date) return d;
  if (typeof d === "string") return new Date(d + "T00:00:00");
  if (d.ts) return new Date(d.ts);
  return new Date(String(d));
}

// ── LOAD & AGGREGATE ──────────────────────────────────────────────
const pages = dv.pages(`"${EXPENSE_FOLDER}"`).where(p => p.amount && p.date);

let dailyTotal = 0, weeklyTotal = 0, monthlyTotal = 0, allTotal = 0, spentSinceStart = 0;
const categoryMap = {};

for (const p of pages) {
  const amt = parseFloat(p.amount) || 0;
  const d   = toDate(p.date);
  if (!d || isNaN(d)) continue;

  allTotal += amt;
  if (d >= startOfMonth)       monthlyTotal    += amt;
  if (d >= startOfWeek)        weeklyTotal     += amt;
  if (localStr(d) === todayStr) dailyTotal     += amt;
  if (d >= SAVINGS_START)      spentSinceStart += amt;

  const cat = p.category ?? "Uncategorized";
  categoryMap[cat] = (categoryMap[cat] ?? 0) + amt;
}

// ── SAVINGS CALCULATION ───────────────────────────────────────────
const todayMidnight = new Date(now); todayMidnight.setHours(0,0,0,0);
const daysElapsed   = Math.floor((todayMidnight - SAVINGS_START) / 86400000) + 1;
const totalBudgeted = daysElapsed * DAILY_LIMIT;
const totalSaved    = Math.max(0, totalBudgeted - spentSinceStart);
const savingsPct    = SAVINGS_GOAL > 0 ? Math.min(totalSaved / SAVINGS_GOAL * 100, 100).toFixed(1) : null;

// ── HELPERS ───────────────────────────────────────────────────────
const fmt = n => CURRENCY + n.toLocaleString("en-PH", {minimumFractionDigits:2});

function bar(value, limit, label) {
  const pct   = Math.min(value / limit * 100, 100).toFixed(1);
  const over  = value > limit;
  const color = over ? "#ef4444" : pct > 75 ? "#f59e0b" : "#22c55e";
  const remain = limit - value;
  const sign   = remain < 0 ? "OVER by " : "Left: ";
  const f = n => CURRENCY + Math.abs(n).toLocaleString("en-PH", {minimumFractionDigits:2});
  return `
<div style="margin-bottom:18px">
  <div style="display:flex;justify-content:space-between;align-items:baseline;margin-bottom:4px">
    <span style="font-weight:600;font-size:0.95em">${label}</span>
    <span style="font-size:0.82em;color:${color};font-weight:700">${sign}${f(remain)}</span>
  </div>
  <div style="background:#2a2a2a;border-radius:999px;height:10px;overflow:hidden">
    <div style="width:${pct}%;height:100%;background:${color};border-radius:999px;transition:width .4s"></div>
  </div>
  <div style="display:flex;justify-content:space-between;font-size:0.78em;margin-top:3px;opacity:0.7">
    <span>${f(value)} spent</span><span>Limit: ${f(limit)}</span>
  </div>
</div>`;
}

// ── TOP CATEGORIES ────────────────────────────────────────────────
const catRows = Object.entries(categoryMap)
  .sort((a,b) => b[1]-a[1]).slice(0,6)
  .map(([cat, amt]) => {
    const pct = allTotal > 0 ? (amt/allTotal*100).toFixed(1) : 0;
    return `<tr>
      <td style="padding:5px 10px 5px 0">${cat}</td>
      <td style="padding:5px 0;text-align:right;font-weight:600">${fmt(amt)}</td>
      <td style="padding:5px 0 5px 12px;color:#888;font-size:0.85em">${pct}%</td>
    </tr>`;
  }).join("");

// ── SAVINGS GOAL BAR ──────────────────────────────────────────────
const goalBar = savingsPct !== null ? `
<div style="margin-top:16px;padding-top:16px;border-top:1px solid #1a4a2a">
  <div style="display:flex;justify-content:space-between;align-items:baseline;margin-bottom:4px">
    <span style="font-weight:600;font-size:0.9em">🎯 Savings Goal</span>
    <span style="font-size:0.82em;color:#06b6d4;font-weight:700">${savingsPct}%</span>
  </div>
  <div style="background:#1a3a2a;border-radius:999px;height:8px;overflow:hidden">
    <div style="width:${savingsPct}%;height:100%;background:#06b6d4;border-radius:999px;transition:width .4s"></div>
  </div>
  <div style="display:flex;justify-content:space-between;font-size:0.78em;margin-top:3px;opacity:0.6">
    <span>${fmt(totalSaved)} saved</span>
    <span>Goal: ${fmt(SAVINGS_GOAL)}</span>
  </div>
</div>` : "";

const startLabel = SAVINGS_START.toLocaleDateString("en-PH", {month:"short", day:"numeric", year:"numeric"});

// ── RENDER ────────────────────────────────────────────────────────
dv.el("div", `
<div style="display:grid;grid-template-columns:1fr 1fr;gap:12px;margin-bottom:24px">
  <div style="background:#1a1a2e;border:1px solid #2d2d4e;border-radius:12px;padding:16px">
    <div style="font-size:0.75em;text-transform:uppercase;letter-spacing:.1em;opacity:.6;margin-bottom:4px">Today</div>
    <div style="font-size:1.8em;font-weight:700;color:#7c3aed">${fmt(dailyTotal)}</div>
  </div>
  <div style="background:#1a2e1a;border:1px solid #2d4e2d;border-radius:12px;padding:16px">
    <div style="font-size:0.75em;text-transform:uppercase;letter-spacing:.1em;opacity:.6;margin-bottom:4px">This Week</div>
    <div style="font-size:1.8em;font-weight:700;color:#16a34a">${fmt(weeklyTotal)}</div>
  </div>
  <div style="background:#2e1a1a;border:1px solid #4e2d2d;border-radius:12px;padding:16px">
    <div style="font-size:0.75em;text-transform:uppercase;letter-spacing:.1em;opacity:.6;margin-bottom:4px">This Month</div>
    <div style="font-size:1.8em;font-weight:700;color:#dc2626">${fmt(monthlyTotal)}</div>
  </div>
  <div style="background:#1a1a1a;border:1px solid #333;border-radius:12px;padding:16px">
    <div style="font-size:0.75em;text-transform:uppercase;letter-spacing:.1em;opacity:.6;margin-bottom:4px">All Time</div>
    <div style="font-size:1.8em;font-weight:700;color:#d97706">${fmt(allTotal)}</div>
  </div>
</div>

<div style="background:linear-gradient(135deg,#0f2027,#0d2a1a);border:1px solid #1a4a2a;border-radius:12px;padding:18px;margin-bottom:24px">
  <div style="display:flex;justify-content:space-between;align-items:flex-start">
    <div>
      <div style="font-size:0.75em;text-transform:uppercase;letter-spacing:.1em;opacity:.6;margin-bottom:4px">💰 Total Saved</div>
      <div style="font-size:2.4em;font-weight:700;color:#4ade80;line-height:1">${fmt(totalSaved)}</div>
      <div style="font-size:0.78em;opacity:.5;margin-top:6px">unspent budget accumulated</div>
    </div>
    <div style="text-align:right;font-size:0.78em;opacity:.55;line-height:1.9">
      <div>${daysElapsed} day${daysElapsed !== 1 ? "s" : ""} tracked</div>
      <div>Since ${startLabel}</div>
      <div>${fmt(totalBudgeted)} budgeted</div>
      <div>${fmt(spentSinceStart)} spent</div>
    </div>
  </div>
  ${goalBar}
</div>

<div style="background:#111;border:1px solid #2a2a2a;border-radius:12px;padding:18px;margin-bottom:24px">
  <h3 style="margin:0 0 16px;font-size:0.85em;text-transform:uppercase;letter-spacing:.12em;opacity:.5">Budget Limits</h3>
  ${bar(dailyTotal, DAILY_LIMIT, "📅 Daily Budget")}
  ${bar(weeklyTotal, WEEKLY_LIMIT, "📆 Weekly Budget")}
</div>

<div style="background:#111;border:1px solid #2a2a2a;border-radius:12px;padding:18px">
  <h3 style="margin:0 0 14px;font-size:0.85em;text-transform:uppercase;letter-spacing:.12em;opacity:.5">Top Categories (All Time)</h3>
  <table style="width:100%;border-collapse:collapse">${catRows || "<tr><td colspan='3' style='opacity:.4;padding:8px 0'>No expenses logged yet.</td></tr>"}</table>
</div>
`, {});
```

---

## Today's Expenses

```dataview
TABLE 
  amount AS "Amount",
  category AS "Category",
  note AS "Note",
  payment_method AS "Method"
FROM "Expenses"
WHERE date = date(today)
SORT date DESC
```

---

## This Week's Expenses

```dataview
TABLE 
  date AS "Date",
  amount AS "Amount",
  category AS "Category",
  note AS "Note"
FROM "Expenses"
WHERE date >= date(today) - dur(7 days)
SORT date DESC
```

---

## This Month's Expenses

```dataview
TABLE 
  date AS "Date",
  amount AS "Amount",
  category AS "Category",
  note AS "Note",
  payment_method AS "Method"
FROM "Expenses"
WHERE date >= date(today) - dur(30 days)
SORT date DESC
```

---

## Monthly Totals

```dataviewjs
const EXPENSE_FOLDER = "Expenses";
const CURRENCY = dv.current().currency ?? "₱";

const pages = dv.pages(`"${EXPENSE_FOLDER}"`).where(p => p.amount && p.date);

const monthMap = {};
for (const p of pages) {
  const d = p.date;
  if (!d) continue;
  const dt = d.ts ? new Date(d.ts) : new Date(String(d));
  if (isNaN(dt)) continue;
  const key = `${dt.getFullYear()}-${String(dt.getMonth()+1).padStart(2,"0")}`;
  monthMap[key] = (monthMap[key] ?? 0) + (parseFloat(p.amount) || 0);
}

const rows = Object.entries(monthMap)
  .sort((a,b) => b[0].localeCompare(a[0]))
  .slice(0, 12)
  .map(([month, total]) => {
    const [y,m] = month.split("-");
    const label = new Date(+y, +m-1, 1).toLocaleString("en-PH", {month:"long", year:"numeric"});
    const fmt = CURRENCY + total.toLocaleString("en-PH", {minimumFractionDigits:2});
    return `| ${label} | **${fmt}** |`;
  }).join("\n");

dv.paragraph(`| Month | Total |\n|-------|-------|\n${rows || "| — | No data yet |"}`);
```
