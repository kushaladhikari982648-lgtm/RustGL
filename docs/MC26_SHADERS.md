# Minecraft 26.3 core shaders on GLES

What changed in 26.3, which of it this layer can carry onto a GLES driver, which of it it
cannot, and how to capture the shader that fails on your device.

## What 26.3 changed

26.3 moved rendering onto `RenderPipelines` / renderpearl, and the core shaders lean harder on
GL 4.x semantics than any version before it. In practice that means three things for a
translation layer:

| Change | Consequence on GLES |
| --- | --- |
| Core shaders carry desktop GLSL (`#version 450 core` era) | everything goes through the desktop → GLSL ES translator, nothing is ES already |
| More of the frame is routed through pipelines that were written against desktop drivers | desktop drivers accept constructs GLSL ES rejects outright, and the rejection kills the whole program |
| A failing core shader aborts resource loading | one bad shader is a black screen or a stall, not a visible glitch |

The failure mode is not subtle: the program fails to compile, the pipeline is dropped, and
whatever that pipeline drew is simply missing.

## Output arrays indexed at run time — fixed

26.3's OIT (order-independent transparency) shader declares its coefficient attachments as one
array and picks the attachment while it runs:

```glsl
layout(location = 0) out vec4 coeff[OIT_COEFF_ATTACHMENT_COUNT][4];
...
coeff[attachmentIndex][i] = ...;
```

Desktop GL compiles that. GLSL ES does not, for two separate reasons, and the errors are
different:

```
S0015: Outputs declared as arrays may only be indexed by a constant integral expression
error: fragment shader output cannot have an array of arrays
```

The first is the run-time index. The second is the shape: an array of arrays cannot be a
fragment output at all, whatever indexes it. Adreno and Mali report the first; Mesa reports the
second for the shape, and the same message with `vertex shader` for a vertex output. Every
pipeline routed through OIT — text, item, entity, clouds, block, particle — dies with it, which
is the 26.3 black screen.

This layer rewrites it. The array keeps its name and its values but stops being an output:

```glsl
vec4 coeff[2][4];                              // an ordinary global, index it as you like
layout(location = 0) out vec4 rustgl_out_coeff_0;   // one scalar output per element,
layout(location = 1) out vec4 rustgl_out_coeff_1;   // holding the locations the array had
...
// at the end of main(), and before any early return:
rustgl_out_coeff_0 = coeff[0][0]; rustgl_out_coeff_1 = coeff[0][1]; ...
```

A run-time index into an ordinary array is legal in GLSL ES, and every write to an output now
carries a constant index, which is exactly what the driver could not prove before. Locations
are preserved: OpenGL numbers the elements of an output array consecutively with the last index
varying fastest, and that is the order the scalar outputs take.

It fires when either trigger is present, and otherwise leaves the shader alone:

* the array is indexed by anything that is not a literal, or
* the declaration is an array of arrays.

It does not fire on a plain one-dimensional output array indexed with constants, which GLSL ES
accepts and which therefore needs no rewriting.

### What it deliberately does not do

**Vertex stages are left alone**, even though GLSL ES rejects an array of arrays there too.
The substituted outputs are new names, and a vertex output is matched by name by the fragment
stage: renaming one side silently breaks the link. Fixing a vertex stage needs a pass that can
rename both stages of the same program at once, and this layer sees one shader at a time.

**A computed size or location is left alone.** If the declaration's dimensions or its
`layout(location = ...)` cannot be read as an integer — a macro that expands to an expression
rather than a number, say — the shader is passed through untouched rather than half-rewritten.
The same applies to more than 32 elements, a `main()` it cannot find, and a generated name that
the shader already uses.

Every one of those is the behaviour of the build before this one: a shader that still fails to
compile tells you so in the log instead of rendering something wrong.

## Triage: capture the shader that fails

Set `RENDERER_DUMP_SHADER_DIR` and every shader the game hands over is written twice:

```
RENDERER_DUMP_SHADER_DIR=/sdcard/rustgl
```

| File | Contents |
| --- | --- |
| `<id>.src` | what the game gave us, before translation |
| `<id>.translated` | what we handed the driver |

A compile failure is logged with the same id:

```
[GLCompat] compile failed shader=22: 0:14(1): error: fragment shader output cannot have an array of arrays
```

So the pair to send is the log line and the two files for that id. `RENDERER_TRACE_GL=1` adds
the GL calls around it, and `RENDERER_GL_VERSION=33|43|45` forces a different claimed version
if you suspect the driver is being asked for more than it has.

## Verification

The GLES smoke test in CI compiles every shader fixture on a real driver (Mesa, surfaceless
EGL). The three 26.3 fixtures are:

| Fixture | Shape | Expected |
| --- | --- | --- |
| `OIT_ADD_TRANSMITTANCE_FSH_26_3` | array of arrays, run-time index | rewritten, compiles |
| `OIT_ARRAY_OF_ARRAYS_FSH_26_3` | array of arrays, constant index | rewritten, compiles |
| `OIT_CONSTANT_INDEX_FSH_26_3` | one dimension, constant index | untouched, compiles |

That is a driver compiling the rewritten output, not a text assertion. It is one driver, on one
implementation of GLSL ES; Adreno and Mali enforce the same rules with different words, and the
only way to confirm those is on the device.
