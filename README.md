# smsdat

**Single-file HTML tool for SMS/GG homebrew development.** Analyse ROM bank
usage and diff RAM dumps, entirely in the browser — no server, no CDN, no
tracking. Bilingual UI (FR/EN).

## Two tabs

### 🗂️ Bank Analyzer

Reports the occupation of each 16 KiB ROM bank and the 8 KiB main RAM from a
symbolic file — either **WLA-DX `.sym`** or **SDCC `.map`**. Format is
auto-detected.

<img width="1136" height="1034" alt="Capture d’écran 2026-07-12 à 06 58 50" src="https://github.com/user-attachments/assets/2ecd7ae2-f3ad-4c98-b611-b08bc27814f9" />

- Progress bar per bank (green < 60 %, orange < 85 %, red beyond)
- Sortable symbol table (address, name, area, size in decimal + hex)
- Per-symbol size bar overlaid on the row for a quick visual scan
- **Treemap** SVG built with a squarified layout (Bruls-Huijbregts-van-Wijk),
  hover tooltip showing address, size, % of bank, and area
- Global summary (banks used, ROM/RAM free, average occupation)
- CSV export

### 🔍 RAM Diff

Compares two SMS RAM binary dumps (8 KiB) and shows a hex grid of the
differences.

<img width="1136" height="1034" alt="Capture d’écran 2026-07-12 à 06 58 55" src="https://github.com/user-attachments/assets/f1750de3-207a-4a3a-ae06-ce47139253f1" />

- 16 × 512 hex grid with identical bytes greyed out and diffs highlighted
  (v1 red, v2 green)
- Filter to hide rows with no difference
- ASCII column on the right
- Context-aware tooltip when SDCC files are provided: load `rom.map` +
  `_structures.h` to see the struct name, field, type, and byte offset for
  the cell under the cursor

## Usage

Open `smsdat.html` in a recent browser. Drag & drop
the files or use the pickers. Everything runs locally.

## Expected file formats

### WLA-DX (Bank Analyzer)

The `.sym` file must be produced with:

```sh
wlalink -A -S linkfile out.sms
```

The **`-S` flag is required** — it writes the `[definitions]` section with
the size of every symbol.

For **accurate** per-bank occupation, add a sentinel label `_end_bank_N:`
(where `N` is the bank number, decimal or hex — your choice) right after
the last data of each bank. Without it, WLA-DX inflates the `_sizeof_` of
the last label in each bank to reach the end of the 16 KiB slot, which
makes banks look 100 % full even when they aren't.

```wla
.BANK 3 SLOT 2
.ORG $0000
    ...your data...
    pause_lut:
    .INCBIN "pause.bin"
_end_bank_3:   ; ← this label marks the true end of data
```

Banks with no `_end_bank_N` marker get an orange ⚠ badge, and a banner at
the top of the results lists them.

### SDCC (Bank Analyzer)

The `.map` file produced by `sdld` is used as-is. The devkitSMS convention
is recognised (`_BANK2 = 2 × 0x10000 + 0x4000`, etc.). The tool:

- Merges area blocks re-emitted at every linker page break
- Keeps only symbols **with a module** (`rom`, `banked_code_2`, `crt0`) for
  display, and uses the linker-generated ones (`_abs`, `s__CODE`, …) purely
  as boundary markers for size computation
- Attributes to the last symbol of an **ABS** area the remainder of the
  area's declared size (necessary for SMS header areas, where the area's
  `Addr` field is nominal)

**Known limitation:** for `__at()` variables in RAM, the `.map` only
contains addresses, not array sizes. The tool falls back to
`size = next_address − current_address`, which **can overestimate** a
variable's size when it is followed by a large unlabelled gap. Typical case:
two `__at(0xD100) uint8_t buf[256]` with the next symbol at `0xD600` — the
tool shows 1280 B instead of 256 B.

### RAM Diff

- **BIN**: raw RAM binary dump (typically 8 192 B). Two files required.
- **`rom.map`** (optional): SDCC linker map, provides symbol names at each
  address.
- **`_structures.h`** (optional): C header with `typedef struct` and
  `extern __at(0xADDR) type name;` declarations. Used to resolve field
  names and types in the tooltip.

## Code layout

- **`smsdat.html`** — single file, two independent IIFEs (`BANK ANALYZER`
  and `RAM DIFF`) sharing only `LANG` and an i18n callback list through the
  module scope. No ID or variable collisions.

## License

MIT (to be confirmed)
