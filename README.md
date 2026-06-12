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
| Rendering | NOKIAFC direct framebuffer + hand-rolled sprite engine | **window server (WS32) + BITGDI draw calls** |
| HLE surface | small (233 imports, 11 DLLs) — bypasses the S60 UI | **broad (598 imports, ~30 DLLs)** — full S60 UI toolkit |
| Structure | single `.app` | **multi-binary**: launcher + engine + custom DLLs |

SonicN's hand-rolled sprite engine is the wall it hit (blank screen, custom sprite format).
Snakes renders through standard Symbian **draw primitives** — so HLE'ing `BitBlt`/`DrawText`
etc. should produce visible pixels *directly*, without reverse-engineering a sprite format.
The cost is a much bigger framework HLE surface (EIKCOCTL, AVKON, CONE, ESTOR, …). Different
trade-off; the more "standard" rendering is the bet.

## The target

| | |
|---|---|
| Engine | `system/apps/6r45_1/6r45_1.app` — Symbian `E32Image`, ARMv4, 656 KB, **2028 functions** |
| Launcher | `system/apps/6r45/6r45.app` (36 KB) — the S60 shell that starts the engine |
| Assets | `6r45-zz0*.pak` (zlib-compressed via `EZLIB`) |
| Custom DLLs | `gamecomms`, `arenafoundation`, `snapcomm`, `sc_lib`, `gameutils` … (N-Gage Arena / Bluetooth multiplayer — **stubbable for single-player**) |

## Status — the framework generalizes ✅

The first milestone is already met: **NGageRecomp lifts the whole Snakes engine with zero
changes** beyond one instruction.

```
[✔] Engine lifts: 2028 functions / 138,010 instructions → 100% (0 stubs)
[✔] Whole 154k-line corpus compiles clean under clang -Wall
[✔] Only lifter addition needed: the `smlal` instruction (Snakes uses it, SonicN didn't)
[ ] HLE bring-up: EUSER/EFSRV reuse from the framework; then the S60 UI + window server
[ ] Boot the engine entry → first window-server draw → first frame
```

So the **recompiler is proven not to be SonicN-specific**. The remaining work is Snakes's
(larger) HLE surface — most of EUSER/EFSRV/descriptors/soft-float reuse directly from the
framework; the new work is the S60 UI toolkit + window-server drawing, plus stubbing the
multiplayer/Arena stack.

## Legal

Snakes is © Nokia. Nothing copyrighted is distributed here — bring your own dump into
`game/` (gitignored). The recompiled output is a derivative of *your* copy.
