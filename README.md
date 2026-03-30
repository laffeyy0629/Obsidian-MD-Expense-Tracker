# Obsidian-MD-Expense-Tracker
This is small expense tracker with some automation features for Obsidian. This requires the plugins Dataview and Templater for it to work.

# Expense Tracker — Setup & Usage Guide

> **What this includes:**
> - A live dashboard with daily/weekly/monthly/savings totals
> - Configurable daily & weekly budget limits with visual progress bars
> - Category breakdowns and monthly history
> - A Templater-powered quick-entry template

---

## File Structure

After setup, your vault should look like this:

```
Your Vault/
├── Expense Dashboard.md       ← Main dashboard (open this daily)
├── Templates/
│   └── Expense Template.md    ← Template for new expenses
└── Expenses/
    ├── 2025-01-15 Expense.md     ← Individual expense notes
    ├── 2025-01-15 Expense 2.md
    └── ...
```

---

## Step 1 — Install Required Plugins

You need 2 community plugins.

| Plugin | Purpose |
|--------|---------|
| **Dataview** | Powers all the live tables and calculations |
| **Templater** | Enables the interactive expense entry form |

After installing both, enable them and restart Obsidian.

---

## Step 2 — Configure Dataview

Go to `Settings → Dataview` and turn on:

- **Enable JavaScript Queries** (required for the dashboard charts)
- **Inline Query Prefix** — leave as default (`=`)
- **Automatic View Refreshing** — optional but recommended

---

## Step 3 — Place the Files

Copy the provided files into your vault **exactly** as shown in the folder structure above:

1. `Expense Dashboard.md` → vault root (or anywhere you prefer)
2. `Templates/Expense Template.md` → inside a `Templates/` folder
3. `Expenses/` → create this empty folder (expenses go here)

> **Important:** The dashboard looks for expense notes inside a folder called `Expenses`. If you rename it, update line 3 of each `dataviewjs` block:
> ```js
> const EXPENSE_FOLDER = "Expenses"; // ← change this
> ```

---

## Step 4 — Configure Templater

Go to `Settings → Templater`:

1. Set **Template folder location** to `Templates`
2. Enable **Trigger Templater on new file creation** (optional but convenient)
3. Under **Folder Templates**, you can map `Expenses/` → `Expense Template` so every new note in that folder auto-fills the template

---

## ✅ Step 5 — Set Your Budget Limits

Open `Expense Dashboard.md` and edit the frontmatter at the top:

```yaml
---
daily_limit: 1000      ← Your daily spending limit
weekly_limit: 5000     ← Your weekly spending limit  
currency: "₱"          ← Change to $, €, etc. if needed
---
```

Save the file — the dashboard updates instantly.

---

## How to Log an Expense (Daily Use)

### Method A — Templater Quick Entry (Recommended)

1. Press `Alt+N` (or your Templater hotkey) to create a new note
2. Choose `Expense Template`
3. Save the file inside the `Expenses/` folder
4. Templater will prompt you to:
   - Pick a **category** from a dropdown
   - Pick a **payment method**
   - Type a **note** (optional)
5. In the created note, fill in the `amount:` field in the frontmatter:
   ```yaml
   amount: 250
   ```
6. Save — your dashboard updates immediately!

### Method B — Manual Entry

1. Create a new note inside the `Expenses/` folder
2. Name it anything (e.g. `2025-01-15 Coffee`)
3. Paste this frontmatter at the top:

```yaml
---
date: 2025-01-15
amount: 85
category: Food
payment_method: Cash
note: Coffee at Starbucks
tags: [expense]
---
```

4. Save — it will appear in the dashboard right away.

---

## Dashboard Sections Explained

| Section | What it shows |
|---------|--------------|
| **Summary cards** | Today / This Week / This Month / All Time totals / Total Saved|
| **Budget Limits** | Progress bars for daily and weekly limits (green → yellow → red) |
| **Top Categories** | Your biggest spending categories, all time |
| **Today's Expenses** | All notes with `date = today` |
| **This Week's Expenses** | Last 7 days |
| **This Month's Expenses** | Last 30 days |
| **Monthly Totals** | A running month-by-month history |

---

## Valid Categories

The template includes these defaults. You can use any text value — just be consistent!

| Category | Use for |
|----------|---------|
| `Food` | Meals, groceries, drinks |
| `Transport` | Grab, commute, fuel |
| `Shopping` | Clothes, gadgets, personal items |
| `Health` | Medicine, checkups, gym |
| `Entertainment` | Movies, games, subscriptions |
| `Utilities` | Electricity, water, internet, phone load |
| `Education` | Books, courses, tuition |
| `Housing` | Rent, repairs, supplies |
| `Work` | Work-related purchases |
| `Gifts` | Gifts for others |
| `Travel` | Hotels, flights, tours |
| `Other` | Anything else |

---

## Payment Methods

| Value | Meaning |
|-------|---------|
| `Cash` | Physical cash |
| `GCash` | GCash e-wallet |
| `Maya` | Maya / PayMaya |
| `Bank Transfer` | Direct bank transfer |
| `Credit Card` | Any credit card |
| `Debit Card` | Any debit card |

---

## Customization 

### Change the currency symbol
Edit `currency: "₱"` in the dashboard frontmatter.

### Change the Expenses folder name
Edit `const EXPENSE_FOLDER = "Expenses"` in each `dataviewjs` block.

### Add a new category
Just type a new category name in any expense note's frontmatter — it will auto-appear in the category table.

### Change week start to Monday (instead of Sunday)
In the dashboard's `dataviewjs` block, find:
```js
startOfWeek.setDate(now.getDate() - now.getDay()); // Sunday
```
Change to:
```js
const day = now.getDay();
startOfWeek.setDate(now.getDate() - (day === 0 ? 6 : day - 1)); // Monday
```

### Filter a specific month's expenses
Add a new `dataview` block anywhere:
```dataview
TABLE date, amount, category, note
FROM "Expenses"
WHERE date >= date(2025-01-01) AND date <= date(2025-01-31)
SORT date ASC
```

---

## Troubleshooting

**Dashboard shows "No expenses logged yet"**
→ Make sure your expense notes are inside the `Expenses/` folder and have `date:` and `amount:` in the frontmatter.

**Dates not being recognized**
→ Use `YYYY-MM-DD` format only (e.g. `2025-01-15`). Avoid `January 15, 2025`.

**JavaScript queries not running**
→ Go to `Settings → Dataview → Enable JavaScript Queries` and toggle it on.

**Templater prompts not appearing**
→ Make sure Templater is enabled and the template folder is set correctly in its settings.

**Budget bars not updating**
→ Click away and return to the dashboard, or use `Ctrl+R` to refresh the view.
