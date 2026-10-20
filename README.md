# HSDE — hybrid symbolic discovery engine

Notebook prototypes for reasoning systems that sit between neural and symbolic methods. Two lines of work: a cellular-automaton-plus-transformer stack where I mine symbolic rules out of a causal graph built from attention, and a pipeline that turns a local LLM's informal proof sketch into SymPy expressions a verifier can check.

These are exploratory prototypes. The modules run and the intermediate artifacts in this repo are real outputs, but there is no benchmark evaluation here: the GSM8K-style harness in `esde.ipynb` was only ever pointed at three hand-written arithmetic problems, and the AIME / SAT targets described in the design cells were never run.

Project page: https://rohitravi.com/projects/hsde

## ASI.ipynb — rules mined from attention over a cellular automaton

The idea I wanted to test is that attention weights are not themselves an explanation, but a causal graph derived from them might be a substrate you can do symbolic discovery on. The notebook builds it in stages:

1. A 2D cellular automaton as a controllable dynamical system, with pattern generation and visualization.
2. A transformer encoder trained on CA history sequences with hooks that expose attention. Training runs 100 epochs on CUDA; the loss trace is in the notebook.
3. Attention tensors averaged across heads, thresholded adaptively at mean + 0.5 sigma into a binary causal matrix of shape [4, 20, 20] (heads, timesteps, timesteps), then turned into a temporal causal graph with networkx and mined for frequent patterns with apriori / FP-growth from mlxtend.
4. Sketches of recursive self-modification and an evaluation section, both unfinished.

What exists: the pipeline runs end to end up through rule mining on the causal graph, with heatmaps and graph plots in the notebook. What does not: a controlled comparison against a transformer-only baseline, which is the experiment that would make this worth writing up.

## phase1 / phase2 / esde / esde2 — proof sketch to verified symbolic steps

A chain of prototypes for the same loop: ask a local model (Mistral or Phi-3 through Ollama) for a structured proof sketch, parse it into SymPy, check it, and feed failures back.

- `phase1.ipynb` is the first version: prompt for a sketch, score it, save JSON.
- `phase2.ipynb` is the full six-phase version: sketch generation with JSON repair, a verifier with base-case and inductive-step checks, self-refinement over up to five attempts, rule mutation to explore nearby formulations, a contrastive evolver that picks the best variant, a memory graph that hashes proven sketches and extracts templates to suggest prior methods for new tasks, and a symbolic AST compiler.
- `esde.ipynb` and `esde2.ipynb` reorganize this into named components: `SketchNegotiator`, `RuleEngine`, `Verifier` (SymPy, with z3 wired in for integer constraint checks), `Evolver`, `SymbolicEmbedding` (CodeBERT), `MemoryGraph` (FAISS plus FP-growth), `FeedbackLoop`, and an `ESDE` class that wires them together. Each has a sanity check cell with its output kept in the notebook.

The verifier does real work. On the sum-of-odd-numbers identity it accepts `Eq(Sum(2*i-1,(i,1,n)), n**2)` with loss 0.0 and rejects the perturbed `n**2+1` with loss 1.0. The failure mode is the other direction: in `esde2.ipynb` the rule engine hands the verifier junk (`Eq(3, 4)`), and the sketch-to-expression step is the weak link in the whole pipeline.

## Artifacts in the repo

- `hsde_phase1_output.json` — a Mistral-generated induction proof of the sum of the first n odd numbers, each step carrying a SymPy expression.
- `hsde_phase6_output.json`, `hsde_phase1_output_phase6_ast.json` — the same problem after the AST compiler pass; most steps still have `ast: null`, which is exactly the gap described above.
- `hsde_memory/<hash>.json` — one stored sketch (sum of the first n even numbers) from the memory-graph phase.
- `*.pdf` — rendered copies of each notebook, so the outputs and figures are readable without running anything.

## Running the notebooks

The ESDE notebooks expect a local Ollama server with `mistral` (and optionally `phi3`) pulled, and install their own Python dependencies in the first cell: torch, sympy, z3-solver, transformers, faiss, mlxtend, networkx, seaborn. `ASI.ipynb` wants a GPU for the transformer training, though it falls back to CPU. Nothing here needs an API key or private data.

## Status

Prototypes, not a system. The architecture boundaries are clean and the individual pieces behave, so the natural next step is the narrow one: take ASI phases 1 to 3B, run it against a transformer-only baseline on the same CA task, and see whether the mined rules predict anything the baseline misses.
