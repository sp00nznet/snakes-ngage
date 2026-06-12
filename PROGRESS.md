# Snakes recomp — progress log

Newest entries on top. The point of this port: prove NGageRecomp generalizes beyond SonicN.

## Milestones

| # | Milestone | State |
|---|---|---|
| 0 | Binary identified (engine `6r45_1.app`, E32Image, ARMv4, 2028 funcs) | ✅ done |
| 1 | Whole engine lifts ARM→C | ✅ done — 100% / 0 stubs (added `smlal`) |
| 2 | Whole corpus compiles clean (`clang -Wall`) | ✅ done — 154k lines |
| 3 | Import table dumped → HLE worklist (598 / ~30 DLLs) | ✅ done |
| 4 | Engine runs: NewApplication + AppUi::ConstructL execute | ✅ done (63/598, framework reuse) |
| 5 | Window-server HLE (WS32 objects + CWindowGc draws) → first frame | 🟡 in progress |

## Log

### 2026-06-12 (cont.) — window-server HLE: ConstructL drives deep into init
- Resolved the WS32 ordinals from call-site usage (firmware `ws32.dll` exports are
  symbol-less; Snakes uses only 5 raw WS32 ordinals — its drawing goes via CONE).
- Built the window-server HLE start: a shared **self-referential HLE-object factory**
  (`hleobj.c` — `ngage_hle_object`: vtable of no-op slots + members seeded with a single
  "universal" object, so the window-server object graph never dereferences null), the WS32
  factory shims (`hle/wserv.c`), and the CONE/EIKCORE singleton getters
  (`CCoeEnv::Static`, `CEikAppUi::Application`) returning that universal object.
- Effect: `AppUi::ConstructL` now drives **far deeper** — the fault chain went from ~5
  frames (window-server setup) to **12+ frames** into the engine's data-init subsystem
  (`sub_10050xxx`). The synthetic-object patterns from SonicN's graphics HLE transfer
  directly to Snakes's window server. **67/598 shims.**
- **Next:** keep grinding the init nulls (now specific game objects, not just getters);
  then the CONE/CWindowGc draw path → first window-server frame.

### 2026-06-12 (cont.) — the engine runs its own code
- Dumped the engine import table (598 imports / 31 modules). `gen_hle` reused **63 framework
  shims for free** (EUSER/EFSRV/descriptors/soft-float/heap/leave), named-stubbed the other
  535. No ordinal collisions with SonicN's HLE map.
- **First contact identical to SonicN**: the engine's `NewApplication` (`_6r45_1_1` @
  0x1008a1e8) executes — `operator new` the `CEikApplication` object, ctor, vtable, returns
  a valid object — hitting only the same benign `CEikApplication` ctor stub. Zero
  Snakes-specific HLE needed.
- Drove the AppUi `ConstructL` (`sub_1008A560`): it runs into the **window-server setup**
  and faults at `*(a1+96)` being null — `WS32_348` (a window-server object factory) is a
  named stub returning null, then the game derefs it.
- **Frontier = the window-server HLE.** Snakes renders through `RWsSession`/`RWindow`/
  `CWsScreenDevice`/`CWindowGc` (the WS32 client-server stack) + draw calls — the subsystem
  SonicN bypassed via NOKIAFC. Same synthetic-vtable-object pattern as the SonicN graphics
  objects, applied to WS32. That, plus the S60 UI toolkit (EIKCOCTL/AVKON), is the work to
  a first frame.

### 2026-06-12 — the framework generalizes
- Selected Snakes as the 2nd target specifically because it renders via the **window server +
  BITGDI draw calls** (no NOKIAFC) — a different path from SonicN, and one where HLE'ing the
  draw primitives should yield visible pixels directly.
- Found Snakes is **multi-binary**: launcher `6r45.app` + engine `6r45_1.app` + custom DLLs
  (Arena/Bluetooth multiplayer — stubbable for single-player). See `docs/BINARY-NOTES.md`.
- **Ran the full NGageRecomp pipeline on the engine, unchanged**: 2028 functions / 138,010
  instructions lifted at **100%, 0 stubs**, and the whole **154k-line corpus compiles clean
  under `clang -Wall`**. The *only* lifter change required was adding the `smlal` instruction
  (committed upstream to ngagerecomp). SonicN still lifts 100% (no regression).
- **Conclusion:** the recompiler is not SonicN-specific. The remaining work is Snakes's
  larger HLE surface — core Symbian (EUSER/EFSRV/descriptors/soft-float) reuses from the
  framework; the new work is the S60 UI toolkit + window-server drawing.
- **Next:** dump the engine import table and start HLE bring-up.
