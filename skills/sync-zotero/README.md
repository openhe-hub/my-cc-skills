# sync-zotero

Claude Code Skill for syncing Zotero PDF storage between machines via rsync.

## Why

Zotero syncs metadata (titles, tags, notes) across devices, but PDF attachments require a paid storage plan or WebDAV setup. This skill uses rsync over SSH to keep PDF files in sync — free and fast.

## Features

- Bidirectional sync (local <-> remote), neither side loses files
- Dry-run preview before any transfer
- Shows exactly which PDFs will be pushed/pulled
- Defaults to `legion` host, configurable via argument

## Usage

Copy `SKILL.md` to your personal or project skills directory:

```bash
mkdir -p ~/.claude/skills/sync-zotero
cp SKILL.md ~/.claude/skills/sync-zotero/SKILL.md
```

Then invoke in Claude Code:

```
/sync-zotero            # sync with default host (legion)
/sync-zotero myserver   # sync with a specific SSH host
```

## Requirements

- `rsync` installed on both machines
- SSH access to the remote host (key-based auth recommended)
- Zotero installed on both machines with default storage path (`~/Zotero/storage/`)
