# Claude Instructions

Instructions for Claude Code when working in this repo setup.

---

## Repos

| Repo | Local Path | Remote | Purpose |
|---|---|---|---|
| `coding_practice` | `/home/wukong/Robot_practice/` | `github.com/wukongxzero/coding_practice` | C++ NeetCode 150 solutions |
| `Robotics_Knowledge_Vault` | `/home/wukong/Downloads/Robotics_Knowledge_Vault/` | `github.com/wukongxzero/Robotics_Knowledge_Vault` | Private Obsidian vault |
| `robotics-swe-prep` | `/home/wukong/Downloads/robotics-swe-prep/` | `github.com/wukongxzero/robotics-swe-prep` | Public open-source RSE prep mirror |

---

## Push Workflow

When committing vault notes (new or updated `.md` files):
1. Commit and push to `Robotics_Knowledge_Vault` (private)
2. Copy the same files to `robotics-swe-prep` and push there too (public)

When committing DSA solutions:
1. Commit and push to `coding_practice`
2. If the solution has a vault entry, update `DSA Patterns.md` and push vault too

---

## Vault Sync Rules

After any DSA commit, add the solution to `DSA Patterns.md` with:
- Pattern tag
- Signal words (what in the problem hints this pattern)
- Key insight
- C++ code
- Complexity
- Common traps
- Flashcard entries

---

## Git Rules

- **Never commit ELF binaries** — compiled binaries with no extension (`lcs`, `topk`, `gird_pathfinding`, `kalman_filter`, etc.) are build artifacts. Skip them.
- **Always stage specific files by name** — never `git add .` or `git add -A`
- **Never commit `.obsidian/workspace.json` or `.obsidian/plugins/*/data.json`** — these are Obsidian app state, not content

---

## Behavior

- Explain the *why* behind fixes and patterns, not just what changed — user is building interview fluency
- Review code and show diff before committing when asked
- Keep responses concise
