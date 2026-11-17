# Ristar - Nintendo Switch build

## Build

You need the devkitPro Switch toolchain and these portlibs:

```
dkp-pacman -S switch-dev switch-sdl2 switch-mesa switch-freetype switch-harfbuzz switch-libpng switch-bzip2 switch-zlib
cd switch
make                 # -> Ristar_nx.nro
make NXLINK=1        # also logs over nxlink (printf / stderr / a fps line every 5 s)
make clean
```

The Makefile is the Mickey & Donald Collection's (v1.1) with this project's names and source list. For logs:
`make NXLINK=1`, then `nxlink -s Ristar_nx.nro` on the PC.

## Install

Copy `Ristar_nx.nro` to `sdmc:/switch/Ristar_nx/` and start it from the homebrew menu. Put your ROM(s) in
`sdmc:/switch/Ristar_nx/roms/` (or next to the `.nro`): they are found by CRC32, so the file name does not matter and a
`.zip` is fine. The USA / Europe and the Japanese release are both supported; with both there, Play asks which one.
Settings and save states live in `sdmc:/switch/Ristar_nx/`.

## Controls

Genesis A / B / C are on Y / B / A, D-pad or left stick to move, + is Start; all rebindable under Options > Controls.
Menus: (A) choose, (B) back. In the game, - opens the pause menu (save / load state in 20 slots, options, reset,
back to the launcher); ZR held is fast forward, ZL held is rewind.

## Widescreen

Options > Video > Widescreen: Off, 16:10 or 16:9 (the first start picks 16:9). Stages are shown wide, with the score,
stars and lives at the screen edges ("Re-anchor HUD"); round cards, bonus rooms and the scenes between stages are
wide too; the title and menus stay 4:3. It can be changed
in the middle of a stage. If the frame rate drops on the console, Options > Video > Video thread and the CPU Overclock
are the levers from the v1.1 emulator. See `../docs/WIDESCREEN.md` for what is known not to be perfect.
