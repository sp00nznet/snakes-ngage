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

### 2026-06-13 (cont.) — oracle-driven differential debugging: ConstructL advancing
- Built the **differential loop**: run recomp ConstructL under hard mem-guard (faults at
  first bad access), hook the faulting functions in EKA2L1 to see what reality does, fix
  the HLE to match. Captured the real WS object (`0x00700720`, ROM vtable `0x50593888`) and
  the focused control (vtable `0x1009BDAC`, screen rect 176x208) — see `docs/ORACLE-FINDINGS.md`.
- Three oracle-validated fixes, each advancing ConstructL past a real divergence:
  1. **Stubs now return `r0=0`** (gen_hle) — unmapped imports were leaking garbage in r0;
     `GAMECOMMS_20` (multiplayer) returned non-zero → the game took its Bluetooth
     `BTGetName` branch (the recomp's old "wall" `sub_100507B4(a1=0)`). The oracle proved
     that chain never runs at boot. 0 = "feature absent".
  2. **`TDesC16/8::Ptr` + `Length`** now real (`hle_TDesC_Ptr/_Length` via `ngage_desc`).
  3. **Snakes audio `NewL`** → object-returning shim (not 0).
- ConstructL fault frontier: deep-BT `sub_100507B4` → region `@0x6` → audio `@0` →
  `sub_10024648 @0`. Methodology proven; remaining faults are mostly factories (return an
  HLE object) or accessors (need real impl). Framework files compile clean (-Wall).

### 2026-06-13 — EKA2L1 live oracle online; run-loop + real layouts captured
- Stood up **EKA2L1 as a ground-truth oracle**: installed the Nokia N-Gage device
  (NEM-4) headless from the firmware ROM, and **Snakes runs in it** (clean, no panic).
  Recipe + scripts in `ngagerecomp/harness/` (`run-ngage-oracle.ps1`, `snakes_oracle.lua`).
- Lua breakpoint hooks fire at **raw IDA addresses** (no rebasing); `mem`/`cpu` dump
  registers + memory. Captured real object layouts and the run-loop — see
  `docs/ORACLE-FINDINGS.md`.
- **Found the run-loop**: at the menu, dispatcher `sub_1002A830` runs every frame with
  event command `0x3E8` (idle/redraw) — the event pump the recomp lacked. The recomp can
  drive the menu by calling it per-frame with `*(evt+28)=0x3E8`.
- **Found the divergence**: the recomp's soft-guarded `sub_100507B4(a1=0)` path never
  executes in the real ConstructL — it was an artifact of faked WS32 objects. The true
  fix is to make the recomp's window-server HLE objects match the captured real layouts
  (WS object `0x00700720`, AppUi/game-state graphs).
- **Next:** dump the real WS object + control; inject menu input to capture the real
  game-start command + connection-object setup; hook WS/fbs IPC for the draw spec.

### 2026-06-12 (cont.) — run-loop primitive built; menu/multiplayer wall confirmed
- Built the reusable **active-scheduler + event-injection primitive** in the framework
  (`runtime/src/hle/scheduler.c`): `CActiveScheduler::Add` (records CActive objects),
  `User::RequestComplete`/`WaitForRequest`, `CActive::SetActive`, a scheduler step
  (`ngage_as_step` — completes requests & calls `RunL`), and direct injection helpers
  (`ngage_inject_key` → `OfferKeyEventL`, `ngage_invoke_draw` → `Draw`, `ngage_call_vmethod`).
  Compiles clean under `-Wall -Wextra`. **74/598 shims.**
- Recovered the focused control's full **vtable from the live (relocated) runtime**
  (IDA shows 0 — the pointers are relocation-filled, which our recompiler applies): the
  control = `appui[25]` is a plain container whose `OfferKeyEventL` (slot 3) and `Draw`
  (slot 26) are the **base CCoeControl thunks — NOT overridden**. So it neither handles keys
  nor renders game content; key handling lives in `CAknAppUi`/the AVKON menu, rendering in
  the engine.
- Confirmed the scheduler shim works: `ConstructL` registers **8 CActive objects**.
- Mapped the start command: the dispatcher `sub_1002A830` (reached via a multiple-inheritance
  command-observer thunk) starts the game on command **`0x43E`** — but that path creates the
  engine *alongside a Bluetooth/Arena connection object* (`appui+98700`, `TBTSockAddr`).
  Injecting `0x43E` synthetically does NOT start the engine (precondition state the menu
  would set is absent), confirming there is **no clean shortcut to a frame**.
- **Conclusion (honest):** a Snakes frame requires either (a) faithful AVKON menu + event +
  active-scheduler emulation to navigate menu→play naturally, or (b) standing up the
  multiplayer connection-object graph so the engine-start path runs — *plus* `.pak`/`.mbm`
  asset decode for visible content. This is the same class of wall SonicN hit: getting from
  "init runs" to "real runtime state" is the hard part of HLE recomp. The run-loop primitive
  is the first reusable piece of that; the rest is a multi-session build.

### 2026-06-12 (cont.) — render architecture fully mapped; the wall is the game-state machine
- **Correction to the original thesis:** Snakes does NOT render via BitBlt/DrawText draw
  primitives. Its import table has *no* draw calls — only `CFbsBitmap::DataAddress` (raw
  pixel buffer), `CFbsScreenDevice::Update` (present), and gc *setters*. So Snakes
  **hand-rolls pixels into a CFbsBitmap then presents via `Update`** — structurally the
  *same* as SonicN, just window-server-presented instead of NOKIAFC. The chosen-game
  advantage was smaller than thought; the good news is the `CFbsBitmap` HLE + present point
  transfer directly, and `Update` is a *clean* present hook (mapped → `hle_NOKIAFC_present`).
- With **soft-guard**, `AppUi::ConstructL` now runs to completion (rc=0, only 16 soft reads).
  But it registers **0 periodics** — unlike SonicN, Snakes does NOT start its game loop in
  ConstructL.
- **Mapped the full game-loop architecture** (IDA):
  - Engine object = `newL(864)` @ `sub_1006845C` (ctor `sub_100684B0`), created via
    `sub_10068440`. Layout: state @ `+24`, control @ `+848`, counter @ `+852`,
    CPeriodic @ `+856`.
  - Game-start `sub_10068E60` copies a name, sets state, calls `sub_10068C78`/`D54` →
    `sub_100695A0` = `CPeriodic::Start(5 000 000µs, …, callback sub_100697A8)`.
  - Tick `sub_100697A8` → `sub_10069618`: a **state machine** on `+24` (2/3 = playing →
    `sub_10068DC8`/`sub_10069140` under TRAP; 6/7 = transition; else idle). Render goes
    through the control @ `+848` (vtable+36).
- **The wall:** the render path is gated behind the menu→play **state-machine transition**,
  which in a real run is driven by the active scheduler processing window-server **key
  events**. The HLE has no faithful active-scheduler/event pump, so soft-guard "completing"
  ConstructL does not produce a playable game object — driving the tick on it renders
  garbage/faults. Reaching a real frame needs EITHER a faithful event/active-scheduler pump
  OR a large manual reconstruction of the engine object + state + a real window/gc control.
- **69/598 shims** (added Snakes graphics maps: named `CFbsBitmap::Load`,
  `CFbsScreenDevice::Update`→present, `CFbsBitmapDevice::NewL`, `CWsBitmap`).

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
