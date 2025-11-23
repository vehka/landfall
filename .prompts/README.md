# Norns Development Prompt Library

This directory contains modular prompts for AI-assisted Norns script development. The prompts are organized by task type and domain, designed to be loaded conditionally based on context.

## Directory Structure

```
.prompts/
├── tasks/                 # Development task workflows
│   ├── create.md         # Building new scripts from scratch
│   ├── create-mod.md     # Building system-level mods
│   ├── understand.md     # Analyzing existing scripts/mods
│   ├── enhance.md        # Adding features to scripts/mods
│   └── bugfix.md         # Debugging and fixing issues
├── references/           # API and pattern documentation
│   ├── api-core.md       # Core Norns API reference
│   └── patterns.md       # Common script code patterns
├── hardware/             # Hardware integration guides
│   ├── grid.md           # Monome Grid integration
│   ├── arc.md            # Monome Arc integration
│   └── midi.md           # MIDI controller integration
├── engines/              # Synthesis and engine development
│   └── supercollider-basics.md  # SuperCollider engine guide
└── mods/                 # System mod development
    ├── create-mod.md     # Full mod creation workflow
    └── patterns.md       # Common mod patterns (8 patterns)
```

## Quick Start

1. **Task Detection**: Identify whether the user is asking about:
   - **Create Script**: Building a new standalone script
   - **Create Mod**: Building a system-level modification
   - **Understand**: Analyzing an existing script or mod
   - **Enhance**: Adding features to a script or mod
   - **Bugfix**: Fixing problems in a script or mod

2. **Load Relevant Prompts**:
   - **For Scripts**:
     - Start with `tasks/create.md` (or understand/enhance/bugfix)
     - Add `hardware/` guides if grid/arc/MIDI is involved
     - Add `engines/supercollider-basics.md` if custom synthesis needed
     - Reference `references/patterns.md` and `api-core.md`

   - **For Mods**:
     - Start with `tasks/create-mod.md` (or `mods/create-mod.md`)
     - Add `mods/patterns.md` for common mod patterns and documentation
     - Reference `references/patterns.md` for general Lua patterns

3. **Provide Targeted Guidance**:
   - Use specific code examples from appropriate patterns file
   - Match the complexity level
   - Link to relevant external resources
   - For scripts: Test on real hardware constraints
   - For mods: Test with multiple mods enabled, check hook ordering

## Prompt Sizes

All prompts are designed to be reasonably sized for efficient token usage:
- Task workflows: 2-4 KB each (5 files)
- References: 3-6 KB each (2 files)
- Hardware guides: 4-7 KB each (3 files)
- Engine docs: 5-8 KB each (1 file)
- Mod development: 4-8 KB each (2 files)

Total library: ~95 KB of prompt content, easily assembled into working sessions without exceeding reasonable token budgets.

**Typical Session Loading**:
- Script creation: ~15-20 KB (core + references + hardware if needed)
- Mod creation: ~10-14 KB (core + mod patterns + documentation)
- Debugging: ~8-12 KB (bugfix workflow + patterns)
- Quick lookup: ~3-5 KB (specific reference file)

## Integration Points

These prompts work with:
- **Norns API**: https://monome.org/docs/norns/api/
- **Norns Studies**: https://monome.org/docs/norns/studies/
- **Example Scripts**: https://github.com/monome/dust
- **Norns Source Code**: Available at `../norns/` relative to this repo

## Notes

- **Script Development**: Lua code with examples for hardware integration (Grid, Arc, MIDI)
- **Mod Development**: Lua code focusing on hooks, system integration, and API exposure
- **Engine Development**: SuperCollider code for synthesis
- **Performance**: All tips account for Norns hardware constraints and mod ordering
- **Skill Levels**: Prompts designed for beginner through advanced developers
- **Separation**: Scripts and mods have distinct development workflows and constraints

## Key Distinctions: Scripts vs Mods

| Aspect | Scripts | Mods |
|--------|---------|------|
| **Location** | `~/dust/code/` | `~/dust/mods/` |
| **Lifecycle** | User-launched applications | Load at system startup |
| **Scope** | Standalone functionality | System-wide modifications |
| **Hooks** | None (scripts don't use hooks) | Multiple hooks for lifecycle integration |
| **User Control** | Always enabled when loaded | Must be explicitly enabled in SYSTEM > MODS |
| **Default State** | N/A | Disabled by default (safety) |
| **File Required** | Main `.lua` file | `lib/mod.lua` only |
| **Use Cases** | Instruments, tools, effects | Utilities, system enhancements, libraries |

See the main AGENTS.md file for the complete conditional loading system.
