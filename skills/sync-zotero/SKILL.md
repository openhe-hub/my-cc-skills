---
name: sync-zotero
description: Sync Zotero PDF storage between local Mac and a remote SSH host via rsync. Zotero syncs metadata but not PDFs — this skill fills that gap.
argument-hint: [remote-host]
allowed-tools: Bash(rsync *) Bash(ssh *) Bash(ls *) Bash(du *) Bash(comm *) Bash(find *)
---

Bidirectional sync of Zotero storage (PDFs and attachments) between this Mac and a remote SSH host using rsync.

The remote host defaults to `legion` if `$ARGUMENTS` is empty. Otherwise use the host provided in `$ARGUMENTS`.

## Zotero storage paths

- **Local (macOS)**: `~/Zotero/storage/`
- **Remote (Linux)**: `~/Zotero/storage/`

## Steps

1. **Verify connectivity**: Run `ssh <host> echo ok` to confirm the remote host is reachable.

2. **Preview**: Compare both sides before transferring anything:
   - Count storage folders and total size on each side (`ls | wc -l`, `du -sh`).
   - Identify folders only on local, only on remote, and in common (use `comm`).
   - List the actual PDF filenames in local-only and remote-only folders so the user knows what will be synced in each direction.
   - Run `rsync -avhn` dry run for both directions and summarize.

3. **Confirm with the user**: Present a clear summary table showing:
   - What will be pushed (local -> remote).
   - What will be pulled (remote -> local).
   - Ask for confirmation before executing.

4. **Execute** (both directions):
   - **Push**: `rsync -avh --progress ~/Zotero/storage/ <host>:~/Zotero/storage/`
   - **Pull**: `rsync -avh --progress <host>:~/Zotero/storage/ ~/Zotero/storage/`

5. **Report**: Show the final rsync summary for both directions.

## Notes

- Default is **bidirectional** (merge both sides). Neither side loses files.
- Never use `--delete` unless the user explicitly requests it — we don't want to remove PDFs that exist only on one side.
