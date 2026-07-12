# PR Response Doc — CineLog Watchlist Feature
This file is my written record of the code review. It documents what I changed, why, and my reasoning for the two design decisions.

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
- **Detail**: `save_to_watchlist()` in `services/watchlist_service.py` should follow the project's naming convention. Compare with `add_to_collection()` - the pattern here is verb_to_noun -> rename to `add_to_watchlist()` and update all call sites.
- **What I did:** right click on the `save_to_watchlist` function name and selected "Rename symbol" to rename it to `add_to_watchlist`. This will rename the function and update all call sites automatically.
- **How I verified:** I did a project-wide search for `save_to_watchlist` to ensure there's no usage of the old function's name that hasn't been updated. I also compare the changes with VSCode Git tool to see the differences I made before committing the change. This help me verify no out-of-scope issue was touched

## Comment 2 — Deduplication
- **Detail**: `add_to_watchlist()` in `services/watchlist_service.py` doesn't check if a film already existed in watchlist. If a user calls this with a film that's already on their watchlist, the current implementation would add a duplicate entry -> Need to add deduplication logic. (follow the same pattern in `add_to_collection()` in `services/collection_service.py`)
- **What I did:** Added a deduplication check to `add_to_watchlist()` following the same pattern as `add_to_collection()`. I also defined a new `AlreadyInWatchlistError` exception class directly in `watchlist_service.py` rather than reusing `AlreadyInCollectionError` from `collection_service.py`. This keeps the two services independent and ensures the error name and message accurately reflect the watchlist domain.
- **How I verified:** I reviewed the updated file to confirm the deduplication query, the new exception class, and the corrected error message are all in place.

## Comment 3 — Missing test
**What I did:**
**How I verified:**

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->