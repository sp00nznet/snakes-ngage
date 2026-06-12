# Snakes recomp — progress log

Newest entries on top. The point of this port: prove NGageRecomp generalizes beyond SonicN.

## Milestones

| # | Milestone | State |
|---|---|---|
| 0 | Binary identified (engine `6r45_1.app`, E32Image, ARMv4, 2028 funcs) | ✅ done |
| 1 | Whole engine lifts ARM→C | ✅ done — 100% / 0 stubs (added `smlal`) |
| 2 | Whole corpus compiles clean (`clang -Wall`) | ✅ done — 154k lines |
| 3 | Import table dumped → HLE worklist (598 / ~30 DLLs) | 🟡 next |
| 4 | HLE bring-up (reuse EUSER/EFSRV/desc/soft-float; add S60 UI) | ⬜ |
| 5 | Engine entry runs → window-server draw → first frame | ⬜ |

## Log

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
