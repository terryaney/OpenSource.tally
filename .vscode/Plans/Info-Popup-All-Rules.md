## Plan: Info Popup for all Rules

You are planning only. Do not implement code.

Repository context:

- Core popup files: src/tally/spending_report.js, src/tally/spending_report.css, src/tally/report.py, src/tally/merchant_utils.py.
- Current popup is single-match oriented with fields like ruleName, pattern, assignedCategory, assignedTags.
- We have UI polish already for popup alignment, loader shell, and tag chips.
- We recently changed popup tags to use merchant union tags (same source as row badges) and dynamic tagColor styling.
- We also promoted fallback match_info so when no categorization rule exists but tag-only rules match, fallback can carry a first tag-rule pattern/rule_name.

Goal:

Design a robust “multi-match popup” architecture compatible with issue/88 + PR #91 behavior (same merchant name with different category/subcategory/tags by pattern).

Need from you:

1. Data model plan:

- How report payload should represent multiple match entries.
- Backward compatibility with current single-match fields.
- Explicit semantics for:
- primary categorization match
- tag-only matches
- merchant-union tags
- provenance per tag.

1. UI plan:

- Popup title/sections for multi-entry list.
- Repeated entry block structure (pattern, explanation, assignment, tags).
- Divider behavior, max-height, internal scrolling, sticky header option.
- Empty/null field handling rules.

1. Migration plan:

- Risk assessment and regression hotspots.

1. Test plan:

- Manual click-test matrix using merchants like Target, Salomon, Josh Aney.
- Unit/integration ideas for payload shaping and rendering.
- Edge cases:
- no categorization match + tag-only matches
- same merchant, multiple category rules
- multiple tag sources for same tag
- missing explanation/pattern in fallback states.
Output format:

- Concise but specific implementation plan with phases, acceptance criteria, and rollback strategy.
- Include example JSON payloads for “single match” and “multi match” cases.
- Include clear “Do not break” compatibility constraints for existing report behavior.
