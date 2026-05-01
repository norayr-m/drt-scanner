# DRT Scanner — the 7-Column Inspection Pair

> This is an amateur engineering project. We are not HPC professionals and make no competitive claims. Errors likely.

The dual of the Generator. Where the Generator runs a signal *forward* through seven columns (phase → lookup → routing → weights → activation → hidden layers → wave assembly), the Scanner runs the same machinery *backward* — the signal flows from a master output back through the columns, exposing where the reconstruction is faithful and where it is not.

**[Live demo →](https://norayr-m.github.io/drt-scanner/)**

## What it does

The seven columns are the same ones used in the Generator, but the wires now flow from right to left:

- A "master" signal sits in the right panel.
- At each tick, that master signal is projected back through the columns.
- The intermediate values (Phase, Pattern ID, Row, Vector Weights, …) are read off and shown in the left panel.
- Where the reverse pass agrees with the forward pass, the columns light up green; where they diverge, the columns flag the divergence.

Three things this lets you eyeball:

1. **Reconstruction quality** — how closely the reverse pass matches what the forward pass would have produced.
2. **Hallucination boundaries** — points at which the reverse-projected signal predicts content that the forward pass did not actually emit. This is the "imagined structure" boundary.
3. **Faithful-completion convergence** — whether, given enough ticks, the reverse-pass output stabilises at the original input.

You can pause the scanner, mute the audio, and step through phase by phase.

## Why it matters (DRT angle)

This is the **inspection slice** of the framework — the dual to the Generator's encoder slice. The honest interesting object in the Distributed Reconstruction work is not just the aggregate $R_t(f)$ itself; it is the comparison between $R_t(f)$ and what you can compute from $f$ alone. The Scanner is the visual playground for that comparison on the toy 7-column system.

Two specific framework concerns are visible here:

1. **Approximate faithfulness.** The framework asks each return operator $C_i$ to satisfy something like $\Pi_i \circ C_i \circ \Pi_i \approx \Pi_i$ — that the reverse projection through $C_i$, when re-projected, matches the original projection. The Scanner shows that condition empirically on the 7-column system.
2. **Hallucination as deviation.** The same growth mechanism that lets the aggregate add real new structure also lets it add fabricated structure. The Scanner is the dial for spotting the fabricated end of that spectrum on this toy system.

The honest current claim from the v0.1 draft is conditional and weaker than some earlier prose suggested. The earlier "$\|R(f)\| > \|f\|$" inequality has been retracted; the current claim is about exposure of computational features under a bounded budget. The Scanner is a small empirical playground for those words — not a proof of them.

## How to run

Single HTML, open in a browser:

```
git clone https://github.com/norayr-m/drt-scanner.git
open drt-scanner/index.html
```

No build step. No server. No external dependencies beyond a modern browser. Audio is opt-in.

## The trio — generator, cell simulator, scanner

This repo is one of three that snap together into one workflow:

1. **[drt-generator](https://github.com/norayr-m/drt-generator)** — produces sparse projection matrices from a clock tick.
2. **[drt-cell-simulator](https://github.com/norayr-m/drt-cell-simulator)** — receives the projections as cellular automata on graph topology. Each cell is a *receptor* with local rules over its neighbours' edges.
3. **drt-scanner** (this repo) — runs the transpose pass: projects a target signal back through the transposes of the same matrices, reports column-by-column whether the reverse pass agrees with the forward pass.

End to end is *generator → cell simulator → scanner* — forward pass plus backward pass through the same matrices. Every step is sparse matrix-vector multiplication. The applied target is **bio digital twins** at tissue scale: the scanner's column-by-column inversion check is the honest answer to the inverse-modelling question — *can I attribute a measured tissue state to a known stimulus?* — telling you which regions of input space the digital twin is reliable for, and which it is not.

This is a *diagnostic*, not a guarantee. It tells you at which step in the forward chain the inversion fails on a given configuration; it does not deliver a global inversion theorem. Bio digital twin users who need rigorous inverse-problem theory will need additional machinery (Tikhonov regularisation, posterior sampling, etc.) layered on top.

Full write-up: **[Generator, Cell Simulator, Scanner — a sparse matrix-vector trio for bio digital twins](https://norayr-m.github.io/drt-generator/whitepaper.html)** ([markdown source](https://norayr-m.github.io/drt-generator/whitepaper.md)).

## Companion repository

- **DRT_Generator** — the forward direction of the same 7-column pipeline. Read both side-by-side to see the symmetry.

## References

- Distributed Reconstruction work — v0.1 in preparation, N. Matevosyan and A. Petrosyan.

Visualization co-authored with Claude (Anthropic).

## Author

Norayr Matevosyan

## License

GPLv3
