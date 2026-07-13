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
- **Detail**: add a test for the case where `film_id` doesn't exist in the database for the `add_to_watchlist()` feature, following the pattern in `test_add_to_collection_nonexistent_film_raises` from `tests/test_collection.py`.
- **What I did:** Added `test_add_to_watchlist_nonexistent_film_raises` to `tests/test_watchlist.py`, following the same structure as `test_add_to_collection_nonexistent_film_raises`. The test uses a fake UUID that doesn't exist in the in-memory test database and asserts that `FilmNotFoundError` is raised by `add_to_watchlist()`.
- **How I verified:** 
    - I reviewed the test to confirm it mirrors the pattern from `test_collection.py`, same fixture setup and assertion structure.
    - I ran `pytest tests/test_watchlist.py -v` and confirmed the test passed

## Comment 4 — Default visibility
- **Detail**: the reviewer noticed watchlists default to `public=True`. Since we don't have a documented decision on default visibility for user lists, before they can approve this, they need me to add a note to my PR description explaining my reasoning to  make sure I'm being intentional, not just inheriting a default.
- **Define the scope of issue:** the reviewer mentioned watchlists is default to `public=True`. However, when i did a project-wide search for the keyword `public=True`, I didn't see any result returned -> To look for the code associated with this issue, I had to find how a watchlist is initialized -> In `models.py` where the class `WatchlistEntry` is defined, i see `public = db.Column(db.Boolean, default=True)` -> This is where watchlist instances is defaulted to be public
- **My position:** I am keeping `public=True` as the default. It is an intentional design choice, not an inherited accident.
- **Reasoning:**
    - In `models.py`, the `public` field exists on `WatchlistEntry` but not on `CollectionEntry`. This visibility control difference is on purpose to allow `WatchlistEntry` to be set as public by default.
    - Even when `WatchlistEntry` is a personal watch-later list of films, it can be used for social sharing and discovery as users can browse their friends' watchlists and get film recommendations. Defaulting to `public=True` matches that intent and makes the social feature useful without requiring users to actively opt in.
- **Tradeoff acknowledged:** A `public=True` default means users who don't notice the setting have their watchlist exposed. The safer pattern for any user data is private-by-default and `CollectionEntry` implicitly follows that by having no `public` field at all. 


## Comment 5 — Sort order
- **Detail**: the reviewer prefers watchlists to default to "date added" order rather than alphabetical since most users want to see what they added recently. They want me to document my reasoning for my design decision
- **My position:** I am keeping the alphabetical sort (`Film.title ASC`). The maintainer's preference is reasonable, but I think it fits a feed better than a planning list.
- **Reasoning:** 
    - A watchlist is something users consult when deciding what to watch, so they'd scan it like a menu instead of scrolling  like a timeline. Alphabetical order makes it easy to find a specific title without remembering when they saved it. 
    - I see that the collection feature sorts by `date_added DESC`, but this is because it is a log of past activity where recency is meaningful. A watchlist shouldn't follow the same design since it has different semantics: all films in list are unfinished, so the order that makes it most browsable is more useful than the order that reflects how it was built.
**Engagement with reviewer's point:** The maintainer argues that "most users want to see what they added recently". That framing treats the watchlist as a feed of saves, where freshness signals relevance. That is a fair model if users add films impulsively and want to act on the most recent impulse first. But it also means a film added six months ago drifts to the bottom and may never surface again. Alphabetical keeps every title equally visible regardless of when it was saved, which better matches the "pick something to watch tonight" use case.

## Comment 6 — Rebase
**Commands run:**
```bash
git fetch origin
git rebase origin/main
```

**What conflicted:**

Two files conflicted during the 8-commit rebase:

1. **`.gitignore` (add/add conflict)** — commit `4f38cdc` (rename fix) brought a `.gitignore` that didn't include `.pytest_cache/`, while `origin/main` had added that entry via the "chore: add .gitignore" commit (`718a9a8`). Git flagged both sides as independently adding the file.

2. **`models.py` (content conflict)** — commit `07b8ffb` (change `WatchlistEntry.public` default to `False`) tried to edit the `WatchlistEntry` class, but `WatchlistEntry` no longer existed in the rebase's working tree at that point. The root cause: the UUID-migration commit on `main` (`07ca580` — "refactor: migrate film IDs from integer to UUID") deleted `WatchlistEntry` from `models.py` entirely when it migrated Film IDs. So by the time `07b8ffb` tried to touch that class, it had been wiped by the base-branch change.

**How I resolved it:**

1. **`.gitignore`** — Kept all entries from both sides: retained `.pytest_cache/` (from `origin/main`) plus `.venv/` and `venv/` (from the branch). This is a pure union merge — both sets of ignores are correct and non-overlapping.

2. **`models.py`** — Accepted the `WatchlistEntry` class in full as introduced by `07b8ffb` (with `public = db.Column(db.Boolean, default=False)`). The class had to be re-added because `origin/main`'s UUID migration deleted it. The intent of the commit — changing the default visibility — was preserved exactly. The subsequent commit `3e8b516` then applied cleanly on top and reverted the default back to `True` as intended.

**How I verified no conflict remains:**

- `git rebase --continue` completed all 8 commits without further errors, finishing with: `Successfully rebased and updated refs/heads/feature/watchlist.`
- Ran `grep -rn "<<<<<<"` across all `.py`, `.md`, and `.gitignore` files — no conflict markers found.
- Final log shows a clean linear history of 8 commits on top of `bbe206c` (origin/main tip) with no merge commits.
- `git log --oneline --merges origin/main..HEAD` shows no output, which means zero merge commits. The flag --merges filters to only merge commits; empty result confirms your branch history is clean.
    - A merge commit: is a special commit git creates when you join 2 branches together with git merge. It has two parent commits instead of one, and looks like this in the log:
    ```
    *   bbe206c Merge pull request #2 from ascherj/chore/add-gitignore
    |\  
    | * 718a9a8 chore: add .gitignore for generated files
    |/  
    * 07ca580 refactor: migrate film IDs from integer to UUID
    ```
    - Why rebasing avoids them: Instead of merging main into your branch, git rebase replays your commits one by one on top of the new base. The result is a straight line with no forks — every commit has exactly one parent. That's what your branch looks like now:
    ```
    * bd3af08 fix: add reasoning for watchlist's sort order
    * c22c088 fix: change position on the default visibility of watchlist
    * ...
    * bbe206c Merge pull request #2 (← origin/main tip)
    ```
    => A clean linear history is easier to read and review

- I can also visually confirm with the graph show when running `git log --oneline --graph origin/main..HEAD`  (See how many commits are on our branch relative to main)
    ```
    * bd3af08 (HEAD -> feature/watchlist) fix: add reasoning for watchlist's sort order
    * c22c088 fix: change position on the default visibility of watchlist
    * 9a9ea6d fix: change WatchlistEntry's public to False by default and add supporting reasoning
    * 34a9985 fix: add a new test_watchlist.py file with test case test_add_to_watchlist_nonexistent_film_raises
    * 032e05b fix: add deduplication logic to add_to_watchlist()
    * 7214421 fix: rename 'save_to_watchlist()' to 'add_to_watchlist()' to follow the naming convention
    * f95ecde fix: update film retrieval method to use db.session.get in collection and watchlist services
    * 51f358d added watchlist model and endpoint fixed a bug more changes
    ```
    Every line starts with * (a regular commit) — no |\  or |\ fork lines that indicate a merge commit. All 8 commits are in a straight line on top of origin/main.

## git log --oneline

![git log --oneline](git%20logs.png)

```
787951d (HEAD -> feature/watchlist) fix: update pr-response.md with documentation of the 'git rebase' process
e4fdc21 fix: update pr-response.md to add reasoning for watchlist's sort order decision
5eb44cc fix: update pr-response.md with default visibility decision for watchlist
67e78d1 test: add test for nonexistent film_id in add_to_watchlist
4f0df8b fix: add deduplication logic to add_to_watchlist() to prevent duplicate watchlist entries
dee2712 fix: rename 'save_to_watchlist()' to 'add_to_watchlist()' to follow the naming convention
75477cb feat: add .gitignore for generated files
4bb5181 fix: update WatchlistEntry film_id to UUID after main branch refactor
7c37bcd (origin/feature/watchlist) fix: update film retrieval method to use db.session.get in collection and watchlist services
ec90edb added watchlist model and endpoint fixed a bug more changes
014ae54 feat: initial CineLog API with film collection feature
```

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->