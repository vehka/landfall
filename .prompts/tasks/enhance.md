# Task: Enhance an Existing norns Script

## Objective
Add new features or improve existing functionality in a norns script while maintaining stability and code quality.

## Workflow

### 1. Feature Definition and Planning
- **Understand Current Scope**: Review the existing script using the "understand" workflow
- **Define New Feature**: Be specific about:
  - What functionality to add
  - How it integrates with existing features
  - UI/control requirements for the new feature
  - Any new parameters or state needed

- **Impact Assessment**:
  - What existing code must change?
  - Will new libraries or engines be needed?
  - Are there performance implications?
  - Does screen layout need redesign?

### 2. Implementation Strategy

**Modular Approach**:
- Create changes in isolated sections where possible
- Write the feature with minimal impact on existing code
- Use temporary or feature-flag patterns if needed for testing

**Code Integration Points**:
- Parameter setup: Where and how to add new parameters
- Control handling: Map new inputs if needed
- Screen rendering: Update display without breaking existing layout
- Cleanup: Ensure new resources are properly released

### 3. Development Process

1. **Backup/Branch**: Work in a git branch or save original
2. **Add Feature Core**: Implement the basic functionality
3. **Integrate Controls**: Add parameter and input handling
4. **Update UI**: Integrate into screen rendering
5. **Test Thoroughly**: Verify functionality and edge cases
6. **Optimize**: Address performance if needed

### 4. Testing Strategy

**Functional Testing**:
- Verify new feature works as expected
- Test edge cases and boundary conditions
- Confirm parameter ranges are sensible
- Check for unexpected interactions with existing features

**Performance Testing**:
- Monitor CPU usage (watch for glitches or dropouts)
- Check memory stability over time
- Verify screen refresh remains smooth

**Integration Testing**:
- Ensure feature works with all existing features
- Test parameter persistence (save/load)
- Verify cleanup doesn't leak resources

## Feature Categories and Patterns

### UI Enhancements
- Adding parameters or menu items
- Reorganizing screen layout
- Adding visual feedback (animations, indicators)
- Implementing nested menus or pages

**Pattern**: Add a new parameter group and redraw section
```lua
-- In init():
params:add_group("new feature", 3)
params:add_number("param1", "description", min, default, max, units)

-- In redraw():
screen.move(x, y)
screen.text(params:get("param1"))
```

### Audio Enhancements
- Adding effects or processing
- Expanding synthesis parameters
- Adding new audio sources
- Implementing mixer or routing

**Considerations**:
- Changes to audio engines require SuperCollider knowledge
- Parameter range matching between Lua and SC
- Real-time safety of parameter changes
- CPU headroom for additional processing

### Control Integration
- Grid or arc support
- MIDI controller integration
- Additional encoder/key remapping
- Improved gesture recognition

**Pattern**: Grid button handling
```lua
function grid_key(x, y, z)
  if z == 1 then
    -- button press logic
  else
    -- button release logic
  end
  redraw()
end
```

### Sequencer/Data Features
- Pattern storage and recall
- Undo/redo functionality
- File I/O for presets
- Advanced pattern generation

**Considerations**:
- Memory constraints for large data structures
- File system organization
- State serialization format

## Code Quality Checklist

- [ ] Feature is well-integrated with existing code
- [ ] No undefined variables or scope issues
- [ ] All new parameters have sensible defaults and ranges
- [ ] New UI elements don't clutter the display
- [ ] Performance impact is acceptable
- [ ] Feature works at various script states
- [ ] Cleanup handles new resources
- [ ] Changes don't break existing features

## Reference Resources

- Use `.prompts/references/api.md` for API lookups
- Use `.prompts/references/patterns.md` for common enhancement patterns
- Check `.prompts/hardware/` for grid/arc/MIDI patterns
- Review existing scripts at https://github.com/monome/dust for inspiration
