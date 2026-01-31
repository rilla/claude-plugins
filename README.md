# Rilla's Claude Code Plugins Marketplace

> **⚠️ EXPERIMENTAL / WORK IN PROGRESS ⚠️**
>
> This marketplace is an **exploration and experiment** in building Claude Code plugins. All plugins here are **NOT production-ready** and should be considered **tests/proof-of-concepts**.
>
> **Use at your own risk.** Feedback and contributions welcome!

---

A personal collection of Claude Code plugins for Rails development and beyond.

## 🔌 Available Plugins

### rails-best-practices (v0.1.0-experimental)

**⚠️ EXPERIMENTAL** - Comprehensive Rails development standards combining Evil Martians' conventions with 37signals' One Person Framework philosophy.

**Features:**
- Rich models, thin controllers
- State management decision tree (enums vs state records)
- Comprehensive naming conventions
- Anti-pattern guide with solutions
- RSpec + FactoryBot testing standards
- 12 detailed reference files

**Philosophy:** "Vanilla Rails is plenty" - write Rails code so simple that one person can understand the entire system.

[→ See full documentation](./plugins/rails-best-practices/PLUGIN_README.md)

---

### domain-modelling (v1.0.0)

Domain-Driven Design skill for modeling the problem space before writing code.

**Features:**
- Domain-first workflow: Understand → Model → Validate → Code
- Pragmatic inquiry (infers obvious, asks about complexity)
- DDD building blocks (Entities, Value Objects, Aggregates)
- Ubiquitous language establishment
- Model validation before coding
- Prevents common modeling mistakes

**Philosophy:** "The model is the understanding. Code is the expression. Get the understanding right first."

[→ See full documentation](./plugins/domain-modelling/README.md)

---

## 📦 Installation

### Option 1: Clone Entire Marketplace

```bash
# Clone to Claude Code plugins directory
git clone git@github.com:rilla/claude-plugins.git ~/.claude/plugins/rilla-marketplace

# All plugins will be available
```

### Option 2: Install Individual Plugin

```bash
# Clone and symlink specific plugin
git clone git@github.com:rilla/claude-plugins.git
ln -s $(pwd)/claude-plugins/plugins/rails-best-practices ~/.claude/plugins/rails-best-practices
```

## 🚧 Status

This marketplace is:
- ✋ **Experimental** - Testing ideas and patterns
- 🧪 **Proof of Concept** - Demonstrating approaches
- 📝 **Work in Progress** - Subject to major changes
- 🚧 **Not Battle-Tested** - Use with caution

## 🤝 Contributing

Found an issue? Have a suggestion? Want to add your own plugin?

1. Open an issue
2. Submit a PR
3. Share feedback

All contributions welcome!

## 📜 License

Each plugin has its own license (typically MIT). See individual plugin directories for details.

## 🙏 Credits

Plugins in this marketplace consolidate patterns and knowledge from:
- Evil Martians (@palkan and team)
- 37signals team (DHH, Jorge Manrubia, and team)
- Mario Alberto Chávez
- Rails community

All credit goes to the original authors.

---

**Current Plugins:** 2
**Last Updated:** 2025-01-31
**Status:** Experimental
