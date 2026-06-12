# Snakes — binary notes

## Structure (multi-binary, unlike SonicN)

```
system/apps/6r45/6r45.app        36 KB   S60 launcher shell
system/apps/6r45_1/
  6r45_1.app                     656 KB  THE ENGINE — recompile target, 2028 functions
  6r45-zz0[1-5].pak              ~440 KB total — zlib assets (decompressed via EZLIB)
  6r45_1.r0[1-5]                 localized resource files
  arenafoundation.dll  gamecomms.dll  snapcomm.dll  sc_lib.dll
  gameutils.dll  lightdrv.dll  scxmlparser.dll  sustandard.dll
  arena.cfg  6r45_*.cfg
```

The launcher (`6r45.app`) starts the engine (`6r45_1.app`). The engine is the static-recomp
target; its single export entry is `_6r45_1_1` @ `0x1008a1e8` (the `NewApplication` analogue).

## Engine (`6r45_1.app`)

- EPOC `E32Image`, ARMv4, 656,804 bytes, **2028 functions**, entries `0x1008a1e8` / `0x10000000`.
- Lifts at **100% (0 stubs)** and the corpus compiles clean — the only lifter change needed
  vs SonicN was the `smlal` instruction.

## Imports — 598 across ~30 DLLs

| group | DLLs (counts) | HLE status |
|---|---|---|
| **core (reuse from framework)** | EUSER 193, EIKCORE 59, CONE 52, EFSRV 36, AVKON 21, BITGDI 7, FBSCLI 6, WS32 5, APPARC 4 | mostly covered by ngagerecomp |
| **S60 UI toolkit (new)** | EIKCOCTL 56, EGUL 1 | needs HLE — controls/dialogs/menus |
| **storage/parse (new)** | ESTOR 21, BAFL 10, EZLIB 2, SCXMLPARSER | stream store, zlib, config parsing |
| **install/system (new)** | INSTENG 26, SYSAGT 9, HAL 1, MSGS 1, PLPVARIANT 1 | mostly stubbable |
| **multiplayer / Arena (stub)** | GAMECOMMS 20, ESOCK 18, ARENAFOUNDATION 15, IROBEX 8, BTEXTNOTIFIERS 8, BLUETOOTH 5, INSOCK 4, SDPAGENT 4, SDPDATABASE 2 | **stub for single-player** |
| audio | MEDIACLIENTAUDIOSTREAM 1 | reuse |

## Rendering

Snakes has **no NOKIAFC** import — it renders through the **window server (WS32)** and
**BITGDI draw calls** (`BitBlt`, `DrawText`, …). This is the standard Symbian graphics path,
so HLE'ing the draw primitives should produce visible output directly — the opposite of
SonicN's hand-rolled, NOKIAFC framebuffer approach.

## Next data to gather

- [ ] Dump the engine import table → map the new DLLs to the HLE worklist.
- [ ] The window-server draw path (`CWindowGc::BitBlt`/`DrawText`) → the framebuffer.
- [ ] `.pak` format (zlib via EZLIB) for assets.
