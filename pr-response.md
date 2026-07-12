# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude throughout this project for a few specific things:

- **Orientation**: Before touching any review comments, I had Claude help me read
  through collection_service.py and test_collection.py to understand the existing
  naming conventions and dedup pattern before writing my own version for the
  watchlist feature.
- **Comment 2 (deduplication)**: I got stuck on the exact query syntax for the
  duplicate check. Claude pointed me to the existing pattern in add_to_collection()
  (query by user_id + film_id, raise an error if found), and I adapted it into
  add_to_watchlist() myself. I verified it didn't break existing tests with pytest,
  and later manually tested the duplicate case with curl.
- **Commit history cleanup (Milestone 4)**: Used Claude to help troubleshoot an
  interactive rebase that kept hitting merge conflicts when trying to reorder and
  squash commits in place. Ended up using `git reset --soft origin/main` to
  uncommit everything while preserving all file changes, then recommitting in
  clean, logical groups. All actual code logic (dedup check, sort order, private
  default, rename) was written by me — this was purely commit-history mechanics.
- **Comments 4 and 5**: I formed my own positions first (private-by-default for
  visibility, date-added for sort order) based on my own reasoning about what a
  watchlist represents versus what's already-watched. I didn't have Claude
  generate the arguments — I stated my position and reasoning, and it helped me
  make sure the writing was concrete rather than a vague one-liner.


## Comment 1 — Rename
> @jamjamgobambam: "save_to_watchlist() should follow the project's naming convention.
> Compare with add_to_collection() — the pattern here is verb_to_noun. Please rename to
> add_to_watchlist() and update all call sites."

**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`
to match the verb_to_noun pattern used by `add_to_collection()`. Updated both the import
and the function call in `routes/watchlist/watchlist.py`.

**How I verified:**
Ran `grep -rn "save_to_watchlist" .` before making changes to find every reference —
it caught 3 places: the function definition and 2 spots in the routes file (import +
call site). Ran it again after editing to confirm zero matches remained (aside from the
review comment itself quoted in this doc, which isn't a code reference). Re-ran
`pytest tests/ -v` afterward and all 4 tests still passed.


## Comment 2 — Deduplication
> @jamjamgobambam: "What happens if a user calls this with a film that's already on their
> watchlist? The current implementation would add a duplicate entry. Please handle this case."

**What I did:**
Added a check in `add_to_watchlist()` that queries WatchlistEntry for an existing
row matching the same user_id and film_id before creating a new one. If a match
is found, it raises an error instead of silently creating a duplicate — same
approach as add_to_collection()'s AlreadyInCollectionError pattern.

**How I verified:**
Ran the full test suite (pytest tests/ -v) to confirm the existing 4 tests still
passed after the change. Used Claude to confirm the query pattern matched
collection_service.py's approach before writing it in. Manually tested with curl
by adding the same film twice in a row and confirming the second call fails.


## Comment 3 — Missing test
> @jamjamgobambam: "Please add a test for the case where film_id doesn't exist in the
> database. Look at the existing tests in test_collection.py — the pattern is there."

**What I did:**
Created tests/test_watchlist.py, mirroring test_add_to_collection_nonexistent_film_raises
from test_collection.py. Copied the same app/sample_user/sample_film fixtures, then
wrote test_add_to_watchlist_nonexistent_film_raises using a fake UUID and asserting
FilmNotFoundError is raised.

**How I verified:**
Ran pytest tests/test_watchlist.py -v — passed. Then ran the full suite
(pytest tests/ -v) — all 5 tests passed.


## Comment 4 — Default visibility
> @jamjamgobambam: "I notice watchlists default to public=True. We don't have a documented
> decision on default visibility for user lists. Before I can approve this, I need you to
> add a note to your PR description explaining your reasoning. I want to make sure we're
> being intentional here, not just inheriting a default."

**My position:**
Watchlists should default to public=False (private), not True.

**Reasoning:**
A watchlist reveals intent — what someone *wants* to watch — which can expose more
about a person than a list of films they've already seen. Someone might have films
on there tied to something personal or embarrassing, and defaulting to public means
that gets exposed before the user has even thought about who can see it. Requiring
an explicit opt-in respects that this data is more sensitive than a "watched" list,
and puts the choice in the user's hands rather than assuming it for them.

**Tradeoff acknowledged:**
Defaulting to private weakens the social/discovery layer of the app — friends can't
stumble onto what someone's planning to watch, and there's less organic, shareable
behavior driving engagement. Private-by-default is the safer choice for users, but
it's a real cost to growth and virality compared to defaulting public.


## Comment 5 — Sort order
> @jamjamgobambam: "I'd prefer watchlists to default to 'date added' order rather than
> alphabetical. Most users want to see what they added recently. I'm open to discussion
> if you see it differently — but let's make a decision and document it."

**My position:**
I agree with the reviewer — watchlists should default to date-added order
(newest first), not alphabetical.

**Reasoning:**
A watchlist is a "what's next" list, not a reference catalog. Most users check
it to decide what to watch right now, and what they added most recently is
usually what's top of mind — often because they just heard about it or saw a
trailer. Alphabetical order buries that recency signal and makes users hunt
for it every time. It also matches how get_collection() already works — that
sorts by date_added descending, so keeping watchlist alphabetical would mean
the two features behave inconsistently for what is fundamentally the same
kind of "list of films tied to a date" pattern.

**Engagement with reviewer's point:**
The reviewer's reasoning was "most users want to see what they added recently,"
which is exactly the case here — I don't see a strong argument for alphabetical
being more useful for a watchlist specifically (it's more useful for something
like a media library you browse, not a to-do-style list). Implementing date-added
order also brings this feature in line with the collection service's existing
convention rather than introducing a second, inconsistent pattern.


## Comment 6 — Rebase
> @jamjamgobambam: "A refactor merged to main that changed film IDs from integers to
> UUIDs. Your watchlist code still references integer IDs. Please rebase on main and
> update accordingly."

**What conflicted:**
Git flagged a conflict in .gitignore — main had its own .gitignore added in a
separate PR while my branch was open, so both branches added the file
independently. The actual UUID migration didn't show as a traditional conflict,
but the rebase auto-merged in a way that silently dropped the WatchlistEntry
class from models.py entirely, since main's version of models.py didn't include
it and git took main's full file content.

**How I resolved it:**
Manually merged the two .gitignore lists into one. For the missing model, I
re-added the WatchlistEntry class to models.py, matching the UUID pattern
already used by CollectionEntry (String(36) primary keys, same foreign key
structure to user.id and film.id), plus the date_added and public fields that
services/watchlist_service.py already expected. I also fixed a stale docstring
that still described film_id as an integer, and reordered the duplicate check
in add_to_watchlist() to run before constructing the entry object.

**How I verified no conflict remains:**
Ran pytest tests/ -v after each fix — all 5 tests passed. Ran
git log --oneline --graph to confirm the branch history is a straight line
on top of main with no merge commits. Manually reviewed models.py and
watchlist_service.py to confirm no leftover conflict markers or
integer-ID assumptions remained.


## Commit History

```
d4a0a6a (HEAD -> feature/watchlist) test: add test for nonexistent film_id in add_to_watchlist
e6758da fix: update film retrieval to use db.session.get
1178bf9 docs: add pr-response.md with design decisions and AI usage
d8031a3 feat: add watchlist model and add_to_watchlist endpoint
bbe206c (upstream/main, origin/main, main) Merge pull request #2 from ascherj/chore/add-gitignore
```

*(Insert your `git log --oneline` screenshot here.)*


## PR Description

### What this feature does
Adds a watchlist feature to CineLog, letting users save films they want to watch
(as opposed to the collection feature, which tracks films they've already watched).
Includes a WatchlistEntry model, add_to_watchlist()/get_watchlist() service functions,
and a POST /watchlist/<user_id>/add endpoint.

### Design decisions
- **Default visibility (Comment 4)**: Watchlists default to `public=False` (private).
  A watchlist reveals intent — what someone *wants* to watch — which can expose more
  about a person than an already-watched list. Users have to explicitly opt in to
  sharing it. Tradeoff: this weakens organic social discovery compared to defaulting
  public.
- **Sort order (Comment 5)**: Watchlists are sorted by date added, newest first
  (matching the same convention as the collection feature's get_collection()).
  A watchlist is a "what's next" list, not a reference catalog — recency is more
  useful here than alphabetical order.

### How to manually test
1. Start the app: `python app.py`
2. Add a film to a watchlist:
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id>"}'
   ```
   Should return 201 with the new entry, and `public` should be `false` by default.
3. Run the same command again with the same user_id/film_id — should fail with a
   duplicate error instead of creating a second entry.
4. Add 2+ different films at different times, then fetch the watchlist:
   ```
   curl http://127.0.0.1:5000/watchlist/<user_id>
   ```
   The most recently added film should appear first in the list.
5. Run the automated test suite: `pytest tests/ -v` — all 5 tests should pass.