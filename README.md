# A-Team (Agentic Team)

## Installation

### All Projects

```bash
claude install-skill https://github.com/ealves-pt/ateam-skill
```

### Project Scoped

> Change to the project directory before running the commands below.

```bash
mkdir -p .claude/skills/
git clone https://github.com/ealves-pt/ateam-skill /tmp/ateam-skill
cp -r /tmp/ateam-skill/ateam .claude/skills/
rm -rf /tmp/ateam-skill
```

## Updating

### All Projects

```bash
claude install-skill https://github.com/ealves-pt/ateam-skill
```

Running the install command again will update the skill to the latest version.

### Project Scoped

> Change to the project directory before running the commands below.

```bash
git clone https://github.com/ealves-pt/ateam-skill /tmp/ateam-skill
cp -r /tmp/ateam-skill/ateam .claude/skills/
rm -rf /tmp/ateam-skill
```
