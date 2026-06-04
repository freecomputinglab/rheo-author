# rheo-author

A Claude Code skill for authoring, configuring, and building rheo projects.

## About

This skill enables Claude Code to work with rheo projects, including:
- Creating and editing `rheo.toml` configuration files
- Authoring `*.typ` content files
- Running rheo build and watch commands

## Installation

To install this skill in your local Claude Code environment:

```bash
# Clone the skill repository
git clone <repository-url> ~/.claude/skills/rheo-author

# Or copy the skill directory
cp -r /path/to/rheo-author ~/.claude/skills/rheo-author
```

The skill will be automatically available when you restart Claude Code.

## Usage

Once installed, you can use the skill by typing `/rheo-author` or by mentioning rheo-related tasks in your conversation with Claude Code.

## Requirements

- Claude Code CLI
- rheo (if building projects locally)
