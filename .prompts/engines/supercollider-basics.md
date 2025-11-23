# SuperCollider Engine Development for Norns

Norns can load custom SuperCollider (SC) synthesis engines. Engines are defined in `.sc` files and controlled via Lua wrapper scripts.

## File Structure

```
~/dust/code/my_script/
├── my_script.lua          (main Lua script)
└── engine/
    ├── MyEngine.sc        (SuperCollider synth definition)
    └── MyEngine.lua       (Lua wrapper, optional)
```

## Basic SuperCollider Synth Definition

SuperCollider synths are sound-generating processes defined in `.sc` files:

```supercollider
// MyEngine.sc

Engine.register(\MyEngine, trickle: false);

(
SynthDef(\MyEngine_default, {
  | out=0, gate=1, hz=440, amp=0.1 |

  var sig, env;

  // Create envelope
  env = Env.adsr(0.01, 0.1, 0.7, 0.5);
  env = EnvGen.kr(env, gate, doneAction: 2);

  // Create oscillator
  sig = Saw.ar(hz) * amp * env;

  // Output to audio bus
  Out.ar(out, sig!2);  // !2 makes stereo (duplicate signal)
}).add;
);
```

### Key Components

**Gate Parameter**: Triggers envelope when set to 1, releases when 0
- `doneAction: 2` kills the synth when envelope finishes
- Allows polyphonic overlapping notes

**Envelope (`Env.adsr`)**: Standard ADSR (Attack, Decay, Sustain, Release)
- `Env.adsr(attack, decay, sustain, release)`
- `EnvGen.kr()` generates control-rate envelope signal
- Gate controls when envelope starts/stops

**Oscillators**:
- `Saw.ar()`: Sawtooth wave
- `SinOsc.ar()`: Sine wave
- `Pulse.ar()`: Pulse/square wave
- `LFSaw.ar()`: Low-frequency sawtooth (for LFO)

**Filters**:
- `LPF.ar(signal, freq)`: Low-pass filter
- `HPF.ar(signal, freq)`: High-pass filter
- `BPF.ar(signal, freq, rq)`: Band-pass filter

**Output**: `Out.ar(out, signal)` sends to output bus
- First argument is the bus index
- `!2` makes mono signal stereo

## Lua Wrapper and Control

Create a corresponding `.lua` file in the `engine/` folder to expose parameters:

```lua
-- engine/MyEngine.lua

local ControlSpec = require "controlspec"
local Formatters = require "formatters"

local engine = {}

-- Define which synth to use
engine.name = "MyEngine"

-- Parameter specifications (controlled from Norns)
local specs = {
  hz = ControlSpec.FREQ,  -- 20 to 20000 Hz
  amp = ControlSpec.AMP,   -- -inf to 6 dB
  gate = ControlSpec.new(0, 1, "lin", 0, 0),
}

-- Define parameter handlers
function engine.hz(hz)
  _norns.engine.set_param("hz", hz)
end

function engine.amp(amp)
  _norns.engine.set_param("amp", amp)
end

function engine.gate(gate)
  _norns.engine.set_param("gate", gate)
end

return engine
```

## Common Engine Patterns

### Polyphonic Synthesizer

```supercollider
Engine.register(\PolySynth, trickle: false);

(
SynthDef(\PolySynth_default, {
  | out=0, gate=1, hz=440, amp=0.1, attack=0.01, decay=0.1, sustain=0.7, release=0.5 |

  var sig, env, filter_env;

  // Amplitude envelope
  env = Env.adsr(attack, decay, sustain, release);
  env = EnvGen.kr(env, gate, doneAction: 2);

  // Filter envelope
  filter_env = Env.adsr(attack, decay, sustain, release);
  filter_env = EnvGen.kr(filter_env, gate);

  // Oscillator
  sig = Saw.ar(hz) * amp * env;

  // Filter sweep
  sig = LPF.ar(sig, 1000 + (filter_env * 4000));

  Out.ar(out, sig!2);
}).add;
);
```

### FM Synthesis

```supercollider
Engine.register(\FM, trickle: false);

(
SynthDef(\FM_default, {
  | out=0, gate=1, hz=440, amp=0.1, ratio=2, index=1 |

  var sig, env, mod;

  env = Env.adsr(0.01, 0.1, 0.7, 0.5);
  env = EnvGen.kr(env, gate, doneAction: 2);

  // Modulator
  mod = SinOsc.ar(hz * ratio, mul: hz * index);

  // Carrier
  sig = SinOsc.ar(hz + mod) * amp * env;

  Out.ar(out, sig!2);
}).add;
);
```

### Granular Synthesis

```supercollider
Engine.register(\Granular, trickle: false);

(
SynthDef(\Granular_default, {
  | out=0, gate=1, hz=10, amp=0.1, grain_size=0.1 |

  var sig, env;

  env = Env.new([0, amp, amp, 0], [0.05, 1, 0.5], 'sine');
  env = EnvGen.kr(env, gate, doneAction: 2);

  // Trigger grains
  sig = TGrains.ar(2, Impulse.ar(hz), Buffer.read(0), 1, 0);

  Out.ar(out, sig * env);
}).add;
);
```

## Norns Integration

### Loading in Lua

```lua
-- In my_script.lua init()
engine.name = "MyEngine"

-- Control parameters
function enc(n, d)
  if n == 1 then
    engine.hz(params:get("frequency"))
  elseif n == 2 then
    engine.amp(params:get("amplitude"))
  end
end

-- Gate on key press
function key(n, z)
  if z == 1 then
    engine.gate(1)
  else
    engine.gate(0)
  end
end
```

### Testing Engines Locally

Engines listen on `127.0.0.1:57110` (default SuperCollider port).

You can test engines in SuperCollider directly:

```supercollider
// Start SuperCollider server
s.boot;

// Load your synth definition
(
SynthDef(\test, {
  | out=0, gate=1, hz=440, amp=0.1 |
  var sig, env;
  env = Env.adsr(0.01, 0.1, 0.7, 0.5);
  env = EnvGen.kr(env, gate, doneAction: 2);
  sig = Saw.ar(hz) * amp * env;
  Out.ar(out, sig!2);
}).add;
);

// Create a synth
x = Synth(\test, [\hz, 440, \amp, 0.2]);

// Gate off when done
x.set(\gate, 0);
```

## Advanced Patterns

### Parameter Ranges with ControlSpec

```lua
local ControlSpec = require "controlspec"

-- Standard specs
ControlSpec.UNIPOLAR       -- 0 to 1
ControlSpec.BIPOLAR        -- -1 to 1
ControlSpec.FREQ           -- 20 to 20000 Hz
ControlSpec.AMP            -- -inf to 6 dB
ControlSpec.MS             -- 0.001 to 60 seconds

-- Custom spec
ControlSpec.new(min, max, warp, step, default)
-- warp: "lin" (linear), "exp" (exponential), "db", "sin"
```

### Multiple Synth Voices

```supercollider
// Define multiple synth variations
(
SynthDef(\MyEngine_default, {
  | out=0, gate=1, hz=440, amp=0.1 |
  // Default version
}).add;

SynthDef(\MyEngine_alt, {
  | out=0, gate=1, hz=440, amp=0.1 |
  // Alternative version
}).add;
);
```

## Common Issues

**Engine won't load**: Check file names and paths
- Engine file: `~/dust/code/script/engine/EngineName.sc`
- Lua wrapper: `~/dust/code/script/engine/EngineName.lua` (optional)
- Script references: `engine.name = "EngineName"`

**No sound**: Verify `Out.ar()` is called with correct bus
- Default: `Out.ar(out, signal)` where `out` is parameter

**Parameters not changing**: Ensure Lua wrapper implements control functions
- Must call `_norns.engine.set_param()` or equivalent

**Clicks/pops**: Use `doneAction: 2` for cleanup and ensure proper gating

## Reference

- SuperCollider Language: https://doc.sccode.org/
- Norns Engine Development: https://monome.org/docs/norns/studies/skilled-labor/
- Example engines: https://github.com/monome/norns/tree/main/lua/core/engine
- Complete Norns SC integration: https://github.com/monome/norns/tree/main/crone
