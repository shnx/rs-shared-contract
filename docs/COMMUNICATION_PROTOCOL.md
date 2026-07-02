# Web ↔ iOS Communication Protocol

**Last updated:** July 2, 2026

## Channels (check ALL of these)

| Channel | What to check | URL |
|---------|--------------|-----|
| **GitHub Issues** | New comments on issues #1 and #2 | https://github.com/shnx/rs-shared-contract/issues |
| **shared/docs/ folder** | New or modified `.md` files | `shared/docs/` in the repo |
| **shared/CHANGELOG.md** | Schema version bumps and breaking changes | `shared/CHANGELOG.md` |
| **shared/schemas/** | JSON schema changes | `shared/schemas/` |

## Rules

### When you have an update or question
1. **Post a comment on GitHub issue #1** — this is the primary channel
2. **If it's a schema change**, also update `shared/` files and bump `CHANGELOG.md`
3. **If it's a long-form doc**, also add a `.md` file to `shared/docs/` and mention it in the issue comment

### When you check for updates
1. **Pull the shared submodule:** `cd shared && git pull`
2. **Check for new/changed docs:** `git log --oneline -10 -- docs/`
3. **Check GitHub issues:** `gh issue list --repo shnx/rs-shared-contract --state all` or visit the URL
4. **Read new comments:** `gh issue view 1 --repo shnx/rs-shared-contract --comments`

### Notification keywords
Use these in issue comments to flag urgency:
- **`@web-team`** — web team must read this
- **`@ios-team`** — iOS team must read this
- **`BREAKING`** — schema change that requires code updates
- **`ACTION REQUIRED`** — someone needs to do something

## Cadence
- **Check daily** during active development
- **Comment within 24 hours** of seeing a question directed at you
- **Bump CHANGELOG.md** for every schema change, no matter how small

## Quick check command (web team)

```bash
# Check for new iOS comments and shared doc changes in one command:
cd /Users/mohammadshannak/Documents/RS_website && \
  echo "=== Recent GitHub Issues ===" && \
  gh issue list --repo shnx/rs-shared-contract --state all --limit 5 && \
  echo "" && \
  echo "=== Latest issue #1 comments ===" && \
  gh issue view 1 --repo shnx/rs-shared-contract --comments 2>&1 | tail -30 && \
  echo "" && \
  echo "=== Shared submodule recent changes ===" && \
  cd shared && git pull --quiet 2>&1 && git log --oneline -10 && \
  echo "" && \
  echo "=== New/changed docs ===" && \
  git log --oneline -10 -- docs/ && \
  echo "" && \
  echo "=== Changelog ===" && \
  head -20 CHANGELOG.md
```
