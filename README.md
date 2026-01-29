# My ~Claude Code~ Agent Skills

Collection of personal agent-agnostic skills. Feel free to use, no warranty.
These skills are compatible with:
* Claude (via the Claude skills directory)
* Codex (via the Codex skills directory)
* Other agents that support the same [skills specification](https://developers.openai.com/codex/skills/)


## Skills

| Skill | Description |
|-------|-------------|
| [iterative-plan-review](./iterative-plan-review) | Plan creation and review loop with configurable reviewer |
| [iterative-implementation](./iterative-implementation) | Task implementation with worktrees + review |

## Installation
```bash
# Clone
git clone https://github.com/kareemf/claude-skills.git ~/claude-skills

# Ensure skills directories exist (they should already)
mkdir -p ~/.claude/skills
mkdir -p ~/.codex/skills

# Symlink the entire repo under a `personal` namespace
ln -sfn ~/claude-skills ~/.claude/skills/personal
ln -sfn ~/claude-skills ~/.codex/skills/personal

# Or cherry-pick skills individually
ln -s ~/claude-skills/iterative-implementation ~/.claude/skills/iterative-implementation
ln -s ~/claude-skills/iterative-implementation ~/.codex/skills/iterative-implementation

ln -s ~/claude-skills/iterative-plan-review ~/.claude/skills/iterative-plan-review
ln -s ~/claude-skills/iterative-plan-review ~/.codex/skills/iterative-plan-review
\
```
