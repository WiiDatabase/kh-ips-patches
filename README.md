# kh-ips-patches

Atmosphère IPS patches for the Nintendo Switch release of Kingdom Hearts HD 1.5+2.5 ReMIX.

## Patches

| Folder | Effect |
| --- | --- |
| `kh-skip-logos` | The launcher starts at the game selection without the Kingdom Hearts logo. Kingdom Hearts Final Mix, Kingdom Hearts II Final Mix and Birth by Sleep start at the title menu without the Disney, Square Enix and copyright screens; Birth by Sleep also skips its intro video |

All patches apply to game version 1.0.0 only.

## Install

Copy `exefs_patches/` to `SD://atmosphere/`.

## How kh-skip-logos works

Each program of the collection already skips its logos when you return from a game or to the title screen. Every patch changes one instruction in the program's `main` so that the first start takes that path as well.

| Program | Program ID | `main` build ID | Offset | Original | Patched |
| --- | --- | --- | --- | --- | --- |
| Launcher | `010090C022978000` | `95CB70F6AC62647258336DFEBDEA37427F933995` | `0x3a45c` | `cbnz w0, 0x39a28` | `b 0x39a28` (`73 fd ff 17`) |
| Kingdom Hearts Final Mix | `010090C022978001` | `BADF2FAF580B712464365C4D529D92C9D9511A9B` | `0x6ba3c` | `bl 0xdf90` | `mov w0, #1` (`20 00 80 52`) |
| Kingdom Hearts II Final Mix | `010090C022978003` | `E1BCED80FFD14436980F7490B1D3C3E7EAF06E7A` | `0x1a138c` | `mov w4, #8` | `mov w4, #0x11` (`24 02 80 52`) |
| Birth by Sleep Final Mix | `010090C022978004` | `F6716B59006BE88B0C55874FBF83DC3B24AB4D93` | `0x2d4090` | `cbz w9, 0x2d40c8` | `nop` (`1f 20 03 d5`) |

Offsets are relative to the start of the decompressed `main` module. In the IPS files they are 0x100 higher, because Atmosphère counts from the start of the NSO header.

- **Launcher:** The function at `0x3a938` returns true when `nn::oe::GetPreviousProgramIndex()` is greater than 0 or the launch parameter from `nn::oe::TryPopLaunchParameter` is `-reboot`. Its only caller then sets the launcher state to `0x17` (game selection) instead of 9 (Kingdom Hearts logo). The patch makes the branch unconditional.
- **Kingdom Hearts Final Mix:** At startup, `0x6b9d0` compares the launch parameter with `-reboot` and stores the result in a flag. With the flag set, the title state machine at `0x85630` starts at state `0x17` (title menu) instead of 0, skipping the states that show `title/disney.tm2`, `title/square.tm2` and `title/copyright.tm2`. This is the same path as the `-reboot=true` launch option of the PC version. The patch replaces the comparison with `true`.
- **Kingdom Hearts II Final Mix:** The logos are textures of `menu/us/title.2ld` and play as sequence 8 of the title layout. On the first start, `0x1a12f4` clears a first-boot flag and starts sequence 8. Otherwise it starts sequence `0x11`, the one used when you return to the title screen. The patch starts sequence `0x11` in both cases, and the first-boot flag still gets cleared.
- **Birth by Sleep Final Mix:** The title screen constructor at `0x2d4048` counts how often it ran. The first time, it loads the `copyright` layout with the logos, and after that the intro video plays. From the second time on, it loads `title_txt` directly. The patch removes the branch to the first-time path.

### Reproduce

1. Extract `exefs/main` of each program from the game and decompress the NSO, e.g. with `hactool --uncompressed`. In the decompressed file, module offset 0 sits at file offset 0x100.
2. The build ID is at offset 0x40 of the NSO header. Atmosphère matches the IPS file name against it; trailing zero bytes can be left out.
3. Load the decompressed `main` into Ghidra or IDA. Strings such as `-reboot`, `title/disney.tm2`, `title.2ld` and `copyright` lead to the functions above.
4. An IPS file is `PATCH`, then per record a 3-byte big-endian offset, a 2-byte big-endian length and the data, then `EOF`. For example, the Kingdom Hearts Final Mix patch is `PATCH 06 bb 3c 00 04 20 00 80 52 EOF`.
