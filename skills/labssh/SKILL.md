---
name: labssh
description: Manage the labssh SSH host book — list, search, connect to lab servers, run remote commands, and edit the host configuration. Use when the user asks about SSH hosts, lab servers, remote machines, wants to connect to a specific host, or refers to a host by short name instead of full IP.
argument-hint: "[action] [name|pattern] [command...]"
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

# labssh — SSH Host Book Skill

You are managing SSH hosts via `labssh`, a lightweight CLI host book located at `/home/openhe/Workspace/software/labssh_tool/`.

The binary is at `/home/openhe/.local/bin/labssh` (symlink to the script).

## Host Configuration

- **Config file:** `/home/openhe/Workspace/software/labssh_tool/labssh_hosts.tsv`
- **Override:** set `LABSSH_CONFIG` environment variable to use a different file
- **Format:** TSV with 3 columns (tab-separated):

```
# name	target (use <=> to set display label)	note
myhost	user@192.168.1.1	optional note
myhost2	user@10.0.0.1<=>user@192.168.1.2	note here
```

### Column Details

| Column | Required | Description |
|--------|----------|-------------|
| `name` | yes | Unique identifier for the host |
| `target` | yes | SSH target (`user@host`), optionally with `<=>` or `=>` label mapping |
| `note` | no | Free-text note (e.g. `hpc-entry`, `ProxyJump:...`) |

### Target Mapping Syntax

- **Simple:** `user@host` — connects directly
- **With label:** `user@172.28.164.155<=>user@10.224.36.118` — connects to the left side, displays the right side as label
- **Alternative:** `user@host=>label` — same behavior with `=>` separator

## Resolving Host Names to SSH Targets

**This is a key capability.** When the user mentions a host by short name (e.g. "chatsign-164", "nyuair-212", "jubail1") anywhere in conversation — not just via `/labssh` — you should resolve it to the actual SSH target.

To resolve a host name to its SSH target:

```bash
labssh show <name>
```

Then use the target (the `user@ip` part) directly in `ssh` commands. This way the user never needs to type or remember full IPs.

**Example:** User says "check GPU status on chatsign-164":
1. Run `labssh show chatsign-164` → get target `chatsign@172.28.164.155`
2. Run `ssh chatsign@172.28.164.155 "nvidia-smi"`

## Common Actions

### List All Hosts

```bash
labssh list
# or just:
labssh
```

### Search Hosts

```bash
labssh search <pattern>
```

Searches across name, label, target, and note fields (case-insensitive).

### Run Remote Commands

Use `labssh show` to resolve the host, then `ssh <target> "<command>"` to execute:

```bash
# Single command
ssh <target> "nvidia-smi"

# Multiple commands
ssh <target> "cd /project && python train.py"

# Interactive-style work (run multiple commands sequentially)
ssh <target> "ls -la /data"
ssh <target> "df -h"
```

**Do NOT use `labssh ssh`** in Claude Code — it calls `execvp` which replaces the process and won't return output. Always resolve the target first, then use `ssh <target> "command"` directly.

### Show Host Details

```bash
labssh show <name|index>
```

### Host Selection Logic

1. Pure number → match by index
2. Exact match on name or label → select if unique, show candidates if multiple
3. Partial substring match → select if unique, show candidates if multiple

## Editing the Host Configuration

When the user wants to add, remove, or modify hosts, directly edit the TSV file:

```
/home/openhe/Workspace/software/labssh_tool/labssh_hosts.tsv
```

Rules:
- Use **tabs** (not spaces) to separate columns
- Lines starting with `#` are comments
- Empty lines are ignored
- `name` and `target` must be non-empty
- If using `<=>` or `=>`, both sides must be non-empty

## Handling User Requests

- If the user says **"$ARGUMENTS"**, interpret it as a labssh action:
  - `list` / `ls` / `hosts` / (empty) → run `labssh list`
  - `search <pattern>` / `find <pattern>` / `grep <pattern>` → run `labssh search <pattern>`
  - `run <name> <command...>` / `exec <name> <command...>` → resolve host via `labssh show <name>`, then run `ssh <target> "<command>"`
  - `show <name>` / `info <name>` → run `labssh show <name>`
  - `add <name> <target> [note]` → append a new line to the TSV config file
  - `remove <name>` / `delete <name>` → remove the matching line from the TSV config file (confirm first)
  - `edit` / `config` → open the TSV config file for editing
  - Otherwise, interpret the intent and act accordingly

- If the user mentions a host short name (like "chatsign-164") **outside of `/labssh`**, proactively resolve it via `labssh show` to get the SSH target, then work on that server directly. The user should never need to type a full IP address.
