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
const cfg            = dv.current();
const DAILY_LIMIT    = cfg.daily_limit  ?? 500;
const WEEKLY_LIMIT   = cfg.weekly_limit ?? 2500;
const CURRENCY       = cfg.currency     ?? "₱";
const SAVINGS_GOAL   = cfg.savings_goal ?? 0;

const rawStart = cfg.savings_start;
let SAVINGS_START = rawStart
  ? (rawStart.ts ? new Date(rawStart.ts) : new Date(String(rawStart) + "T00:00:00"))
  : new Date();
SAVINGS_START.setHours(0,0,0,0);

// ── DATE HELPERS ──────────────────────────────────────────────────
function localStr(d) {
  return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,"0")}-${String(d.getDate()).padStart(2,"0")}`;
}
function toDate(d) {
  if (!d) return null;
  if (d instanceof Date) return d;
  if (typeof d === "string") return new Date(d + "T00:00:00");
  if (d.ts) return new Date(d.ts);
  return new Date(String(d));
}

const now           = new Date();
const todayStr      = localStr(now);
const todayMidnight = new Date(now); todayMidnight.setHours(0,0,0,0);

const startOfWeek = new Date(now);
const dow = now.getDay();
startOfWeek.setDate(now.getDate() - (dow === 0 ? 6 : dow - 1));
startOfWeek.setHours(0,0,0,0);

const startOfMonth = new Date(now.getFullYear(), now.getMonth(), 1);

// ── LOAD PAGES ────────────────────────────────────────────────────
const pages = dv.pages(`"${EXPENSE_FOLDER}"`).where(p => p.amount && p.date);

// ── AGGREGATORS ───────────────────────────────────────────────────
let dailyTotal = 0, weeklyTotal = 0, monthlyTotal = 0, allTotal = 0, spentSinceStart = 0;
const categoryMap = {}, paymentMap = {}, dailyMap = {};
let txCount = 0, maxSingleExpense = 0, maxSingleLabel = "";

for (const p of pages) {
  const amt  = parseFloat(p.amount) || 0;
  const d    = toDate(p.date);
  if (!d || isNaN(d)) continue;
  const dStr = localStr(d);

  txCount++;
  allTotal += amt;
  if (amt > maxSingleExpense) { maxSingleExpense = amt; maxSingleLabel = p.note ?? p.file.name; }
  if (d >= startOfMonth)        monthlyTotal    += amt;
  if (d >= startOfWeek)         weeklyTotal     += amt;
  if (dStr === todayStr)        dailyTotal      += amt;
  if (d >= SAVINGS_START)       spentSinceStart += amt;

  categoryMap[p.category ?? "Uncategorized"] = (categoryMap[p.category ?? "Uncategorized"] ?? 0) + amt;
  paymentMap[p.payment_method ?? "Unknown"]  = (paymentMap[p.payment_method ?? "Unknown"]  ?? 0) + amt;
  dailyMap[dStr] = (dailyMap[dStr] ?? 0) + amt;
}

// ── SAVINGS ───────────────────────────────────────────────────────
const rawDaysElapsed = Math.floor((todayMidnight - SAVINGS_START) / 86400000);
const daysElapsed    = rawDaysElapsed >= 0 ? rawDaysElapsed + 1 : 0;
const totalBudgeted  = daysElapsed * DAILY_LIMIT;
const totalSaved     = Math.max(0, totalBudgeted - spentSinceStart);
const savingsPct     = SAVINGS_GOAL > 0 ? Math.min(totalSaved / SAVINGS_GOAL * 100, 100).toFixed(1) : null;

// Daily / weekly / monthly saved = limit - spent (floored at 0)
const savedToday   = Math.max(0, DAILY_LIMIT  - dailyTotal);
const savedWeek    = Math.max(0, WEEKLY_LIMIT - weeklyTotal);
// Monthly saved: days in month so far × daily limit − monthly spend
const dayOfMonth   = now.getDate();
const savedMonth   = Math.max(0, dayOfMonth * DAILY_LIMIT - monthlyTotal);

// Remaining budget (can go negative = over budget)
const remainDay    = DAILY_LIMIT  - dailyTotal;
const remainWeek   = WEEKLY_LIMIT - weeklyTotal;

// ── STATS ─────────────────────────────────────────────────────────
const avgDaily     = txCount > 0 && daysElapsed > 0 ? spentSinceStart / daysElapsed : 0;
const avgTx        = txCount > 0 ? allTotal / txCount : 0;
const spendDays    = Object.keys(dailyMap).length;
const busiestDay   = Object.entries(dailyMap).sort((a,b)=>b[1]-a[1])[0];
const topCategory  = Object.entries(categoryMap).sort((a,b)=>b[1]-a[1])[0];
const topPayment   = Object.entries(paymentMap).sort((a,b)=>b[1]-a[1])[0];

// Projected monthly spend based on daily average
const daysInMonth  = new Date(now.getFullYear(), now.getMonth()+1, 0).getDate();
const projMonth    = avgDaily * daysInMonth;

// ── RENDER HELPERS ────────────────────────────────────────────────
const fmt  = n  => CURRENCY + Math.abs(n).toLocaleString("en-PH", {minimumFractionDigits:2});
function esc(s) {
  return String(s ?? "").replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;").replace(/"/g,"&quot;");
}

function card(bg, border, labelColor, label, value, sub = "") {
  return `<div style="background:${bg};border:1px solid ${border};border-radius:12px;padding:16px">
    <div style="font-size:0.72em;text-transform:uppercase;letter-spacing:.1em;opacity:.6;margin-bottom:4px">${label}</div>
    <div style="font-size:1.7em;font-weight:700;color:${labelColor};line-height:1.1">${value}</div>
    ${sub ? `<div style="font-size:0.74em;opacity:.5;margin-top:5px">${sub}</div>` : ""}
  </div>`;
}

function bar(value, limit, label) {
  if (limit <= 0) {
    const f = n => CURRENCY + Math.abs(n).toLocaleString("en-PH", {minimumFractionDigits:2});
    return `
<div style="margin-bottom:20px">
  <div style="display:flex;justify-content:space-between;align-items:baseline;margin-bottom:5px">
    <span style="font-weight:600;font-size:0.95em">${esc(label)}</span>
    <span style="font-size:0.82em;color:#888;font-weight:700">No limit set</span>
  </div>
  <div style="background:#2a2a2a;border-radius:999px;height:10px;overflow:hidden">
    <div style="width:0%;height:100%;background:#888;border-radius:999px"></div>
  </div>
  <div style="display:flex;justify-content:space-between;font-size:0.76em;margin-top:4px;opacity:0.6">
    <span>${f(value)} spent</span><span>No limit</span>
  </div>
</div>`;
  }
  const pct   = Math.min(Math.max(value / limit * 100, 0), 100).toFixed(1);
  const over  = value > limit;
  const color = over ? "#ef4444" : pct > 75 ? "#f59e0b" : "#22c55e";
  const remain = limit - value;
  const sign   = remain < 0 ? "⚠️ OVER by " : "Left: ";
  const f = n => CURRENCY + Math.abs(n).toLocaleString("en-PH", {minimumFractionDigits:2});
  return `
<div style="margin-bottom:20px">
  <div style="display:flex;justify-content:space-between;align-items:baseline;margin-bottom:5px">
    <span style="font-weight:600;font-size:0.95em">${esc(label)}</span>
    <span style="font-size:0.82em;color:${color};font-weight:700">${sign}${f(remain)}</span>
  </div>
  <div style="background:#2a2a2a;border-radius:999px;height:10px;overflow:hidden">
    <div style="width:${pct}%;height:100%;background:${color};border-radius:999px;transition:width .4s"></div>
  </div>
  <div style="display:flex;justify-content:space-between;font-size:0.76em;margin-top:4px;opacity:0.6">
    <span>${f(value)} spent</span><span>Limit: ${f(limit)}</span>
  </div>
</div>`;
}

function savingsBar(value, limit, label, color) {
  const pct = limit > 0 ? Math.min(value / limit * 100, 100).toFixed(1) : 100;
  return `
<div style="margin-bottom:16px">
  <div style="display:flex;justify-content:space-between;align-items:baseline;margin-bottom:4px">
    <span style="font-weight:600;font-size:0.88em">${label}</span>
    <span style="font-size:0.82em;color:${color};font-weight:700">${fmt(value)}</span>
  </div>
  <div style="background:#1a3a2a;border-radius:999px;height:7px;overflow:hidden">
    <div style="width:${pct}%;height:100%;background:${color};border-radius:999px"></div>
  </div>
</div>`;
}

// Top categories rows
const catRows = Object.entries(categoryMap)
  .sort((a,b)=>b[1]-a[1]).slice(0,6)
  .map(([cat, amt]) => {
    const pct = allTotal > 0 ? (amt/allTotal*100).toFixed(1) : 0;
    const barW = pct;
    return `<tr>
      <td style="padding:6px 10px 6px 0;white-space:nowrap">${esc(cat)}</td>
      <td style="padding:6px 0;min-width:120px">
        <div style="background:#222;border-radius:4px;height:6px;overflow:hidden">
          <div style="width:${barW}%;height:100%;background:#7c3aed;border-radius:4px"></div>
        </div>
      </td>
      <td style="padding:6px 0 6px 10px;text-align:right;font-weight:600;white-space:nowrap">${fmt(amt)}</td>
      <td style="padding:6px 0 6px 8px;color:#888;font-size:0.82em;white-space:nowrap">${pct}%</td>
    </tr>`;
  }).join("");

// Payment rows
const payRows = Object.entries(paymentMap)
  .sort((a,b)=>b[1]-a[1])
  .map(([method, amt]) => {
    const pct = allTotal > 0 ? (amt/allTotal*100).toFixed(1) : 0;
    return `<tr>
      <td style="padding:5px 10px 5px 0">${esc(method)}</td>
      <td style="padding:5px 0;text-align:right;font-weight:600">${fmt(amt)}</td>
      <td style="padding:5px 0 5px 8px;color:#888;font-size:0.82em">${pct}%</td>
    </tr>`;
  }).join("");

const startLabel = SAVINGS_START.toLocaleDateString("en-PH", {month:"short", day:"numeric", year:"numeric"});

const goalBar = savingsPct !== null ? `
<div style="margin-top:16px;padding-top:16px;border-top:1px solid #1a4a2a">
  <div style="display:flex;justify-content:space-between;align-items:baseline;margin-bottom:4px">
    <span style="font-weight:600;font-size:0.88em">🎯 Savings Goal Progress</span>
    <span style="font-size:0.82em;color:#06b6d4;font-weight:700">${savingsPct}%</span>
  </div>
  <div style="background:#1a3a2a;border-radius:999px;height:8px;overflow:hidden">
    <div style="width:${savingsPct}%;height:100%;background:#06b6d4;border-radius:999px"></div>
  </div>
  <div style="display:flex;justify-content:space-between;font-size:0.76em;margin-top:4px;opacity:0.6">
    <span>${fmt(totalSaved)} saved</span><span>Goal: ${fmt(SAVINGS_GOAL)}</span>
  </div>
</div>` : "";

// ── RENDER ────────────────────────────────────────────────────────
dv.el("div", `

<!-- ROW 1: Spending totals -->
<div style="display:grid;grid-template-columns:1fr 1fr;gap:12px;margin-bottom:12px">
  ${card("#1a1a2e","#2d2d4e","#7c3aed","📅 Today",fmt(dailyTotal))}
  ${card("#1a2e1a","#2d4e2d","#16a34a","📆 This Week",fmt(weeklyTotal))}
  ${card("#2e1a1a","#4e2d2d","#dc2626","🗓️ This Month",fmt(monthlyTotal))}
  ${card("#1a1a1a","#333","#d97706","🗃️ All Time",fmt(allTotal),`${txCount} transactions`)}
</div>

<!-- ROW 2: Remaining budget -->
<div style="display:grid;grid-template-columns:1fr 1fr;gap:12px;margin-bottom:24px">
  ${card(
    remainDay >= 0 ? "#0f1f0f" : "#2a0f0f",
    remainDay >= 0 ? "#1a4a1a" : "#4a1a1a",
    remainDay >= 0 ? "#4ade80" : "#ef4444",
    "💵 Remaining Today",
    (remainDay < 0 ? "-" : "") + fmt(remainDay),
    remainDay < 0 ? "Over daily limit" : `of ${fmt(DAILY_LIMIT)} limit`
  )}
  ${card(
    remainWeek >= 0 ? "#0f1a1f" : "#2a0f0f",
    remainWeek >= 0 ? "#1a3a4a" : "#4a1a1a",
    remainWeek >= 0 ? "#38bdf8" : "#ef4444",
    "📊 Remaining This Week",
    (remainWeek < 0 ? "-" : "") + fmt(remainWeek),
    remainWeek < 0 ? "Over weekly limit" : `of ${fmt(WEEKLY_LIMIT)} limit`
  )}
</div>

<!-- BUDGET PROGRESS BARS -->
<div style="background:#111;border:1px solid #2a2a2a;border-radius:12px;padding:18px;margin-bottom:24px">
  <h3 style="margin:0 0 18px;font-size:0.82em;text-transform:uppercase;letter-spacing:.12em;opacity:.5">⚖️ Budget Usage</h3>
  ${bar(dailyTotal,  DAILY_LIMIT,  "📅 Daily Budget")}
  ${bar(weeklyTotal, WEEKLY_LIMIT, "📆 Weekly Budget")}
</div>

<!-- SAVINGS CARD -->
<div style="background:linear-gradient(135deg,#0f2027,#0d2a1a);border:1px solid #1a4a2a;border-radius:12px;padding:18px;margin-bottom:24px">
  <div style="display:flex;justify-content:space-between;align-items:flex-start;margin-bottom:16px">
    <div>
      <div style="font-size:0.72em;text-transform:uppercase;letter-spacing:.1em;opacity:.6;margin-bottom:4px">💰 Total Saved</div>
      <div style="font-size:2.4em;font-weight:700;color:#4ade80;line-height:1">${fmt(totalSaved)}</div>
      <div style="font-size:0.74em;opacity:.45;margin-top:5px">unspent budget since ${startLabel}</div>
    </div>
    <div style="text-align:right;font-size:0.76em;opacity:.55;line-height:2">
      <div>${daysElapsed} day${daysElapsed !== 1?"s":""} tracked</div>
      <div>${fmt(totalBudgeted)} budgeted</div>
      <div>${fmt(spentSinceStart)} spent</div>
    </div>
  </div>
  <div style="border-top:1px solid #1a4a2a;padding-top:14px">
    <div style="font-size:0.72em;text-transform:uppercase;letter-spacing:.1em;opacity:.45;margin-bottom:10px">Saved by period</div>
    ${savingsBar(savedToday, DAILY_LIMIT,   "📅 Saved Today",      "#4ade80")}
    ${savingsBar(savedWeek,  WEEKLY_LIMIT,  "📆 Saved This Week",  "#34d399")}
    ${savingsBar(savedMonth, dayOfMonth * DAILY_LIMIT, "🗓️ Saved This Month", "#6ee7b7")}
  </div>
  ${goalBar}
</div>

<!-- STATISTICS -->
<div style="display:grid;grid-template-columns:1fr 1fr;gap:12px;margin-bottom:24px">
  ${card("#1a1a2e","#2d2d4e","#a78bfa","📈 Avg Daily Spend",fmt(avgDaily),"since tracking started")}
  ${card("#1a1a2e","#2d2d4e","#a78bfa","🧾 Avg per Transaction",fmt(avgTx),`across ${txCount} transactions`)}
  ${card("#1f1a2e","#3d2d4e","#c084fc","📆 Projected Month",fmt(projMonth),`based on ${fmt(avgDaily)}/day avg`)}
  ${card("#1a1f1a","#2d3d2d","#86efac","📅 Active Spend Days",spendDays + " days","days with at least 1 expense")}
</div>

<!-- TOP CATEGORIES -->
<div style="background:#111;border:1px solid #2a2a2a;border-radius:12px;padding:18px;margin-bottom:24px">
  <h3 style="margin:0 0 14px;font-size:0.82em;text-transform:uppercase;letter-spacing:.12em;opacity:.5">🏷️ Spending by Category</h3>
  <table style="width:100%;border-collapse:collapse">
    ${catRows || "<tr><td colspan='4' style='opacity:.4;padding:8px 0'>No expenses logged yet.</td></tr>"}
  </table>
</div>

<!-- PAYMENT METHODS + HIGHLIGHTS -->
<div style="display:grid;grid-template-columns:1fr 1fr;gap:12px;margin-bottom:8px">
  <div style="background:#111;border:1px solid #2a2a2a;border-radius:12px;padding:18px">
    <h3 style="margin:0 0 12px;font-size:0.82em;text-transform:uppercase;letter-spacing:.12em;opacity:.5">💳 Payment Methods</h3>
    <table style="width:100%;border-collapse:collapse">
      ${payRows || "<tr><td colspan='3' style='opacity:.4'>No data</td></tr>"}
    </table>
  </div>
  <div style="background:#111;border:1px solid #2a2a2a;border-radius:12px;padding:18px">
    <h3 style="margin:0 0 12px;font-size:0.82em;text-transform:uppercase;letter-spacing:.12em;opacity:.5">⚡ Highlights</h3>
    <div style="font-size:0.84em;line-height:2.2;opacity:.85">
      <div>🔥 Biggest single: <strong>${fmt(maxSingleExpense)}</strong></div>
      <div style="opacity:.6;font-size:0.88em;margin-top:-6px;margin-bottom:4px;padding-left:1.2em">${esc(maxSingleLabel)}</div>
      <div>📂 Top category: <strong>${topCategory ? esc(topCategory[0]) : "—"}</strong></div>
      <div>💳 Top payment: <strong>${topPayment ? esc(topPayment[0]) : "—"}</strong></div>
      <div>📅 Busiest day: <strong>${busiestDay ? esc(busiestDay[0]) : "—"}</strong></div>
      <div style="opacity:.6;font-size:0.88em;margin-top:-4px;padding-left:1.2em">${busiestDay ? fmt(busiestDay[1]) + " spent" : ""}</div>
    </div>
  </div>
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
SORT file.ctime ASC
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
