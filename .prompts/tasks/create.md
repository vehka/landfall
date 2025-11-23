# Task: Create a New Norns Script

## Objective
Build a new Norns script from scratch with proper structure and functionality.

## Workflow

### 1. Clarify Requirements
- What does the script do? (sequencer, effect processor, synthesizer, etc.)
- What is the primary interaction model? (encoders, grid, arc, MIDI)
- What audio sources? (built-in engines, hardware audio in, external synths)
- Estimated complexity level? (beginner/intermediate/advanced)

### 2. Initial Architecture
- Define main script structure with required lifecycle functions:
  - `init()`: Initialize parameters, engines, UI state
  - `key(n, z)`: Handle 3 buttons
  - `enc(n, d)`: Handle 3 encoders
  - `redraw()`: Render screen updates
  - `cleanup()`: Graceful shutdown

- Plan parameter categories (uses, portamento, volume, etc.)
- Identify required libraries (`musicutil`, `lattice`, `sequins`, etc.)

### 3. Modular Development
Start with minimal viable version:
1. Basic UI and screen rendering
2. Parameter controls via encoders
3. Audio source integration
4. Hardware controller support (grid/arc/MIDI if needed)
5. State management and saving

### 4. Testing Strategy
- Test each component in isolation
- Verify parameter ranges and edge cases
- Check screen rendering at various states
- Validate audio performance (no clicks, dropouts)

## Key Considerations

**Screen Layout**: 128x64 monospace display requires careful text placement
- Use `screen.clear()`, `screen.move()`, `screen.text()`, `screen.update()`
- Plan layout in advance to avoid clutter

**Parameter System**: Use paramset for persistent user settings
- Supports number, option, control, file, group, and trigger types
- Automatically saves/loads via PARAMETERS menu

**Timing**: Use `metro` for precise timing rather than polling loops
- Create metros for UI refreshes (~30-60 Hz), audio updates, sequencer steps

**Memory**: Norns has limited RAM; avoid storing large tables or frequent allocations
- Use local variables within functions where possible
- Profile memory usage if building complex scripts

## File Structure

```
~/dust/code/my_script/
├── my_script.lua       (main entry point)
├── lib/                (optional utilities)
│   ├── ui.lua
│   └── sequencer.lua
└── engine/             (optional SuperCollider engine)
    ├── my_engine.sc
    └── my_engine.lua
```

## Reference Resources

- Use `.prompts/references/api.md` for API lookups
- Use `.prompts/references/patterns.md` for common code patterns
- Check `.prompts/hardware/` guides for grid, arc, or MIDI integration
- Review studies at https://monome.org/docs/norns/studies/ for in-depth examples
