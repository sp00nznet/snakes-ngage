# Snakes · N-Gage → Native

**Static recompilation of _Snakes_ (Nokia N-Gage, 2003) to a native executable, using
[NGageRecomp](https://github.com/sp00nznet/ngagerecomp) — the second game on the framework.**

> Before the N-Gage there was *Snake*, crawling across every monochrome Nokia phone ever
> sold. *Snakes* is its N-Gage glow-up: a 3D-ish, multiplayer, Bluetooth-and-Arena snake
> arena. It's the most quintessentially **Nokia** thing on the platform — and a deliberate
> second target to prove NGageRecomp isn't a one-game trick.

This is the **second port** built on NGageRecomp, after [`sonicn-ngage`](https://github.com/sp00nznet/sonicn-ngage).
The reusable recompiler + runtime live in [ngagerecomp](https://github.com/sp00nznet/ngagerecomp);
this repo is Snakes-specific.

## Why Snakes (and what's different from SonicN)

The two games stress the framework in opposite ways:

| | SonicN | **Snakes** |
|---|---|---|
| Presentation | NOKIAFC direct framebuffer | **window server (WS32) + `CFbsScreenDevice::Update`** |
| HLE surface | small (233 imports, 11 DLLs) — bypasses the S60 UI | **broad (598 imports, ~30 DLLs)** — full S60 UI toolkit |
| Structure | single `.app` | **multi-binary**: launcher + engine + custom DLLs |
| Boot path | NewApplication → ConstructL → CPeriodic loop | S60 app framework → `CAknAppUi::ConstructL` → active-scheduler event loop |

> **Correction (from analysis + the EKA2L1 oracle):** an early bet was that Snakes renders
> through standard draw primitives (`BitBlt`/`DrawText`), making pixels "free." It does
> **not** — its import table has no such draw calls. Like SonicN, Snakes **hand-rolls
> pixels into a `CFbsBitmap`** (`DataAddress`) and presents that bitmap via the window
> server (`CFbsScreenDevice::Update`). The real difference is the **breadth of S60/CONE
> HLE** needed to boot it through the standard app framework, not the rendering style.

What makes Snakes a strong second target stands regardless: it's structurally different
from SonicN (multi-binary, 8× the import surface, full S60 UI + window server), so getting
it to run proves the recompiler and core HLE aren't SonicN-specific.

## The target

| | |
|---|---|
| Engine | `system/apps/6r45_1/6r45_1.app` — Symbian `E32Image`, ARMv4, 656 KB, **2028 functions** |
| Launcher | `system/apps/6r45/6r45.app` (36 KB) — the S60 shell that starts the engine |
| Assets | `6r45-zz0*.pak` (zlib-compressed via `EZLIB`) |
| Custom DLLs | `gamecomms`, `arenafoundation`, `snapcomm`, `sc_lib`, `gameutils` … (N-Gage Arena / Bluetooth multiplayer — **stubbable for single-player**) |

## Status

```
[✔] Engine lifts: 2028 functions / 138,010 instructions → 100% (0 stubs)
[✔] Whole 154k-line corpus compiles clean under clang -Wall
[✔] Only lifter addition needed: the `smlal` instruction (Snakes uses it, SonicN didn't)
[✔] Engine runs its own code: NewApplication + CAknAppUi::ConstructL execute
[✔] EKA2L1 live oracle: the real game runs under instrumentation (see docs/ORACLE-FINDINGS.md)
[~] ConstructL bring-up via differential debugging vs the oracle — advancing fault-by-fault
[ ] ConstructL completes → menu event loop (dispatcher every frame, event 0x3E8)
[ ] Asset (.pak/.mbm) decode + game-start (needs oracle input injection) → first frame
```

**The framework generalizes** (milestone met): the recompiler lifts a structurally
different binary with zero game-specific changes. Bring-up is now **oracle-driven** — we run
the recomp's `ConstructL` under the hard memory guard, find where it diverges from the real
game running in EKA2L1, and fix the HLE to match. This already corrected a key
misunderstanding: the suspected "multiplayer wall" was just unmapped import stubs leaking
garbage into `r0`, faking a Bluetooth path that never runs at boot. Fixes so far: stubs
return `0`, real `TDesC::Ptr`/`Length`, object-factory `NewL`s return real objects.

See [`docs/ORACLE-FINDINGS.md`](docs/ORACLE-FINDINGS.md) and [`PROGRESS.md`](PROGRESS.md).

## Legal

Snakes is © Nokia. Nothing copyrighted is distributed here — bring your own dump into
`game/` (gitignored). The recompiled output is a derivative of *your* copy.
