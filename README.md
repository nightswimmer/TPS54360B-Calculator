# TPS54360B Component Calculator

A single-file, self-contained web app that designs the complete external component set for a Texas Instruments **TPS54360B** buck converter (60 V, 3.5 A, step-down DC/DC). Everything — HTML, CSS, JS, and the SVG reference schematic — lives inside one file you can open locally or host on GitHub Pages.

All equations are taken from TI datasheet **SNVSB93** and are labelled by equation number in the UI.

## Features

- **13 design inputs** with linked number box + slider pairs and live validation (Vin min/max, Vout, Iout, fSW, inductor DCR, diode Vf, output-cap ESR, current limit, short-circuit Vout, transient ΔVout / ΔIout, ripple factor).
- **Fully reactive** — no Calculate button. Sliders debounce at 50 ms; number fields recalc on Enter or blur.
- **Full component selection**:
  - RT (switching-frequency) resistor with max-fSW limit check (skip-mode and frequency-shift, Eq. 9/10).
  - Feedback divider RLS/RHS (Eq. 3).
  - Output inductor sizing and ripple / peak / rms currents (Eq. 28–31).
  - Output and input capacitor sizing (Eq. 32, 34, 38).
  - Full Type 2A compensation network — fp, fz, fco, R4, C5, C8 (Eq. 44–51).
  - IC power dissipation and estimated efficiency (Eq. 52–55), plus inductor copper loss and catch-diode loss.
- **Nearest-standard lookup** against E24/E96 resistors and E12/E24 capacitor & inductor tables.
- **Per-component overrides** (RLS, L, C_out, R4, C5, C8) with inline input fields that parse unit suffixes (`µF`, `nF`, `kΩ`, `mΩ`, …) and propagate through the whole dependency chain.
- **Formula inspection** — every card has a `∑` toggle that reveals the datasheet equation, substituted values, and the result. A global `∑` toggle opens or closes every card at once.
- **Change highlighting** — cards whose values actually changed since the previous recalc flash amber and keep a left-border marker until the next update.
- **Interactive schematic view** — toggle `▦` to swap the results panel for the reference schematic with live overlay labels on each component (R3, R4, R5, R6, C1/C2, C4, C5, C6/C7, C8, L1, D1). Overridden parts are flagged amber.
- **Save / load configuration** — export inputs and overrides to a JSON file, or reload an existing design.
- **Persistent state** — inputs and overrides are written to `localStorage` so your last design is there when you come back.
- **Light / dark theme** via `prefers-color-scheme`. Compact responsive layout down to 700 px.

## Getting started

No build, no dependencies, no server needed.

```
# Clone
git clone https://github.com/nightswimmer/TPS54360B-Calculator.git
cd TPS54360B-Calculator

# Open directly
start TPS54360B-Calculator.html    # Windows
open  TPS54360B-Calculator.html    # macOS
xdg-open TPS54360B-Calculator.html # Linux
```

Or just double-click the file. The only network fetch is the Google Fonts stylesheet (`JetBrains Mono`, `Syne`) — the app runs fine offline after the first load.

## Usage tips

- Start by setting your `Vin(max)`, `Vin(min)`, `Vout`, `Iout`, and `fSW`. The rest of the design falls out automatically.
- If a card shows an amber "EXCEEDS MAX" badge on `fSW`, drop your switching frequency — you are violating the minimum on-time or the frequency-shift foldback limit.
- The `∑` icon on any card opens the underlying equation with your numbers substituted in. The global `∑` in the sidebar toggles them all.
- To pin a specific off-the-shelf component, type its value into the override field next to the calculated value. The amber border is your reminder that the card is no longer auto.
- Click `↺` in the sidebar to return to defaults and clear every override and localStorage entry.
- The `▦` button swaps to a reference schematic where the same values are overlaid on their reference designators — handy for cross-checking a schematic while laying out a board.

## Project files

| File | Purpose |
| --- | --- |
| `TPS54360B-Calculator.html` | The full app — HTML + CSS + JS + inline SVG schematic. |
| `PROJECT_INSTRUCTIONS.md` | Long-form internal project state for future development sessions. |
| `CLAUDE.md` | Assistant instructions for working in this repo. |
| `LICENSE` | MIT. |

## Disclaimer

This calculator is a design aid. Always verify component values against the current TPS54360B datasheet (SNVSB93) and your own simulation / bench measurements before committing to a design. The author takes no responsibility for blown parts, singed fingers, or missed deadlines.

## License

MIT — see [LICENSE](LICENSE).
