# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used an AI assistant (Claude) throughout, and verified its output against the actual code and the Conventional Commits spec rather than taking it at face value:

- **Orientation (Milestone 1):** Had it summarize `models.py`, `services/collection_service.py`, and `tests/test_collection.py`, and walk me through `add_to_collection()` step by step. I checked each claim against the source — e.g., confirmed the `UniqueConstraint` on `CollectionEntry` and the exact exception types/HTTP codes — before relying on it.
- **Pattern reference (Comment 2):** Used the `add_to_collection()` walkthrough as the model for my dedup logic (existence guard → query for an existing entry → raise *before* `add`/`commit`). I wrote the check myself rather than having AI generate it.
- **Test structure (Comment 3):** Used `test_collection.py` as the template for the fixtures and the `pytest.raises` assertion style.
- **Devil's-advocate on the design decisions (Comments 4 & 5):** After drafting each position I asked, "what's the strongest counterargument a careful reviewer would raise, and what tradeoff am I not acknowledging?"
  - *Comment 4 (visibility):* It raised **privacy-by-design** (GDPR Art. 25) — the safe default is the least-sharing option. I had argued for `public=True` unconditionally; as a result I revised my position to make it *conditional* on a visible per-entry privacy toggle + an onboarding disclosure, and to concede that without those, `public=False` is the correct default.
  - *Comment 5 (sort):* It raised **YAGNI** — "ship the date default; add the `?sort=title` param only when a user asks." I addressed this by grounding the param in the fact that alphabetical sort *already existed* in the inherited code, so `?sort=title` **preserves** an existing behavior rather than speculatively adding one.
- **Commit-format check (Milestone 4):** Ran my `git log --oneline` past AI to sanity-check conventional-commit prefixes and one-change-per-commit, then verified against the spec myself.

## Comment 1 — Rename
> **@dev-lead (inline, `services/watchlist_service.py:12`):** "`save_to_watchlist()` should follow the project's naming convention. Compare with `add_to_collection()` — the pattern here is `verb_to_noun`. Please rename to `add_to_watchlist()` and update all call sites."

**What I did:** Renamed `save_to_watchlist()` → `add_to_watchlist()` in `services/watchlist_service.py`, matching the `verb_to_noun` convention used by `add_to_collection()` / `remove_from_collection()` / `get_collection()`. Updated the one call site in `routes/watchlist/watchlist.py` (both the import statement and the call inside `add_film()`), and updated the function's docstring summary line from "Save a film…" to "Add a film…" so the prose matches the new name.

**How I verified:** Before editing I ran a project-wide search (`grep -rn save_to_watchlist`) to enumerate every reference — it found exactly three (the definition, the import, the call). After renaming, I re-ran the same search and it returned no matches outside `pr-response.md`. I also confirmed the app still imports cleanly (`python -c "from app import create_app; create_app()"`) so the blueprint wiring didn't break. Committed as its own commit: `fix: rename save_to_watchlist to add_to_watchlist per naming convention`.

## Comment 2 — Deduplication
> **@dev-lead (inline, `services/watchlist_service.py:30`):** "What happens if a user calls this with a film that's already on their watchlist? The current implementation would add a duplicate entry. Please handle this case."

**What I did:** I followed the exact pattern `add_to_collection()` already uses for this case, which has two layers:
1. **Service-level guard** (`services/watchlist_service.py`): after the film-exists check, `add_to_watchlist()` now queries for an existing `WatchlistEntry` with the same `(user_id, film_id)`; if one is found it raises a new `AlreadyInWatchlistError` instead of inserting. This mirrors `AlreadyInCollectionError` in `collection_service.py`.
2. **DB-level constraint** (`models.py`): added `UniqueConstraint("user_id", "film_id", name="unique_user_film_watchlist")` to `WatchlistEntry`, mirroring `CollectionEntry`'s `unique_user_film_collection`. This is defense-in-depth so a race between two concurrent adds can't slip a duplicate past the service check.

I also updated the route (`routes/watchlist/watchlist.py`) to catch the new exception and return **409 Conflict**, and to catch the already-raised `FilmNotFoundError` and return **404** — previously the route caught neither, so both cases produced an unhandled 500. This matches how `routes/collection.py` maps these errors.

**How I verified:** Wrote `test_add_to_watchlist_duplicate_raises` (see Comment 3) which adds the same film twice, asserts `AlreadyInWatchlistError` is raised, and asserts exactly one row exists afterward — it passes. I traced the `add_to_collection()` logic first to confirm the reference behavior: on a duplicate it raises `AlreadyInCollectionError` *before* calling `db.session.add`, so nothing is persisted and no commit happens. My implementation does the same (raise before add/commit). Committed separately: `fix: add deduplication check to prevent duplicate watchlist entries`.

## Comment 3 — Missing test
> **@dev-lead (conversation):** "Please add a test for the case where `film_id` doesn't exist in the database. Look at the existing tests in `test_collection.py` — the pattern is there."

**What I did:** Created `tests/test_watchlist.py`. I used `test_add_to_collection_nonexistent_film_raises` as my model for the required test — same structure: an `app`/`sample_user` fixture pair, an `app.app_context()` block, a bogus id, and `pytest.raises(FilmNotFoundError)`. Per CONTRIBUTING.md's rule that a new service function ships with happy-path + duplicate + nonexistent tests, I also added `test_add_to_watchlist_creates_entry` and `test_add_to_watchlist_duplicate_raises`, reusing the `app` / `sample_user` / `sample_film` fixture pattern copied verbatim from `test_collection.py` so the style matches the rest of the suite.

**How I verified:** `pytest tests/test_watchlist.py -v` → 3 passed. `pytest tests/ -v` → 7 passed (4 existing + 3 new), confirming the new tests and my Comment 1/2 changes didn't break the collection suite. Committed: `test: add watchlist service tests for add, duplicate, and nonexistent film`.

## Comment 4 — Default visibility
> **@dev-lead (conversation):** "I notice watchlists default to `public=True`. We don't have a documented decision on default visibility for user lists. Before I can approve this, I need you to add a note to your PR description explaining your reasoning. I want to make sure we're being intentional here, not just inheriting a default."

**My position:** Keep the default `public=True`, *conditioned on* a visible per-entry privacy toggle and a first-run disclosure (see tradeoff). This is an intentional decision, not an inherited default.

**Reasoning:** CineLog is described in the README as "a community film tracking app." The product's value is users seeing each other's film activity, and the app already treats a user's **collection** as a shared surface — `GET /collection/<user_id>` returns any user's watched films by id with no visibility gate. A watchlist ("films I intend to watch") is a natural extension of that same community surface: public watchlists drive discovery ("what's everyone planning to watch?"), enable social nudges, and feed future recommendation features. Defaulting watchlists to public keeps the mental model consistent with the collection — if the collection is public but the watchlist is private, that's a confusing asymmetry for both users and maintainers. I'm optimizing for **frictionless community discovery and consistency with the app's existing shared surfaces.**

**Tradeoff acknowledged:** Public-by-default cuts against privacy-by-design (e.g., GDPR Art. 25), where the safe default is the least-sharing option and users opt *in* to exposure. A watchlist is arguably more sensitive than a collection: a collection is a settled fact ("I watched this"), while a watchlist signals *intent*, which can be more personal. So public-by-default means users share before explicitly choosing to. I accept that cost **only** because (a) `public` is already a per-entry column, so users can flip any single entry private, and (b) it should ship with an onboarding disclosure so the default is *informed*. Absent that toggle-and-disclosure, I'd flip the default to `public=False` — privacy-by-default is the correct call when the sharing benefit isn't made explicit to the user.

*(Devil's-advocate check — see AI Usage. The strongest counterargument raised was the privacy-by-design principle; it made me add the explicit toggle + disclosure condition rather than defend an unconditional public default.)*

## Comment 5 — Sort order
> **@dev-lead (inline, `services/watchlist_service.py:50`):** "I'd prefer watchlists to default to \"date added\" order rather than alphabetical. Most users want to see what they added recently. I'm open to discussion if you see it differently — but let's make a decision and document it."

**My position:** Make sort order a query param — `GET /watchlist/<user_id>?sort=date|title` — **defaulting to `date` (newest first)**. So the out-of-the-box behavior is exactly what the maintainer asked for, and alphabetical remains reachable via `?sort=title`. (Implemented in `feat: add sort option to watchlist, default to date added`.)

**Reasoning:** I agree with the maintainer on the default. "What did I just add?" is the dominant access pattern for a watchlist, and — importantly — `get_collection()` already sorts `date_added` descending. Making the watchlist default match means the two list endpoints behave consistently, which is less surprising for users and less special-casing for us. That consistency argument is actually stronger than the raw "most users want recency" claim, because it also reduces maintenance cost.

**Engagement with reviewer's point:** The maintainer said "I'm open to discussion if you see it differently," so here's the nuance I want on record: the original alphabetical sort wasn't wrong for *one* real case — when a watchlist gets long and a user is hunting for a specific known title to start watching tonight, A–Z is faster to scan than reverse-chronological. Rather than pick a single winner and lose that, I preserved both behind a param. Crucially, alphabetical **already existed** in the inherited code, so `?sort=title` keeps an existing capability rather than adding a speculative one — I'm neither inventing a feature nor silently deleting one.

**Tradeoff acknowledged:** A query param is a small increase in API surface and test burden (I added tests for both orderings). It also doesn't dodge the core decision — a default still had to be chosen, and I chose `date` per the maintainer. If the team would rather keep the endpoint dead-simple, collapsing to date-only is a one-line change.

*(Devil's-advocate check — see AI Usage. Counterargument raised: "YAGNI — ship the date default, add the param when someone asks for title sort." My response, added to the reasoning above: alphabetical was pre-existing behavior, so the param preserves rather than speculates.)*

## Comment 6 — Rebase
> **@dev-lead (conversation):** "A refactor merged to `main` that changed film IDs from integers to UUIDs. Your watchlist code still references integer IDs. Please rebase on `main` and update accordingly."

**What conflicted:** The branch was cut from `014ae54` — *before* the `refactor: migrate film IDs from integer to UUID` commit landed on `main`. Running `git rebase origin/main` replayed my 7 commits onto the UUID `main` and stopped with a **content conflict in `models.py`**: the watchlist commit appends a `WatchlistEntry` class to the end of the file, while the refactor rewrote `Film`/`CollectionEntry` above it to use `db.String(36)` UUID ids. Git flagged the overlap because both sides touched `models.py`. The service/route files applied cleanly (no textual conflict).

**How I resolved it:** I kept **both** sides — `main`'s UUID `Film`/`User`/`CollectionEntry` *and* my `WatchlistEntry` class — since they're complementary, not competing. But the important part is what git *couldn't* see: resolving the text conflict left `WatchlistEntry.film_id` as `db.Integer` pointing at `Film.id`, which is now `db.String(36)`. A textual 3-way merge has no way to know that FK is now semantically broken. So after finishing the rebase I made a **dedicated follow-up commit** (`fix: change WatchlistEntry.film_id to UUID after main branch refactor`) that changes the column to `db.String(36)` and updates the now-stale docstrings (`film_id (int)` → `film_id (str): UUID`; route body `<int>` → `"<uuid>"`).

**How I verified no conflict remains:**
- `grep -n "Integer" models.py` — the only remaining `Integer` columns are `Film.year` and `CollectionEntry.rating`; no `film_id` is integer anymore.
- `git log --merges origin/main..HEAD` is empty → no merge commits; history is linear.
- `pytest tests/ -q` → 9 passed, and `python -c "from app import create_app; create_app()"` imports cleanly.
- **End-to-end smoke test** with the Flask test client against the rebased UUID schema: film ids come back as 36-char UUID strings; `POST /watchlist/<id>/add` → 201; duplicate → 409; nonexistent film → 404; missing body → 400; `GET /watchlist/<id>` returns newest-first and `?sort=title` returns A–Z. All passed.

## Commit History (Milestone 4)
Final `git log --oneline` for `feature/watchlist` (linear, conventional, no merge commits):

```
<tip>    docs: add PR response doc with design decisions and testing steps
5071950  fix: change WatchlistEntry.film_id to UUID after main branch refactor
5544038  feat: add sort option to watchlist, default to date added (newest first)
28806db  fix: add Film.watchlist_entries relationship so get_watchlist can load films
87e2cdc  test: add watchlist service tests for add, duplicate, and nonexistent film
b11a9a4  fix: add deduplication check to prevent duplicate watchlist entries
5b0a8ac  fix: rename save_to_watchlist to add_to_watchlist per naming convention
3ff3172  fix: update film retrieval method to use db.session.get in collection and watchlist services
4647a19  feat: add watchlist model, service, and endpoint
```

### Screenshot

![git log --oneline on feature/watchlist — 9 conventional commits, no merge commits](docs/git-log.png)

Each commit is one logical change, uses Conventional Commits format, and builds on its own (verified: `create_app()` imports cleanly at every commit). No merge commits — the branch is linearly rebased on `main`.

## PR Description

### What this feature does
Adds a **watchlist** to CineLog — films a user *intends* to watch, distinct from the collection (films already watched). It introduces a `WatchlistEntry` model, a service layer (`add_to_watchlist()` / `get_watchlist()`), and two endpoints:

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/watchlist/<user_id>/add` | Add a film. Body `{"film_id": "<uuid>"}`. → **201**; **400** if `film_id` missing; **404** if the film doesn't exist; **409** if it's already on the watchlist. |
| GET | `/watchlist/<user_id>` | List the watchlist. Optional `?sort=date` (default, newest first) or `?sort=title` (A–Z). |

Duplicates are prevented at two levels (a service-side check and a `(user_id, film_id)` unique constraint), mirroring `CollectionEntry`. Film ids are UUIDs, consistent with the rest of the schema after the integer→UUID refactor.

### Design decisions
1. **Default visibility is `public=True`** — CineLog is a community app and the collection is already a shared surface, so public watchlists support discovery. This is intentional and should ship with a per-entry privacy toggle + onboarding disclosure; absent those, `public=False` would be the safer default. (Full reasoning + tradeoff under *Comment 4* above.)
2. **Default sort is date-added (newest first), with `?sort=title` available** — matches `get_collection()`'s convention (recency is the common need) while preserving the inherited alphabetical option for scanning a long list. (Full reasoning under *Comment 5* above.)

### How to manually test end to end
```bash
# 1. Set up
python -m venv .venv && source .venv/Scripts/activate   # Windows Git Bash
pip install -r requirements.txt

# 2. Seed a user + two films (no admin endpoint exists, so use a shell)
python - <<'PY'
from app import create_app, db
from models import User, Film
app = create_app()
with app.app_context():
    u = User(username="ada", email="ada@example.com")
    f1 = Film(title="Zodiac", year=2007, genre="Thriller")
    f2 = Film(title="Amelie", year=2001, genre="Romance")
    db.session.add_all([u, f1, f2]); db.session.commit()
    print("USER", u.id); print("FILM1", f1.id); print("FILM2", f2.id)
PY

# 3. Run the app and exercise the endpoints (substitute the ids printed above)
python app.py            # in one terminal; app serves on http://127.0.0.1:5000
curl -X POST localhost:5000/watchlist/<USER>/add -H "Content-Type: application/json" -d '{"film_id":"<FILM1>"}'   # 201
curl -X POST localhost:5000/watchlist/<USER>/add -H "Content-Type: application/json" -d '{"film_id":"<FILM1>"}'   # 409 duplicate
curl -X POST localhost:5000/watchlist/<USER>/add -H "Content-Type: application/json" -d '{"film_id":"nope"}'      # 404 no such film
curl -X POST localhost:5000/watchlist/<USER>/add -H "Content-Type: application/json" -d '{}'                      # 400 film_id required
curl localhost:5000/watchlist/<USER>            # default: newest first
curl "localhost:5000/watchlist/<USER>?sort=title"   # alphabetical

# 4. Or just run the test suite
pytest tests/ -v         # 9 passed
```
