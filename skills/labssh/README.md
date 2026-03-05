# labssh

Claude Code Skill for managing the labssh SSH host book.

## Features

- List and search SSH hosts
- Connect to lab servers by name or index
- Add, remove, and edit host entries
- Fuzzy and partial matching for host selection

## Tool Info

- Script: `/home/openhe/Workspace/software/labssh_tool/labssh`
- Config: `/home/openhe/Workspace/software/labssh_tool/labssh_hosts.tsv`
- TSV format with `<=>` syntax for display labels

## Usage

Copy `SKILL.md` to your project's `.claude/skills/` directory:

```bash
mkdir -p /path/to/your/project/.claude/skills/labssh
cp SKILL.md /path/to/your/project/.claude/skills/labssh/SKILL.md
```

Then trigger it in Claude Code via `/labssh`, e.g.:

```
/labssh list
/labssh search nyuair
/labssh ssh chatsign-164
/labssh add myserver user@10.0.0.1 dev-box
```

## Source

Standalone script at `/home/openhe/Workspace/software/labssh_tool/`.
