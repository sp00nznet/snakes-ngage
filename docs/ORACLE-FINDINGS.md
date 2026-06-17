# Snakes — EKA2L1 oracle findings

Ground truth captured by running the real game in EKA2L1 (Nokia N-Gage / NEM-4) with
Lua breakpoint + memory-dump hooks (`harness/oracle-trace.lua` →
`<eka2l1>/scripts/snakes_oracle.lua`). IDA addresses work directly in EKA2L1 hooks
(resolved against the `6r45_1.app` codeseg; no rebasing). Code loads at `0xE0000000`
(so a runtime vtable `E009BB20` = IDA `0x1009BB20`).

## The run-loop (the thing the recomp was missing)

At the menu, the command/event dispatcher `sub_1002A830` is invoked **every frame** with
the event's command field `*(a2+28) = 0x3E8` (1000) — the idle/redraw/foreground event.
This is Snakes' actual event pump: CONE's active scheduler delivers window-server events,
and the dispatcher runs continuously. The recomp can reproduce the menu loop by calling
`sub_1002A830(stateObj, evt)` with `*(evt+28)=0x3E8` each frame. Other commands (menu
select → new game) require injecting input events (not seen headless — only `0x3E8`).

`engineFactory@0x10068440` does NOT fire at the menu — the engine is created only when a
game actually starts (confirms game-start is gated behind a menu command we still need to
trigger via input).

## Real object layouts (heap addrs are per-run; structure is stable)

### AppUi (CAknAppUi) — `ConstructL` this
```
+000: <vtable E009BB20> 00700104 0 0 0 <E009C5C4> <E009C628> <E009C618>
+020: 00701E48 0 0 0 00000010 0 0 0
+040: 0 0 <E009C5F8> <E009BBA4> 0 0 0 0
+060: <E009BB98> 0 0 0 0  [+0x64]=00700124  0 0     <- appui[25] = focused control = 0x00700124
+080: 00000188 00702664 FFFFFFFF 0  10000037 10003A38 101FD3DB 0   <- UIDs: 0x10000037, app UID3 0x101FD3DB
```

### Window-server object holder (`sub_10023930` stores, `sub_10023DC0` derefs)
```
+000: <vtable E009B588> 0 0 0 0  FFFFFFEB <E009B5A0 E009B5C8 E009B5D8 E009B5BC E009B5B0 E009B5E4> ...
+050: 00702394 00700124 00700628 0
+060: [a1[24]] = 0  -> becomes 0x00700720   <- the real WS object the recomp must emulate
```
`a1[24]` (offset `+0x60`) is null in `sub_10023930` and holds `0x00700720` by
`sub_10023DC0` — i.e. the window-server object is created in between. Dump `0x00700720`
next to learn the WS object's own layout (what the recomp's `hle_ws_object` should mirror).

### Game-state object — dispatcher this (`0x007F9B28`)
```
+000: <7 vtables E009B458 E009B474 E009B4C0 E009B4DC E009B498 E009B4E8 E009B464> 00000100
+020: 009F1978 009F1978 00A22AD8 00A1279C 00A36F24 009FDB98 009DFBC4 009EB634   <- sub-object graph
+040: 00A89D84 00A8AC50 00A4B550 00A64A38 00A8587C 0 00A8CDDC 00A8D3FC
+060: 0 0081CB88 0081CBE0 0  00000004 00000006 00000008 00000101
+080: 0 0 30000000 00000100 ...
```
Multiple-inheritance object (7 vtables) — the app's main view/command-observer. Reached
via a this-adjusting thunk (consistent with the IDA analysis).

## Divergence found vs the recomp

The recomp (soft-guard) drove into `sub_100507B4 (a1=0)` etc. — but those functions
**never fire** in the real ConstructL at boot. So that path was an artifact of soft-guarded
null reads pushing the recomp down a wrong branch; the true divergence is upstream, where
the recomp's faked `WS32_348`/window-server object differs from the real `0x00700720`
graph. Fix: make the recomp's WS HLE objects match these captured layouts.

## Differential debugging loop (oracle vs recomp) — RESULTS

Method: run the recomp's ConstructL under the hard memory guard (faults at the first bad
access), read the fault's call stack, hook those functions in EKA2L1 to see what reality
does, and fix the recomp's HLE to match. Each fix advances ConstructL to the next real
divergence.

Fixes landed this way:
1. **Stubs return `r0=0`** (gen_hle.py). Unmapped imports left garbage in r0; e.g.
   `GAMECOMMS_20` (multiplayer) "returned" non-zero, so `CgameEngine::BTGetName`
   (`sub_1000BD7C`) took the Bluetooth branch into `sub_1000E6E0 → … → sub_100507B4(a1=0)`.
   The oracle proved that whole chain never runs at boot. Returning 0 = "no multiplayer".
2. **`TDesC16/8::Ptr` → `hle_TDesC_Ptr`** (+ `Length`). The region check
   `(*(Ptr(buf)+6)|*(Ptr(buf)+7)<<8)==0x47` faulted because `Ptr` was stubbed (→0 after
   fix 1). Now returns the real descriptor data pointer via `ngage_desc()`.
3. **Snakes audio `NewL__CMdaAudioOutputStream…` → `hle_CMdaAudioOutputStream_NewL`**.
   `sub_1008F050` does `v4=NewL(); (*(*v4+16))(…)` — an object factory must return a real
   object (no-op vtable), not 0.

ConstructL fault frontier moved: `sub_100507B4` (deep BT) → region check `@0x6` → audio
`@0` → `sub_10024648 @0` (next, inside the window-setup `sub_10023930`). The loop is
repeatable: most remaining faults are either object factories that must return an HLE
object, or accessors that need a real (not stub) implementation.

## Next captures (need input injection or GUI)

- Dump `0x00700720` (real WS object) + the focused control `0x00700124`.
- Drive a menu select to capture the real game-start command + `engineFactory` args +
  the connection-object setup (the multiplayer-entangled path).
- Hook the window-server IPC / fbs to capture the exact draw calls (rendering spec).
