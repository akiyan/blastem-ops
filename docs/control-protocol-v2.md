# BlastEM acknowledged control protocol v2

Status: accepted for M2 implementation (2026-09-01)

Implementation status (akiyan branch, 2026-11-18): the commands below are
implemented and advertised through `hello` capabilities. Everything else in this
document (`resume`, `press_frames`, `reset`, `save_state`/`load_state`, v2 forms
of `pad`/`vramdump`, ROM hashes, 64-bit frame accounting) is the target contract,
not an advertised feature.

| Command | Params | Capability |
|---|---|---|
| `hello` | `client` | — |
| `status` | — | — |
| `save_flush` | — | `save_directory`, `sram_flush` |
| `snapshot` | `path` (absolute PNG path) | `cached_screenshot` |
| `pause`, `advance`, `interrupt` | `frames` 1..600 for `advance` | `frame_control` |
| `audio_start`, `audio_stop` | `path`, `max_seconds` 1..30 for `audio_start` | `host_audio_capture` |
| `speed` | `percent` 25..1000 | `speed_control` |
| `ram_read` | `address`, `size` 1..256 within FF0000..FFFFFF | `paused_work_ram` |
| `trace` | `frames` 1..36000, `path`, `watches` (1..16), optional `until` | `frame_trace` |
| `vdp_dump` | `path` (absolute KITVDMP1 file, written while stopped) | `paused_vdp_dump` |
| `z80_state`, `z80_read` | none; `address` 0..1FFF and `size` 1..256 for `z80_read` | `z80_observation` |
| `line_trace` | `frames` 1..600, `path`, optional `kinds` (`cram`, `vsram`, `reg`, `int`) | `z80_state`, `z80_read` | none; `address` 0..1FFF and `size` 1..256 for `z80_read` | `z80_observation` |
| `line_trace` |

`status` labels its stop location `instruction` (GDB), `frame_sync`,
`paused_loop`, or `event_poll`. The frame is the 32-bit VDP counter. Feature
details: [SRAM](features/sram.md), [paused screenshot](features/paused-screenshot.md),
[frame control](features/frame-control.md), [audio](features/audio-capture.md),
[speed and RAM](features/speed-and-ram.md), [frame trace](features/frame-trace.md), [VDP layout](features/vdp-layout.md), [line trace](features/line-trace.md), [Z80](features/z80.md).

The implemented parser also limits each object to 64 fields and requires IDs to
increase within a connection. It validates UTF-8, duplicate keys, integer-only
numbers, depth, string length, and top-level fields. Invalid/unusable request IDs
produce `id: null`; unsupported versions receive an error then disconnect.

## Persistent cartridge SRAM

`BLASTEM_SAVE_DIR` selects an absolute battery/state directory independently of
the ROM filename. With it unset, the existing directory policy remains in use.
MCP exposes this as optional `blastem_start.save_dir`, keeps the directory after
session cleanup, and holds an OS file lock until the process exits. A symlink
alias for the same directory cannot bypass the lock. Locks are released by the
OS after a wrapper crash. Direct unmanaged core instances do not use this lock.

`save_flush` takes empty params and returns `bytes`, `frame`, and `paused` only
after the SRAM copy is written, file-synced, atomically renamed, and its parent
directory synced. The emulated buffer is never byte-swapped in place. A failure
returns `io_error`; a directory-sync failure may follow a successful rename and
must still be treated as an uncertain write, not success. Temp files are removed
on handled failures. Process/power loss can leave an unused temp file.

This slice supports plain Mega Drive cartridge SRAM on POSIX. EEPROM, flash,
CD BRAM, 32X, and Windows flush are `unsupported`. GDB waits service the control
socket on POSIX without executing emulated instructions. `paused: true` during
GDB stops identifies an instruction stop, not necessarily a frame boundary.

MCP requires the save-directory capability before accepting a persistent
session, flushes before stopping such a session, and reports flush failure even
when the process has been stopped. V2 calls use short separate connections under
the existing client lock; legacy input/artifact streams are never mixed with JSON.
These operations have bounded deadlines. Long-operation cancellation below is
limited to `advance`/`interrupt`; see [frame control](features/frame-control.md).

The transport remains the `AF_UNIX` stream selected by `BLASTEM_CTRL_SOCK`.
V2 uses one UTF-8 JSON object plus LF per message. Lines are limited to 65,536
bytes, nesting to 16 levels, and strings to 4,096 bytes. Duplicate keys,
invalid UTF-8, non-integer numbers, and unknown top-level fields are errors.
One client and one outstanding request are allowed.

## Negotiation and replies

The first non-whitespace byte selects the connection mode: `{` selects v2;
anything else selects the unchanged legacy newline parser for that connection.
A v2 client first sends:

```json
{"id":1,"version":2,"command":"hello","params":{"client":"blastem-mcp"}}
```

The reply reports `server`, `protocol`, numeric limits, and capabilities chosen
from `frame_control`, `frame_input`, `reset`, `state`, and `artifacts`.
Unsupported versions receive `unsupported_version` before disconnect. Clients
gate optional commands on capabilities, not the version alone.

IDs are unique unsigned integers in 1..2^53-1 and are echoed. Every accepted
request gets exactly one ordered reply:

```json
{"id":2,"version":2,"ok":true,"result":{"frame":1234,"paused":true}}
{"id":3,"version":2,"ok":false,"error":{"code":"invalid_params","message":"frames must be 1..1000000"}}
```

Stable error codes are `invalid_request`, `invalid_params`, `unknown_command`,
`unsupported`, `invalid_state`, `not_found`, `io_error`, `busy`, and
`internal`. Diagnostic messages are not API. State-changing success replies
include the absolute unsigned 64-bit VDP `frame` and `paused` state.

## Boundary semantics

A boundary is immediately after rendering a complete frame and before CPU work
for the next one. Requests execute on the emulation thread.

- `status`: return frame, pause state, system, ROM SHA-256, and capabilities.
- `pause`: stop at the next boundary; already paused is a no-op.
- `resume`: resume after replying; already running is a no-op.
- `advance {"frames":N}`: while paused, run exactly N frames and pause again;
  N is 1..1,000,000.
- `press_frames {"pad":P,"button":B,"frames":N}`: while paused, press at the
  current boundary, advance N frames, release at the final boundary, and remain
  paused. Release is guaranteed on failure. P is 1 or 2 and B is a legacy
  button name.
- `reset {"kind":"soft"|"hard"}`: reset at a boundary, clear injected input,
  set frame to zero, and preserve the prior paused/running state. Hard reset
  reconstructs the system without changing ROM.
- `save_state {"name":S}`: while paused, atomically save complete deterministic
  machine state, frame, and injected input.
- `load_state {"name":S}`: while paused, atomically validate and restore it,
  then remain paused.
- `pad`, `screenshot`, and `vramdump`: acknowledged v2 forms; artifact replies
  identify the boundary frame and atomic-write result.

## State and path safety

`BLASTEM_STATE_DIR` names a private state directory. State commands never
accept paths. Names are 1..64 ASCII characters matching
`[A-Za-z0-9][A-Za-z0-9._-]*`, except `.` and `..`, and map to
`<dir>/<name>.state`. Symlinks and non-regular files are rejected. Writes use a
temporary regular file, `fsync`, and atomic rename. Headers contain format
version, ROM SHA-256, system ID, payload length, and checksum; mismatches are
rejected before machine mutation.

Artifact paths must be absolute and contain no CR/LF/NUL. MCP additionally
restricts them beneath its private artifact directory.

## Cancellation, compatibility, and determinism

Clients enforce wall-clock deadlines; the wire request has no host-time field.
Disconnect cancels work not started. An operation already running finishes at
its exact boundary, performs input cleanup, and drops its reply; a new client
can inspect `status`.

MCP probes with a short v2 `hello`. No reply means a legacy fork, exposing only
v0.1 tools. JSON and legacy commands are never mixed on one connection. All
existing legacy commands remain byte-for-byte compatible and unacknowledged.

For the same BlastEM revision, ROM hash, initial state, region, configuration,
and command sequence, boundary frame numbers and state hashes must match across
runs. Host pacing, timestamps, window state, and logs are excluded. Real-time,
network, and nondeterministic host devices must be disabled or reported as
unsupported.
