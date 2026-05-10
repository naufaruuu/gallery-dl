# CLAUDE.md — gallery-dl (ll-archive fork)

Maintainer instructions for this fork. For the "what is this" overview,
see [`README.md`](README.md).

## Branch model

```
upstream/master  ←  fetched from mikf/gallery-dl
       ↓
   origin/master                          (this fork's mirror — read-only)
       ↓
origin/ll-archive-patches                 (our patches replayed on top)
```

**Rules:**
- Never commit to `master`. It exists only to track upstream.
- All custom code goes on `ll-archive-patches`.
- The patches branch is rebased (not merged) onto master after each
  upstream pull, so it stays linear and easy to diff.

## Where the patches live

All edits are in `gallery_dl/extractor/twitter.py`, marked with
`# ll-archive-patch:` comments. Search for that string to find every
modification:

```bash
grep -n "ll-archive-patch" gallery_dl/extractor/twitter.py
```

There are currently three patches, in this order:

1. **REQ logging** — in `_call`, log `method endpoint cursor variables`
   before each HTTP request.
2. **RATELIMIT logging** — in `_call`, log `status remaining/limit reset url`
   after each response. Reads `x-rate-limit-*` headers.
3. **max_id loop fix** — in `_update_variables_search_maxid`, track the
   previous `max_id` value and return `None` when it doesn't advance.
   In `_pagination_tweets`, honor `None` from `update_variables` to
   exit cleanly.

## Adding or modifying a patch

```bash
git checkout ll-archive-patches
# edit gallery_dl/extractor/twitter.py
git add gallery_dl/extractor/twitter.py
git commit -m "twitter: <what changed and why>"
git push origin ll-archive-patches
```

Then in the ll-archive main repo, refresh the patch backup file:

```bash
cd /path/to/ll-archive
cd third_party/gallery-dl  # or wherever your local clone lives
git format-patch master..ll-archive-patches \
  -o /path/to/ll-archive/patches/gallery-dl/
```

Rebuild the api container to pick up the new commit:

```bash
docker compose build --no-cache api
```

## Pulling upstream updates

When mikf/gallery-dl ships a new release:

```bash
# 1. Update master (pure fast-forward, no conflicts possible)
git fetch upstream
git checkout master
git merge --ff-only upstream/master
git push origin master

# 2. Replay our patches on top
git checkout ll-archive-patches
git rebase master

# 3. If conflicts arise (look for `# ll-archive-patch:` markers as anchors):
#    - Resolve in twitter.py
#    - git add gallery_dl/extractor/twitter.py
#    - git rebase --continue

# 4. Push (force is required after rebase)
git push --force-with-lease origin ll-archive-patches

# 5. Refresh the patch backup file in ll-archive main repo
git format-patch master..ll-archive-patches \
  -o /path/to/ll-archive/patches/gallery-dl/
```

## Conflict-resolution hints

The patches anchor on these upstream symbols. If upstream renames or
restructures any of them, expect a rebase conflict:

- `def _call(self, endpoint, params, ...)` — the request loop. Our
  REQ + RATELIMIT logs sit immediately around `self.extractor.request(...)`
  and the `x-rate-limit-remaining` read.
- `def _update_variables_search_maxid(...)` — the search pagination
  step. Our loop-detection check goes right after `max_id` is computed
  but before the regex substitution.
- `def _pagination_tweets(...)` — the outer pagination loop. Our `None`
  handling goes where `update_variables(...)` is called.

If any of these move/rename, port the change manually using the
`# ll-archive-patch:` markers as your guide.

## Testing changes

Quick smoke test (uses ~2 API requests):

```bash
gallery-dl --cookies /path/to/twt.txt -o ratelimit=abort \
  --simulate --range 1-1 --print "{tweet_id}" \
  "https://x.com/search?q=from:USERNAME+max_id:9999999999999999999+filter:media&f=live"
```

Expected output includes:

```
[twitter][info] REQ GET UserByScreenName cursor=None variables=...
[twitter][info] RATELIMIT status=200 remaining=N/150 reset=... url=UserByScreenName
[twitter][info] REQ GET SearchTimeline cursor=None variables=...
[twitter][info] RATELIMIT status=200 remaining=N/50 reset=... url=SearchTimeline
```

If you see request URLs with full feature flags (urllib3 debug spam)
but no REQ / RATELIMIT lines, the patches aren't loaded — verify
`pip install -e .` ran successfully and `import gallery_dl` resolves
to this checkout.

## Contributing back to upstream

If a patch is generally useful (the max_id loop fix likely is), open a
PR against `mikf/gallery-dl`:

```bash
git checkout master
git checkout -b fix-<short-description>
git cherry-pick <commit-sha-from-ll-archive-patches>
git push origin fix-<short-description>
# Open PR on GitHub
```

Drop the cherry-picked commit from `ll-archive-patches` once upstream
merges it.
