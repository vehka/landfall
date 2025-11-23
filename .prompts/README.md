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
├── engines/              # SuperCollider engine development
│   ├── create-engine.md  # Engine creation workflow
│   ├── patterns.md       # 9 Working engine patterns
│   └── osc-testing.md    # Testing via OSC and debugging
├── mods/                 # System mod development
│   ├── create-mod.md     # Full mod creation workflow
│   └── patterns.md       # Common mod patterns (8 patterns)
└── softcut/              # Audio buffer looping and recording
    ├── workflow.md       # Softcut workflow and 4 patterns
    └── api-reference.md  # Complete softcut API reference
```

## Quick Start

1. **Task Detection**: Identify whether the user is asking about:
   - **Create Script**: Building a new standalone script in Lua
   - **Create Mod**: Building a system-level modification in Lua
   - **Create Engine**: Building a SuperCollider audio synth
   - **Understand**: Analyzing existing code
   - **Enhance**: Adding features
   - **Bugfix**: Fixing problems
   - **Audio Manipulation**: Looping, sampling, buffer processing → **Consider Softcut**

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

   - **For Engines**:
     - Start with `tasks/create-engine.md` (or `engines/create-engine.md`)
     - Add `engines/patterns.md` for 9 working synth patterns
     - Add `engines/osc-testing.md` for local testing and debugging
     - Reference SuperCollider documentation for deep dives

   - **For Audio Manipulation**:
     - Start with `softcut/workflow.md` for understanding softcut
     - Add `softcut/api-reference.md` for complete API documentation
     - Choose pattern from workflow.md (looper, tape, time-stretch, granular)
     - Reference Softcut studies at https://monome.org/docs/norns/softcut/

3. **Provide Targeted Guidance**:
   - Use specific code examples from appropriate patterns file
   - Match the complexity level
   - Link to relevant external resources
   - For scripts: Test on real hardware constraints
   - For mods: Test with multiple mods enabled, check hook ordering
   - For engines: Test locally in SuperCollider IDE, then on Norns device

## Prompt Sizes

All prompts are designed to be reasonably sized for efficient token usage:
- Task workflows: 2-4 KB each (5 files)
- References: 3-6 KB each (2 files)
- Hardware guides: 4-7 KB each (3 files)
- Engine development: 6-10 KB each (3 files)
- Mod development: 4-8 KB each (2 files)
- Softcut documentation: 5-8 KB each (2 files)

Total library: ~160 KB of prompt content, easily assembled into working sessions without exceeding reasonable token budgets.

**Typical Session Loading**:
- Script creation: ~15-20 KB (core + references + hardware if needed)
- Script with looping: ~18-25 KB (script + softcut + references)
- Mod creation: ~10-14 KB (core + mod patterns)
- Engine creation: ~18-25 KB (create-engine + patterns + osc-testing)
- Softcut looper: ~12-18 KB (softcut workflow + api-reference)
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
- **Engine Development**: SuperCollider code for synthesis with local testing via OSC
- **Softcut**: Audio buffer looping, sampling, recording with 4 patterns and complete API
- **Performance**: All tips account for Norns hardware constraints, mod ordering, CPU limits
- **Skill Levels**: Prompts designed for beginner through advanced developers
- **Separation**: Scripts, mods, engines, and softcut have distinct workflows and constraints
- **Testing**: Comprehensive testing strategies for each work type (local, device, OSC)

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
