# Task: Create a SuperCollider Engine for Norns

## Objective
Build a custom SuperCollider synthesis engine and integrate it with Norns scripts via Lua.

## Engine vs Script

Before starting, understand the separation of concerns:

- **Norns Script** (Lua): User interface, parameters, control logic running in Matron
- **SuperCollider Engine** (SC): Real-time audio synthesis and processing running in Crone (audio server)
- **Lua Wrapper** (Lua): Exposes engine commands to scripts via `engine.command()` calls

The Lua script and SC engine communicate via **OSC (Open Sound Control)** messages over localhost.

## Workflow

### 1. Plan Your Synth

Before writing code, design:

**Sound Source**:
- What oscillators/generators? (SinOsc, Saw, Pulse, noise, samples)
- What modulation? (LFO, envelopes, FM, AM)
- What polyphony? (monophonic, few voices, full polyphony)

**Processing**:
- Filters? (LPF, HPF, resonance)
- Effects? (delay, reverb, distortion)
- Envelopes? (ADSR, custom)

**Control Parameters**:
- What should users control? (freq, amp, filter cutoff, etc.)
- What are reasonable ranges?
- Should parameters be control-rate or audio-rate?

**Performance**:
- CPU usage expectations
- Will it run on Norns? (CPU is limited)
- Avoid heavy processing or sample playback if possible

### 2. File Structure

```
~/dust/code/my_script/
├── my_script.lua          (main Lua script)
├── lib/
│   └── my_engine.lua      (optional - parameter definitions)
└── engine/
    ├── MyEngine.sc        (SuperCollider synth definition)
    └── MyEngine.lua       (optional - Lua wrapper functions)
```

**Critical files**:
- `engine/MyEngine.sc`: SuperCollider SynthDef (defines the sound)
- Script loads engine via: `engine.name = "MyEngine"`

**Optional files**:
- `lib/my_engine.lua`: Parameter specs for convenient parameter creation
- `engine/MyEngine.lua`: Wrapper functions for convenient calls

### 3. Write SuperCollider SynthDef

Create `engine/MyEngine.sc`:

```supercollider
Engine.register(\MyEngine);

(
SynthDef(\MyEngine_default, {
  | out=0, gate=1, hz=440, amp=0.1 |

  var sig, env;

  // Envelope
  env = Env.adsr(0.01, 0.1, 0.7, 0.5);
  env = EnvGen.kr(env, gate, doneAction: 2);

  // Oscillator
  sig = Saw.ar(hz) * amp * env;

  // Output (stereo)
  Out.ar(out, sig!2);
}).add;
);
```

**Key Components**:

**Gate Parameter**: Controls note on/off
- Set to 1 to trigger envelope
- Set to 0 to release
- `doneAction: 2` kills synth when envelope finishes

**Env.adsr()**: Standard envelope
- `Env.adsr(attack, decay, sustain, release)`
- Attack/decay/release in seconds
- Sustain as fraction (0-1)

**Oscillators** (all audio-rate `.ar()`):
- `Saw.ar(freq)`: Sawtooth
- `SinOsc.ar(freq)`: Sine
- `Pulse.ar(freq, duty)`: Pulse/square
- `WhiteNoise.ar()`: White noise
- `LFSaw.ar(freq)`: Low-freq sawtooth (for LFO)

**Filters**:
- `LPF.ar(signal, freq)`: Low-pass
- `HPF.ar(signal, freq)`: High-pass
- `BPF.ar(signal, freq, rq)`: Band-pass

**Output**: `Out.ar(out, signal)`
- `!2` makes mono signal stereo (duplicate)
- Use `!1` for mono output

### 4. Create Parameter Specifications (Optional)

Create `lib/my_engine.lua` for convenient parameter setup:

```lua
-- lib/my_engine.lua
local ControlSpec = require "controlspec"

local MyEngine = {}

function MyEngine.add_params()
  params:add_number("hz", "Frequency", 20, 440, 20000)
  params:add_number("amp", "Amplitude", 0, 0.1, 1)

  params:set_action("hz", function(x)
    engine.hz(x)
  end)

  params:set_action("amp", function(x)
    engine.amp(x)
  end)
end

return MyEngine
```

### 5. Create Lua Wrapper (Optional)

Create `engine/MyEngine.lua` for convenience functions:

```lua
-- engine/MyEngine.lua
local MyEngine = {}

function MyEngine.hz(hz)
  _norns.engine.set_param("hz", hz)
end

function MyEngine.amp(amp)
  _norns.engine.set_param("amp", amp)
end

function MyEngine.gate(gate)
  _norns.engine.set_param("gate", gate)
end

return MyEngine
```

Actually, Norns auto-generates these, so this is optional.

### 6. Load in Script

In your main script:

```lua
function init()
  engine.name = "MyEngine"

  -- Set up parameters
  params:add_number("freq", "Frequency", 20, 440, 20000)
  params:set_action("freq", function(x)
    engine.hz(x)
  end)

  params:bang()
end

function key(n, z)
  if z == 1 then
    engine.hz(params:get("freq"))
    engine.gate(1)
  else
    engine.gate(0)
  end
  redraw()
end

function redraw()
  screen.clear()
  screen.move(2, 10)
  screen.text("Hz: " .. params:get("freq"))
  screen.update()
end
```

### 7. Test and Debug

**Local Testing** (without Norns device):
1. Start SuperCollider IDE
2. Boot the audio server
3. Load and test the SynthDef
4. Use OSC to send test messages (see `.prompts/engines/osc-testing.md`)

**On Norns Device**:
1. Place engine in script directory
2. Restart Norns
3. Load script
4. Check `~/.norns/matron.log` for errors
5. Use `engine.list_commands()` to verify parameters loaded

**Common Issues**:
- Synth won't play: Check gate parameter and `doneAction: 2`
- Parameters don't work: Verify parameter name matches in script
- No audio: Check `Out.ar()` uses correct output bus
- Crashes: Check Lua syntax in wrapper files

### 8. Advanced Features

**Multiple Synths**:
```supercollider
SynthDef(\MyEngine_lead, { ... }).add;
SynthDef(\MyEngine_pad, { ... }).add;
// Then route in control logic
```

**Real-time Parameter Modulation**:
```supercollider
| hz=440, lfo_rate=1, lfo_amount=100 |
var lfo = SinOsc.ar(lfo_rate) * lfo_amount;
var modded_hz = hz + lfo;
sig = Saw.ar(modded_hz);
```

**Polyphony with Voices**:
```supercollider
Engine.register(\MyEngine, trickle: false);
// Register multiple voice versions
```

**Audio Input Processing**:
```supercollider
var input = In.ar(0, 2);  // Read from audio input
sig = LPF.ar(input, cutoff);
Out.ar(out, sig);
```

## Common Patterns

See `.prompts/engines/patterns.md` for detailed examples:
- Simple monophonic synth
- Polyphonic synthesizer
- FM synthesis
- Granular synthesis
- Additive synthesis
- Audio effect processor

## Testing and Debugging

See `.prompts/engines/osc-testing.md` for:
- Setting up SuperCollider for local testing
- Sending OSC commands to test engines
- Debugging via print statements and logging
- Performance profiling and optimization

## Performance Tips

- **Control-rate parameters** (`Env.kr()`) are cheaper than audio-rate
- Avoid expensive operations like `FFT` on Norns
- Use `FreeSelf` instead of allocating/deallocating synths
- Profile with `s.queryAllNodes` to check CPU usage
- Keep sample loading minimal (limited disk I/O)

## Reference Resources

- SuperCollider Language: https://doc.sccode.org/
- Norns API (Engine): https://monome.org/docs/norns/api/#engine
- Norns Reference (Engine): https://monome.org/docs/norns/reference/engine/
- Example engines: ../norns/lua/engine/
- Crone source: ../norns/crone/src/
- OSC methods: ../norns/crone/osc-methods.txt

## Common Issues

**Engine won't load**:
- Check file names and paths
- Verify `Engine.register(\EngineNameHere)` in .sc file
- Check SuperCollider syntax in .sc file
- Look in `~/.norns/matron.log` for errors

**Parameters don't work**:
- Verify parameter name in `engine.param_name()` call
- Check control-rate vs audio-rate (`Env.kr()` vs `Env.ar()`)
- Ensure parameter is declared in SynthDef

**Audio issues**:
- Verify `Out.ar()` writes to correct output bus (default 0)
- Check `!2` for stereo output
- Use `doneAction: 2` for proper synth cleanup
- Monitor CPU in menu if dropouts occur

**OSC errors**:
- Verify SC server is running on default port 57110
- Check firewall isn't blocking localhost:57110
- Test with basic OSC commands first
