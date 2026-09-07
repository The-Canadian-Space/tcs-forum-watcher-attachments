# tcs-forum-watcher-attachments

Durable storage for Discord attachments mirrored by [tcs-forum-watcher](https://github.com/The-Canadian-Space/tcs-forum-watcher).

## Why this repo exists

When a user posts an image (or any file) in a tracked Discord forum thread, Discord serves it from its CDN. **Those CDN URLs expire in ~24h.** If the bot embedded the raw Discord URL in a GitHub issue body, that image would 404 the next day.

Instead, the bot downloads the bytes from Discord, uploads them here, and embeds the `raw.githubusercontent.com` URL — which never expires.

## What lives here

Every file corresponds to a Discord attachment that was mirrored into a GitHub issue tracked by the bot.

### Path scheme

```
<github_owner>/<github_repo>/<issue_number>/<sanitized_filename>
```

Example: `gitpush-mod/se-tiered-build-and-repair/2/grok-image-abc123.jpg`

- `sanitized_filename` = original name with anything outside `[a-zA-Z0-9._-]` replaced by `_`, then truncated to 100 chars.
- Multiple attachments per issue → multiple files under the same `<owner>/<repo>/<issue>/` directory.

### Lifecycle

| Trigger | What happens here |
|---|---|
| Discord user posts a file in the OP or a reply | Bot uploads bytes → creates `<owner>/<repo>/<issue>/<file>` |
| Discord user re-tags thread to a different project (Case B reroute) | Bot **moves** all files from old path → new path (byte-preserving copy, then delete of old) |
| GitHub issue is closed (`completed` or `not_planned`, non-reroute) | Bot deletes every file under `<owner>/<repo>/<issue>/*` |

## Do not commit manually

Everything here is managed by the bot. Committing files by hand won't break anything, but the bot has no record of your file — it will never be embedded in an issue, never migrated on reroute, never cleaned up on close.

If you need to recover a specific file that was deleted (e.g., after an accidental issue close), the git history has every byte:

```bash
# Find the commit that uploaded the file
gh api "repos/The-Canadian-Space/tcs-forum-watcher-attachments/commits?path=<full/path/to/file>&per_page=5" \
  --jq '.[] | {sha, message: .commit.message}'

# Fetch the content at that commit
gh api "repos/The-Canadian-Space/tcs-forum-watcher-attachments/contents/<full/path/to/file>?ref=<sha>" \
  --jq '.content' | tr -d '\n\r ' > restored.b64

# Re-upload via PUT (use --input for files larger than ARG_MAX)
```

## Related repos

- [tcs-forum-watcher](https://github.com/The-Canadian-Space/tcs-forum-watcher) — the bot that reads from + writes to this repo
- [The-Canadian-Space/.github](https://github.com/The-Canadian-Space/.github) — reusable workflows (unrelated to attachment storage)
