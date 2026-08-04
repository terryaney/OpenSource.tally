# Wish List To Get Into Tally

## Personal Decisions / TODOs

1. AI: Search rules for duplicate names or names with diff suffix and give me list.
  - Then I'll evaluate if I want them grouped together or not since issue-88 was fixed and supports diff categorization for same merchant name
2. Review 'When to use Investment and Transfer Categories' chats and update my rules
3. Search for 'Sports' under school - activities? what should I categorize them?
4. Search for `Shopping / Gifts` and make bday and other holidays (see if any gifts around easter)
5. Should FedEx be categorized as 'gifts' or 'fees'?

## Plans To Implement

1. Need documentation (and skill sample) especially for categorization

2. Plans\Info-Popup-All-Rules.md - Design a robust “multi-match popup” architecture that can show multiple rules that hit a transaction instead of only 1.

3. Plans\Fixed-Spending-Cleanup.md - Better explicit control over fixed spending classification and reporting, including a new `fixed-budget` tag for baseline fixed costs using trailing complete-month average math.

## Bugs / Features

1. Would like a way to flag a file as 'done' so it doesn't have to waste processing power on it?

1. Make a 'group by quarters' option as well and first, second, third months across quarters compare.  Obviously needs to do paging when needed.

2. Mobile Issues - I think mobile use is rare given local html file, guess I could send it over text or something, but could fix this:
  - Merchant Listing Grid - Table could be some sort of 'card' row or something instead of a table row with multi line data instead of so much wrapping
  - Change Category Header title stays single line and the money drops to next line (left aligned)
  - Need to figure out what to do with transaction 'listing' under a merchant in mobile view

1. Rules : Ask it to make a plan to have CATEGORY only rules maybe [*] as merchant?  So doesn't make a merchant but merges (similar to TAG only rules)?  That would eliminate all the account=source and tagging duplication when multiple accounts and/or merchants have overrides