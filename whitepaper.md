# DRT_Scanner — the inversion-fidelity diagnostic

> **Humble disclaimer.** Amateur engineering project. We are not HPC professionals and make no competitive claims. Errors are likely. The scanner is a *diagnostic*, not a guarantee — see [What is not in the paper](#not).

GPLv3.

---

## Contents

- [What it is](#what)
- [Why it matters — bio digital twins](#why)
- [How it works — the transpose pass](#how)
- [Reading the output — agreement and divergence](#reading)
- [What is not in the paper](#not)
- [References and the trio](#refs)

---

## <a id="what"></a>What it is

A browser demo that runs a chain of sparse matrix-vector multiplications in *reverse*. The companion repository [drt-generator](https://github.com/norayr-m/drt-generator) runs a forward chain — clock tick to harmonic waveform, through seven columns of sparse and dense operations. The scanner runs the same machinery backwards: it takes a target output, projects it back through the transposes of the same matrices in reverse order, and reports column-by-column whether the reverse pass agrees with what the forward pass would have produced.

The two repos are deliberately released as a pair. The generator demonstrates the encoder shape — how a signal is built by composing sparse projections. The scanner demonstrates the inspection of that encoder shape — whether the composition can be cleanly inverted at a given configuration. Together they walk a complete forward-and-backward cycle that is the same shape as automatic differentiation in modern numerical libraries and message-passing on transpose graphs in graph neural networks.

**[Live demo →](https://norayr-m.github.io/drt-scanner/)**

---

## <a id="why"></a>Why it matters — bio digital twins

A bio digital twin is a computational model of a specific biological system — a heart chamber, a hepatic lobule, a section of cortex — wired tightly enough that it can predict the system's response to a stimulus. Building a twin that *predicts* is hard. Building a twin that *retrodicts* — that answers *what stimulus produced this measured tissue state?* — is harder, and that is exactly the inverse-modelling question that medical imaging asks every day.

The scanner is a step toward that question on a small enough example to fit in a browser tab. It does not solve the full inverse problem. It does something more modest: it tells you, at every column of the forward chain, whether the reverse pass agrees with the forward pass at the current configuration. A column that agrees is a step that is locally invertible at this state. A column that diverges is a step where information has been lost or where numerical conditioning has broken down.

For the bio digital twin user, this is the difference between a model that can confidently attribute a measured tissue state to a known stimulus and a model that cannot. The scanner gives a per-configuration map of *which regions of input space the twin is reliable for, and which regions it is not*. That information is more useful than a single yes/no answer to the global question.

The framing for the trio (generator + cell simulator + scanner) is the [trio white paper](https://norayr-m.github.io/drt-generator/whitepaper.html). This paper drills into the scanner specifically.

---

## <a id="how"></a>How it works — the transpose pass

The forward chain produced by the generator, in compact form:

```
y = M · σ( W · x )    with    x = sparse_select( S, p )    and    p = LUT( t )
```

The clock tick `t` is sent through a lookup table to a pattern identifier `p`. A one-hot selection matrix `S` picks the corresponding row, producing a sparse weight vector `x`. An element-wise nonlinearity `σ` and a small dense propagation matrix `W` produce an intermediate vector. An output mixing matrix `M` produces the per-tick observable `y`.

The scanner runs the inverse chain. Take a target observable `y` and project it back:

```
x̂ = S^T · σ^{-1}( W^T · M^T · y )
```

Apply the transpose of the output mixing, the inverse of the activation, the transpose of the dense propagation, and the transpose of the selection. The result `x̂` is the reverse pass's reconstruction of the original sparse weight vector. If `x̂ ≈ x`, the chain is locally invertible at this configuration. If `x̂` diverges from `x` at any column, the chain has lost information at that step.

Visually, the demo lays the two passes out as mirror images: the generator's columns flow left-to-right at the top, the scanner's columns flow right-to-left at the bottom, and a comparison link between every matched column lights green for agreement or red for divergence.

```
forward →
[phase] → [index] → [select] → [sparse vec] → [σ] → [W] → [M] → y
   |         |         |            |          |     |     |
compare   compare   compare      compare    compare compare compare
   |         |         |            |          |     |     |
[phase] ← [index] ← [select] ← [sparse vec] ← [σ⁻¹] ← [W^T] ← [M^T] ← y
                                                                ← reverse
```

(The diagram inside the live demo is the visual rendering of the same idea.)

For the math reader: the per-column "agreement" check is a forward-then-backward composition that, in the linear case, would be `S^T · S = I` and `W^T · W^{-1} = I` etc. — the chain reconstructs the input exactly when each step is a unitary or pseudo-inverse. In practice the matrices are not unitary; the "agreement at column k" condition becomes the empirical equivalent: forward pass at column k matches reverse pass at column k within tolerance.

---

## <a id="reading"></a>Reading the output — agreement and divergence

The scanner displays each column with a per-column traffic-light state. Three things this lets you eyeball:

- **Reconstruction quality.** How closely the reverse pass matches what the forward pass would have produced. Green columns are columns where reconstruction is faithful; red columns flag divergence.
- **Hallucination boundaries.** Points at which the reverse-projected signal predicts content that the forward pass did not actually emit. These are the "imagined structure" boundary — the configurations where the reverse pass is making something up rather than recovering what was originally there.
- **Convergence.** Whether, given enough ticks, the reverse-pass output stabilises at the original input or wanders.

You can pause the scanner, mute the audio, and step through phase-by-phase. The three signals (reconstruction, hallucination, convergence) are the three shapes of feedback the diagnostic returns.

---

## <a id="not"></a>What is not in the paper

The honest list of what this paper deliberately does not claim:

- **A general inversion theorem.** The scanner is a *per-configuration* diagnostic. It tells you whether the chain is locally invertible *at this state*. It does not tell you that any specific input region is globally invertible. Bio digital twin users who need rigorous inverse-problem theory will need additional machinery (Tikhonov regularisation, posterior sampling, Bayesian inverse methods) layered on top of what the scanner provides.
- **A norm-growth claim.** Earlier prose around this family of demos asserted that the aggregate of many partial views had bigger norm than the original signal. That inequality has been retracted in the v0.1 draft. The scanner's current claims are mechanical (these matrices, these transposes, this composition) not analytical.
- **A speedup benchmark.** The scanner does not race against optimised inverse-problem solvers. It is a pedagogical demonstration of the forward-backward symmetry on a small enough example to fit in a browser tab. Real bio-digital-twin inversion at tissue scale would use specialised numerical libraries on top of GPU-resident sparse-mat-vec.

---

## <a id="refs"></a>References and the trio

This repository is one of three that snap together into one workflow:

1. **[drt-generator](https://github.com/norayr-m/drt-generator)** — produces sparse projection matrices from a clock tick.
2. **[drt-cell-simulator](https://github.com/norayr-m/drt-cell-simulator)** — receives the projections as cellular automata on graph topology. Each cell is a *receptor* with local rules over its neighbours' edges.
3. **drt-scanner** (this repo) — runs the transpose pass for inversion-fidelity verification.

End to end is *generator → cell simulator → scanner* — forward pass plus backward pass through the same matrices. Every step is sparse matrix-vector multiplication.

Trio framing in detail: **[Generator, Cell Simulator, Scanner — a sparse matrix-vector trio for bio digital twins](https://norayr-m.github.io/drt-generator/whitepaper.html)** ([markdown source](https://norayr-m.github.io/drt-generator/whitepaper.md)).

Distributed Reconstruction work — v0.1 in preparation by N. Matevosyan and A. Petrosyan.

Visualization co-authored with Claude (Anthropic).

---

> **Humble disclaimer.** Amateur engineering project. We are not HPC professionals and make no competitive claims. Numbers speak; ego doesn't. Errors likely.

GPLv3. See [`LICENSE`](LICENSE).
