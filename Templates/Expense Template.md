<%*
const title = await tp.system.prompt("Expense title (e.g. Lunch, Grab, Coffee)");
const date  = tp.date.now("YYYY-MM-DD");
await tp.file.move(`Expenses/${date} ${title}`);
-%>
---
date: <% tp.date.now("YYYY-MM-DD") %>
amount: <% await tp.system.prompt("Amount spent (numbers only, e.g. 150)") %>
category: <% await tp.system.suggester(["🍔 Food", "🚌 Transport", "🛍️ Shopping", "💊 Health", "🎮 Entertainment", "📦 Utilities", "📚 Education", "🏠 Housing", "💼 Work", "🎁 Gifts", "✈️ Travel", "❓ Other"], ["Food", "Transport", "Shopping", "Health", "Entertainment", "Utilities", "Education", "Housing", "Work", "Gifts", "Travel", "Other"]) %>
payment_method: <% await tp.system.suggester(["💵 Cash", "💳 GCash", "💳 Maya", "🏦 Bank Transfer", "💳 Credit Card", "💳 Debit Card"], ["Cash", "GCash", "Maya", "Bank Transfer", "Credit Card", "Debit Card"]) %>
note: <% await tp.system.prompt("Note (optional)", "", false) %>
tags: [expense]
---

# <% tp.date.now("YYYY-MM-DD") %> — <% title %>

| Field | Value |
|-------|-------|
| **Date** | `= this.date` |
| **Amount** | `= this.amount` |
| **Category** | `= this.category` |
| **Payment** | `= this.payment_method` |
| **Note** | `= this.note` |

> [!info]
> Back to [[Expense Dashboard]]
