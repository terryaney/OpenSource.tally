Title: Add per-category-per-month breakdown to spending_summary.json

Description:

Currently the JSON output has two separate dimensions that can't be correlated:

 - by_month — overall monthly totals
 - by_category — category totals across the entire date range

This makes it impossible to answer questions like "what categories changed the most from last month?" or "how did my
grocery spending trend over the quarter?" without re-parsing the raw CSV data.

Requested change:

Add a by_month_by_category (or by_category_by_month) section to the JSON output:

 "by_month_by_category": {
   "2026-01": [
     { "category": "Food", "subcategory": "Grocery", "total": 1423.50 },
     { "category": "Health", "subcategory": "Medical", "total": 890.00 }
   ],
   "2026-02": [
     { "category": "Food", "subcategory": "Grocery", "total": 1198.20 },
     { "category": "Health", "subcategory": "Medical", "total": 4200.00 }
   ]
 }

Use case:

Enables AI assistants (and other tooling) to answer month-over-month analysis questions directly from the output file
— trend detection, anomaly spotting, budget tracking — without needing access to the raw statement CSVs.