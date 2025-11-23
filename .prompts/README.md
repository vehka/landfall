# Norns Development Prompt Library

This directory contains modular prompts for AI-assisted Norns script development. The prompts are organized by task type and domain, designed to be loaded conditionally based on context.

## Directory Structure

```
.prompts/
├── tasks/                 # Development task workflows
│   ├── create.md         # Building new scripts from scratch
│   ├── understand.md     # Analyzing existing scripts
│   ├── enhance.md        # Adding features to scripts
│   └── bugfix.md         # Debugging and fixing issues
├── references/           # API and pattern documentation
│   ├── api-core.md       # Core Norns API reference
│   └── patterns.md       # Common code patterns and examples
├── hardware/             # Hardware integration guides
│   ├── grid.md           # Monome Grid integration
│   ├── arc.md            # Monome Arc integration
│   └── midi.md           # MIDI controller integration
└── engines/              # Synthesis and engine development
    └── supercollider-basics.md  # SuperCollider engine guide
```

## Quick Start

1. **Task Detection**: Identify whether the user is asking about:
   - **Create**: Building a new script
   - **Understand**: Analyzing an existing script
   - **Enhance**: Adding features
   - **Bugfix**: Fixing problems

2. **Load Relevant Prompts**:
   - Start with the task-specific workflow (required)
   - Add hardware guides if grid/arc/MIDI is involved
   - Add engine docs if custom synthesis is involved
   - Reference patterns.md for code examples
   - Reference api-core.md for API documentation

3. **Provide Targeted Guidance**:
   - Use specific code examples
   - Match the complexity level
   - Link to relevant external resources
   - Test on real hardware constraints

## Prompt Sizes

All prompts are designed to be reasonably sized for efficient token usage:
- Task workflows: 2-4 KB each
- References: 3-6 KB each
- Hardware guides: 4-7 KB each
- Engine docs: 5-8 KB each

Total library: ~65-75 KB of prompt content, easily assembled into working sessions without exceeding reasonable token budgets.

## Integration Points

These prompts work with:
- **Norns API**: https://monome.org/docs/norns/api/
- **Norns Studies**: https://monome.org/docs/norns/studies/
- **Example Scripts**: https://github.com/monome/dust
- **Norns Source Code**: Available at `../norns/` relative to this repo

## Notes

- All code examples use Lua (for scripts) or SuperCollider (for engines)
- Hardware examples assume standard controller setup (Grid, Arc, MIDI)
- Performance tips account for Norns hardware constraints
- All prompts are designed for both beginner and advanced developers

See the main AGENTS.md file for the complete conditional loading system.
