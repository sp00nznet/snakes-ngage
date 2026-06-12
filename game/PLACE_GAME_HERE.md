# Put your own Snakes dump here

This repo ships **no game files**. Drop your legally-obtained Series 60 install of Snakes
into this folder:

```
game/
└── system/apps/
    ├── 6r45/6r45.app            <- launcher (36 KB)
    └── 6r45_1/
        ├── 6r45_1.app           <- the engine (the recompile target, ~656 KB)
        ├── 6r45-zz0*.pak        <- zlib-compressed assets
        └── *.dll                <- gamecomms / arenafoundation / sc_lib / ...
```

Everything under `game/` is `.gitignore`d. The engine `6r45_1.app` is an EPOC `E32Image`,
ARMv4, 656,804 bytes, 2028 functions.
