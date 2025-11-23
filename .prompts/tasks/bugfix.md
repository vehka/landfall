# Task: Debug and Fix Issues in a Norns Script

## Objective
Diagnose and resolve bugs in Norns scripts using systematic debugging approaches.

## Workflow

### 1. Issue Characterization

**Gather Information**:
- What is the observed behavior?
- When does the issue occur? (specific conditions, repeatable?)
- What error messages appear? (check `.log` files in script directory)
- Did it work before? (if so, what changed?)

**Question Categories**:
- **Functionality**: Does a feature not work as intended?
- **Performance**: Are there dropouts, glitches, or slowdowns?
- **Stability**: Does the script crash or hang?
- **Audio**: Are there unexpected sounds, clicks, or silence?
- **UI**: Is the display incorrect or unresponsive?

### 2. Debug Logging Strategy

Since Norns has limited console visibility, use logging:

**Print-based Debugging**:
```lua
-- Add debug prints to understand execution flow
print("DEBUG: init() called")
print("DEBUG: parameter value = " .. tostring(param_val))
```

**Error Detection**:
```lua
-- Check for common errors
if engine == nil then print("ERROR: engine not loaded") end
if params == nil then print("ERROR: params system failed") end
```

**Conditional Logging** (for performance):
```lua
local DEBUG = true
if DEBUG then print("DEBUG: " .. message) end
```

### 3. Common Bug Categories and Solutions

#### Initialization Issues
**Symptoms**: Script fails to load, no sound, missing controls

**Investigation**:
1. Check `init()` is defined and complete
2. Verify engine name matches available engines
3. Check parameter definitions for syntax errors
4. Ensure all library imports exist

**Fixes**:
- Verify engine loads: `engine.name = "EngineNameHere"`
- Check parameter syntax: `params:add_number()`
- Ensure libraries are in `lib/` or built-in

#### Parameter Issues
**Symptoms**: Controls don't work, parameters out of range, values reset

**Investigation**:
1. Verify parameter min/max/default are sensible
2. Check `enc()` function for parameter updating
3. Look for type mismatches (number vs string)
4. Check parameter group definitions

**Fixes**:
```lua
-- Proper parameter definition
params:add_number("param_id", "display name", 0, 1, 10)
params:set_action("param_id", function(x) engine.param(x) end)

-- Proper encoder handling
function enc(n, d)
  if n == 1 then params:delta("param_id", d) end
  redraw()
end
```

#### Audio Issues
**Symptoms**: No sound, clicks/pops, audio cuts out, CPU maxing

**Investigation**:
1. Check engine output is enabled
2. Verify gate/trigger timing
3. Look for audio stack overflows
4. Monitor CPU usage in menu

**Fixes**:
- Ensure gate commands reach engine: `engine.gate(1); engine.gate(0)`
- Check timing with `metro`: don't trigger every frame
- Reduce per-frame processing in `redraw()`
- Use appropriate update rates for different tasks

#### Screen Rendering Issues
**Symptoms**: Text not visible, garbled display, UI lag

**Investigation**:
1. Check `redraw()` function exists and is called
2. Verify `screen.update()` is called last
3. Look for screen coordinate overflow
4. Check for excessive redraw frequency

**Fixes**:
```lua
function redraw()
  screen.clear()
  -- drawing calls
  screen.move(x, y)
  screen.text("message")
  screen.update()  -- always last
end
```

#### Performance Issues
**Symptoms**: Glitches, dropouts, stuttering, high CPU

**Investigation**:
1. Check redraw rate (should be ~30-60 Hz, not every frame)
2. Look for tight loops or blocking operations
3. Profile critical sections with timing
4. Check memory allocations in hot paths

**Fixes**:
```lua
-- Use metro instead of tight loops
local redraw_metro = metro.init(function() redraw() end, 1/30)
redraw_metro:start()

-- Avoid allocations in frequently-called functions
local reusable_table = {}  -- allocate once
```

#### State Management Issues
**Symptoms**: Values reset unexpectedly, parameters don't persist, crashes on save

**Investigation**:
1. Check parameter persistence is enabled
2. Verify file I/O operations complete
3. Look for corrupted state files
4. Check cleanup() properly handles resources

**Fixes**:
```lua
-- Ensure parameters persist
params:write()  -- manual save
params:read()   -- manual load

-- Proper cleanup
function cleanup()
  if metro_obj then metro_obj:stop() end
  if engine then engine.gate(0) end
  params:write()
end
```

### 4. Testing and Verification

**After Each Fix**:
1. Reload script (`SYSTEM > RESTART`)
2. Test the specific issue
3. Test nearby functionality
4. Run through a typical usage scenario
5. Leave running for 5 minutes to check stability

**Regression Testing**:
- Verify no previously-working features broke
- Test parameter ranges and edge cases
- Check CPU usage hasn't increased
- Ensure cleanup still works properly

## Debugging Checklist

- [ ] Issue is clearly characterized and reproducible
- [ ] All log messages and error outputs reviewed
- [ ] Initialization code verified
- [ ] Parameter definitions checked for syntax
- [ ] Audio routing and gating examined
- [ ] Screen rendering verified
- [ ] Performance profiled if relevant
- [ ] Fix tested thoroughly
- [ ] No regressions in other features
- [ ] Cleanup still works properly

## Reference Resources

- Check Norns script logs in `~/.norns/` directory
- Use `.prompts/references/api.md` for API documentation
- Review `.prompts/references/patterns.md` for correct patterns
- Check example scripts at https://github.com/monome/dust for reference implementations
- Norns source code in `../norns/` for deep dives

## Advanced Debugging

**Interacting with Norns Shell**:
- Use telnet to Norns for direct Lua execution
- Call functions and check return values in real-time
- This is useful for testing fixes without full reload

**SuperCollider Debugging** (for engine issues):
- Test engine commands directly in SuperCollider
- Verify synthesis code produces expected output
- Check parameter ranges and control rates
