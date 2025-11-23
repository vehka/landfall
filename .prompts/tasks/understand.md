# Task: Understand an Existing norns Script or Engine

## Objective
Analyze and explain existing scripts or SuperCollider engines to understand their implementation and design.

## Workflow

### 1. Initial Script Survey
Examine the script structure:
- Read the main `.lua` file and identify:
  - What lifecycle functions are implemented (`init`, `key`, `enc`, `redraw`, `cleanup`)
  - What parameters are defined
  - What libraries are imported
  - What audio sources or engines are used

- Look for optional files:
  - `lib/` directory with utility modules
  - `engine/` directory with custom SuperCollider code
  - `data/` directory with saved state

### 2. Understand Control Flow

**Initialization (`init()`)**:
- Parameter setup and ranges
- Engine initialization
- UI state initialization
- Metro or clock setup for timing

**Input Handling (`key()` and `enc()`)**:
- Map user inputs to parameter changes or actions
- Look for state machines or mode switching
- Identify UI interaction patterns

**Screen Rendering (`redraw()`)**:
- Understand layout and visual hierarchy
- Note which parameters are displayed
- Check for animated elements or status indicators

**Cleanup (`cleanup()`)**:
- Graceful shutdown of engines, metros, and resources

### 3. Audio Architecture Analysis

If script uses audio:
- **Built-in Engines**: Identify which engine (e.g., `PolyPerc`, `Babysynth`)
  - List what parameters the engine exposes
  - Check timing/trigger synchronization

- **Custom Engines**: If `.sc` files exist:
  - Examine SuperCollider synthesis code
  - Understand signal routing and modulation
  - Note any external input/output handling

### 4. Library and Dependency Analysis

- Check which norns libraries are used (`musicutil`, `lattice`, `sequins`, etc.)
- Understand how data structures are organized (arrays, nested tables, etc.)
- Look for custom abstractions or patterns

### 5. Create Summary Documentation

Generate documentation covering:
- **Purpose**: What does this script do?
- **Key Features**: Main functionality and capabilities
- **Control Interface**: How users interact with it
- **Audio Architecture**: What generates/processes sound
- **Parameter Map**: Which controls affect what
- **Dependencies**: Required libraries and engines
- **Design Patterns**: Notable techniques or approaches used

## Key Questions to Answer

1. What is the script's primary purpose and use case?
2. How is state managed (UI modes, sequencer position, etc.)?
3. What are the performance characteristics? (CPU, memory usage)
4. How does it handle edge cases (empty input, out-of-range values)?
5. What would be the easiest features to extend?

## Common Patterns to Look For

- **Parameter Groups**: Using the `group` parameter type for UI organization
- **Sequencer Pattern**: Using `sequins` or custom table iteration
- **Beat Synchronization**: Using `lattice` or `clock` for timing
- **Screen Layout**: Text positioning and visual organization
- **State Persistence**: How parameters are saved/loaded
- **Hardware Integration**: Grid/arc/MIDI patterns and conventions

## Reference Resources

- Use `.prompts/references/api.md` for understanding API calls
- Use `.prompts/references/patterns.md` for recognizing common code patterns
- Check norns source: `../norns/lua/` for built-in library implementations
- Review related studies at https://monome.org/docs/norns/studies/
