# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end. Be specific: what did you ask, what did you do with the answer,
     and if you used AI on Comment 4 or 5, how does your final reasoning differ from what it gave you? -->


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
**What I did:**
Added a check in `add_to_watchlist()` that queries WatchlistEntry for an existing
row matching the same user_id and film_id before creating a new one. If a match
is found, it raises an error instead of silently creating a duplicate — same
approach as add_to_collection()'s AlreadyInCollectionError pattern.

**How I verified:**
Ran the full test suite (pytest tests/ -v) to confirm the existing 4 tests still
passed after the change. Used Claude to confirm the query pattern matched
collection_service.py's approach before writing it in.

## Comment 3 — Missing test
**What I did:**
Created tests/test_watchlist.py, mirroring test_add_to_collection_nonexistent_film_raises
from test_collection.py. Copied the same app/sample_user/sample_film fixtures, then
wrote test_add_to_watchlist_nonexistent_film_raises using a fake UUID and asserting
FilmNotFoundError is raised.

**How I verified:**
Ran pytest tests/test_watchlist.py -v — passed. Then ran the full suite
(pytest tests/ -v) — all 5 tests passed.

## Comment 4 — Default visibility
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

**My position:**

**Reasoning:**

**Tradeoff acknowledged:**


## Comment 5 — Sort order
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


## PR Description
<!-- Write this last. Must cover: what the watchlist feature does, both design decisions
     (visibility default + sort order), and step-by-step manual testing instructions. -->