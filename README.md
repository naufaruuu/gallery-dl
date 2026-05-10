# gallery-dl (ll-archive fork)

This is a fork of [mikf/gallery-dl](https://github.com/mikf/gallery-dl)
maintained for the [ll-archive](https://github.com/naufaruuu/ll-archive)
project. The unmodified upstream README is at [`README.rst`](README.rst).

## Why this fork exists

ll-archive's download pipeline needs three things upstream doesn't
provide:

1. **Visibility into rate limits.** Every API request and rate-limit
   response is logged so the parent Go process can monitor budget.
2. **Visibility into pagination cursors.** Helps diagnose stuck pagination
   and trace backfills against `from:user max_id:N` search URLs.
3. **A bug fix in `max_id` search pagination.** Upstream loops forever
   on the boundary tweet at the end of a user's history. The fork
   detects non-advancing `max_id` and exits cleanly.

All changes live in `gallery_dl/extractor/twitter.py`, marked with
`# ll-archive-patch:` comments for easy review.

## Branches

| Branch | Role |
|---|---|
| `master` | Pure mirror of upstream `mikf/gallery-dl@master`. **Never commit here.** Only `git merge upstream/master` updates it. |
| `ll-archive-patches` | Upstream + our changes. **All commits go here.** |

The Docker image and local installs always use `ll-archive-patches`.

## Installing

### From this fork (consumers)

In the ll-archive Dockerfile or any environment that needs the patched
gallery-dl:

```bash
pip install git+https://github.com/naufaruuu/gallery-dl.git@ll-archive-patches
```

For reproducible builds, pin to a commit SHA instead of a branch name:

```bash
pip install git+https://github.com/naufaruuu/gallery-dl.git@<sha>
```

### Local development (editable)

Clone the fork, check out the patches branch, and install in editable
mode so further edits take effect immediately:

```bash
git clone git@github.com:naufaruuu/gallery-dl.git
cd gallery-dl
git checkout ll-archive-patches
pip install --user --break-system-packages --force-reinstall -e .
```

Verify:

```bash
python3 -c "import gallery_dl; print(gallery_dl.__file__)"
# → .../gallery-dl/gallery_dl/__init__.py
```

## Maintenance

See [`CLAUDE.md`](CLAUDE.md) for the full maintainer workflow:
adding/modifying patches, pulling new upstream releases, resolving
rebase conflicts.
