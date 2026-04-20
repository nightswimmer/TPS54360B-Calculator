# Project: TPS54360B Buck Converter Component Calculator

## What this project is
A single-file self-contained HTML/JS web application that calculates all component values needed to design a power supply circuit using the TPS54360B (Texas Instruments, 60V 3.5A synchronous step-down DC/DC converter). The datasheet (SNVSB93) is the reference for all equation numbers used in the UI and this document.

## Repository layout
- `index.html` — the full application (HTML + CSS + JS + embedded SVG schematic), all in one file. Named `index.html` so GitHub Pages (and any plain static host) serves it automatically at the repo root.
- `README.md` — GitHub landing page.
- `LICENSE` — MIT.
- `CLAUDE.md` — instructions for the Claude Code assistant when working in this repo.
- `PROJECT_INSTRUCTIONS.md` — this file, the long-form project state for future Claude sessions.

No build step, no server, no dependencies beyond the two Google Fonts (`JetBrains Mono`, `Syne`) pulled from `fonts.googleapis.com` at page load.

## Browser testing
Just double-click `index.html` (or open it via `file://`). Because the file is called `index.html`, hosting the repo on GitHub Pages or any static file server automatically serves the calculator at the root URL with no extra config. The old numbered-filename / `serve_calculator.py` workflow has been retired — the app has no features that require HTTP.

Useful globals for debugging in the console:
- `window._r` — the full result object from the last `calc()` call.
- `window.OV` — the active overrides object `{RLS_ohm, L_uH, C_out_uF, R4_ohm, C5_pF, C8_pF}` (access via the top-level IIFE scope; typically inspect via the Sources panel or add a temporary `window.OV=OV` line for debugging).
- `window._schemUpdate()` — force the schematic overlay labels to refresh.

## App architecture

### Sidebar inputs (13 user parameters)
- `Vin(max)`, `Vin(min)` — input voltage range (4.5–60 V)
- `Vout` — output voltage (0.8–59 V)
- `Iout` — output current (0.1–3.5 A)
- `fSW` — switching frequency (100–2500 kHz)
- `Rdc` — inductor DCR (mΩ)
- `Vd` — diode forward voltage (V)
- `Resr` — output cap ESR (mΩ)
- `ICL` — current limit (A)
- `Vsc` — short-circuit output voltage (V)
- `ΔVout%` — allowable transient voltage deviation (%)
- `ΔIout%` — load step size (%)
- `Kind` — inductor ripple factor

Each input has a paired number-entry box and slider, linked via `linkPair()`. Validation messages appear under each row (`.vm` spans).

### Fixed chip parameters (read-only)
`tON = 135 ns`, `RDS(on) = 92 mΩ`, `fDIV = 8`, `QG = 3 nC` (hard-coded in formula), `IQ = 146 µA` (hard-coded in formula).

### Defaults
`DEFAULTS` object in the JS:
```
vin_max:'60', vin_min:'12', vout:'5', iout:'3.5', fsw:'600',
rdc:'25', vd:'0.7', resr:'2.5', icl:'4.7', vsc:'0.1',
dvout_pct:'4', diout_pct:'75', kind:'0.3'
```
Note: the HTML `value="..."` attribute on the `#resr` input is `0.25`, while `DEFAULTS.resr` is `'2.5'`. A user hitting ↺ reset will get `2.5`; a fresh load with no saved state uses `0.25` from the HTML. If this matters, sync the two.

### Auto-recalc behaviour
- Sliders: 50 ms debounce on the `input` event.
- Number inputs: recalc on `Enter` or `blur`.
- No Calculate button — fully reactive.

### Standard value lookup tables
- Resistors: E24+E96 combined superset — 114 values, `var E96=[...]`, used by `nearE24()` which iterates over `E96` across decades.
- Capacitors µF: E24 — 137 values, `var CAP_UF=[...]`, used by `nearCap()`.
- Capacitors pF: E24 — 89 values, `var CAP_PF=[...]`, used by `nearPF()`.
- Inductors µH: E12 — 29 values, `var IND_UH=[...]`, used by `nearL()`.

`parseVal(raw, defaultUnit)` handles µH/uH/mH, µF/uF/nF/pF, kΩ/Ω/mΩ suffixes on override fields.

## Calculations implemented (by datasheet equation number)

- **Eq. 3** — RLS/RHS feedback resistors.
- **Eq. 8** — RT resistor from fSW.
- **Eq. 9 / 10** — Max fSW limits (skip-mode and frequency-shift), evaluated at `Vin(max)` as worst case.
- **Eq. 28–31** — Inductor: Lmin, ripple, peak, rms.
- **Eq. 32 / 34** — Output capacitor from transient and ripple criteria.
- **Eq. 38** — Input capacitor RMS ripple current, evaluated at `Vin(min)` per the datasheet.
- **Eq. 44–51** — Type 2A compensation: fp, fz, fco, R4, C5, C8.
- **Eq. 52–55** — IC power dissipation:
  - `P_COND = Iout² · RDS(on) · (Vout / Vin_min)` — conduction (worst-case duty).
  - `P_SW = Vin_max · fSW · Iout · trise`, `trise = Vin_max · 0.16 ns/V + 3.0 ns`.
  - `P_GD = Vin_max · QG · fSW` (`QG = 3 nC`).
  - `P_Q = Vin_max · IQ` (`IQ = 146 µA`).
  - `Ptot = P_COND + P_SW + P_GD + P_Q`.
- Non-datasheet standard formulas (displayed with equation number `—`):
  - Inductor copper loss: `P_L = I_rms² · Rdc`.
  - Diode dissipation: `P_D = Vd · Iout · (1 − D)`.

## Component overrides
Overrideable cards: `RLS`, `L`, `C_out`, `R4`, `C5`, `C8`.

Per card:
- Uses `valRow()` to render the value strip.
- Override field sits inline to the right of the calculated / standard value.
- ↺ icon button resets that override to auto.
- Amber border + "overridden" badge when active.
- `OV` object holds active overrides: `{RLS_ohm, L_uH, C_out_uF, R4_ohm, C5_pF, C8_pF}`.
- Changes propagate through the full dependency chain on the next `recalc()`.

## Key JS architecture

- `calc(inp, OV)` — pure function, returns the full result object `r`.
- `render(r)` — builds all card HTML, writes to `#main`.
- `recalc()` — `validate()` → `calc()` → `render()` → `saveState()`.
- `diffAndHighlight(r)` — compares new vs `prevSnapshot`, adds `.changed` (one-shot animation) and `.updated` (persistent amber left border, cleared on next recalc) to cards whose tracked values changed.
- `valRow(calcVal, stdLabel, stdVal, isOv, ovVal, id)` — renders value + inline override field.
- `fb(eq, sym, sub, res)` — renders one formula block; `eq` is the equation number (`'—'` for non-datasheet formulas).
- `initFormulaToggles()` — injects a `∑` button into the `.clbl` header of every `.card` that has `.fblock` children; restores `visibleFormulas` state across recalcs.
- `saveState()` / `loadState()` — localStorage persistence (key `'tps54360b_state'`).
- `saveToFile()` / `loadFromFile()` — JSON config export / import.
- `DEFAULTS` — the 13 input defaults.
- `CARD_KEYS` — maps `data-card-id` to the result fields tracked for change highlighting.

### `CARD_KEYS` (change-highlight map)
```
'RT':      ['RT_calc'],
'fSW_max': ['fsw_max', 'fsw_skip', 'fsw_shift'],
'RHS':     ['RHS_calc'],
'RLS':     ['RLS'],
'L':       ['L', 'I_rip', 'I_peak', 'I_rms', 'P_L'],
'diode':   ['P_D'],
'C_out':   ['C_out', 'C_min'],
'fp':      ['fp'],
'fz':      ['fz'],
'fco':     ['fco'],
'R4':      ['R4_calc', 'R4'],
'C5':      ['C5_calc', 'C5'],
'C8':      ['C8_calc', 'C8'],
```

## CSS layout

- Two-column root: `.side` (sidebar) + `#main` (results) + `#schem-panel` (schematic overlay, same grid column as `#main`, toggled via `display`).
- Results sections use `.g2` (2-col) and `.g3` (3-col) grids with `minmax(0,1fr)` to prevent overflow.
- `.card` has `min-width:0; overflow:hidden` to contain content.
- Compensation section is two separate `.g3` rows (fp/fz/fco, then R4/C5/C8).
- Highlight classes: `.card.changed` (fading amber animation) and `.card.updated` (persistent amber left border).
- `.stat.card` — operating-point tiles that also behave as cards: `text-align:left`, `padding:10px 12px`.
- Responsive break at `max-width:700px` collapses sidebar on top and grids to a single column.

## Operating point section

Four tiles at the top (`class="stat card"`), each with `data-card-id` and a hidden `fblock` revealed by `∑`:
- `op_D` — Duty cycle: `D = Vout / Vin(min)`.
- `op_eff` — Est. efficiency: `η = 1 − P_IC / (Vin(max) · Iout)`.
- `op_Ptot` — IC dissipation: Eq. 52–55 (`P_COND`, `P_SW`, `P_GD`, `P_Q`).
- `op_Pout` — Output power: `P_out = Vout · Iout`.

## Sidebar icon buttons

Horizontal row at the top of the sidebar:
- `∑` — toggle all formula blocks show / hide (turns blue when all shown).
- `↺` — reset all inputs to `DEFAULTS`, clear `OV`, clear localStorage.
- `⬓` (↓ arrow) — save config to JSON file.
- `⬒` (↑ arrow) — load config from JSON file (file picker).
- `▦` — **toggle the schematic view** (see below).

Each button has a hover tooltip (`.tip`).

## Schematic view

The second IIFE at the bottom of the file adds a full-page schematic view that replaces the results panel when toggled.

- SVG schematic lives inline in `#schem-wrap` inside `#schem-panel`. Drawn in Inkscape, 297×210 mm viewBox.
- `OVL` array positions HTML overlay labels (`.svl`) on top of the SVG as percentages of the viewBox. Each entry has `{id, ref, x, y, fmt, sub, ov}`:
  - `fmt(r)` — main value text.
  - `sub(r)` — small note underneath (typically "Override" or "Std: ...").
  - `ov(r)` — boolean, switches the sub text to amber when true.
- Components shown: `C1/C2` (Vin bulk), `C4` (bootstrap), `L1`, `C6/C7` (Vout), `D1` (catch diode), `R3` (RT), `R4`, `C5`, `C8` (compensation), `R5` (RHS), `R6` (RLS).
- `toggleSchematic()` swaps `#main` ↔ `#schem-panel` visibility and lights up the `▦` icon button (`#schem-tog.son`).
- The IIFE wraps the original `window.render` and `window.recalc` so that when the schematic is open:
  - `#main` stays `display:none` even after a re-render.
  - The overlay labels get refreshed via `_update()`.
- Build is lazy: labels are created on first open (`_built` flag), then only updated.

## Formula display

- Each card with formulas has a small 22×22 `∑` button injected into its `.clbl` header by `initFormulaToggles()`.
- Clicking toggles that card's `.fblock` elements between hidden and visible.
- State is persisted in `visibleFormulas` (a `Set` of `data-card-id`) across recalcs and across the render cycle.
- All fblocks are hidden by default on first load.
- Non-datasheet formulas use `'—'` as the equation number in `fb()`.

## Power dissipation cards

- **Inductor L1 card**: always-visible `cnote` `P_inductor = NNNmW`; hidden `fblock` (Eq. —) showing `P_inductor = I_rms² · Rdc` with substituted values.
- **Catch diode card** (`data-card-id="diode"`): always-visible `cnote` `P_diode = NNNmW`; hidden `fblock` (Eq. —) showing `P_diode = Vd · Iout · (1 − D)` with substituted values.
- **Operating-point IC dissipation tile**: `∑` reveals Eq. 52–55 stacked.

## Known issues / things to watch

- `#resr` HTML initial value is `0.25 mΩ` but `DEFAULTS.resr = '2.5'` — a fresh load shows 0.25 until the user touches anything, and ↺ reset jumps it to 2.5. Unify these if you care about first-load consistency.
- localStorage key is `'tps54360b_state'` — may persist old defaults until the user hits ↺.
- Datasheet uses `Vin(max)` for Eq. 9/10/28/29 and `Vin(min)` for Eq. 38 — the code follows this split.
- Eq. 47 uses `fSW/2` (not `fSW`) — geometric mean of `fp` and `fSW/2`.
- Eq. 48 formula: `R4 = 2π · fco · Cout · (Vout / VREF) / (gmPS · gmEA)`.
- Compensation uses the raw chosen `Cout` (no 0.85 derating baked into the calc); a derating advisory is shown in the `fp` card instead.
- IC dissipation uses `Vin_min` for `P_COND` (worst-case conduction) and `Vin_max` for `P_SW` / `P_GD` / `P_Q` (worst-case switching).
- `applyOverride` is referenced in one `saveState` hook (`_origApply = window.applyOverride`) but the function is never defined on `window`. The hook is a no-op; harmless but worth cleaning up if you're in there.
