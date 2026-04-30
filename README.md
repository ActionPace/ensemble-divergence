# ensemble-divergence

**Structural opacity scoring and ensemble divergence methodology for AI systems auditing.**

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/ActionPace/ensemble-divergence/blob/main/ensemble_divergence_demo.ipynb)

---

## What this is

This repository accompanies the working paper:

> **[Toward a Control Architecture for Composable AI Execution Systems](https://buildworlds.s3.us-east-1.amazonaws.com/king2026_control_architecture_draft_v02.pdf)**  
> David E. King · [ActionPace](http://actionpace.com) · April 2026  
> arXiv forthcoming · cs.SE / cs.AI

The methodology uses **ensemble divergence as a structural signal** — running multiple independent models against the same source artifact and treating disagreement, not just agreement, as meaningful information. Where models converge, the finding is high-confidence. Where they diverge, a composition seam has been located: a zone where implicit contracts are unclear and different training histories lead to different structural interpretations.

The methodology has been validated against:
- **Source code** — six production AI execution frameworks (AutoGen, CrewAI, LangGraph, n8n, OpenClaw, PydanticAI)
- **Research methods descriptions** — empirical social science papers, detecting underspecification before re-implementation

---

## Repository contents

| File | What it is |
|------|-----------|
| `ensemble_divergence_demo.ipynb` | Source code analysis demo — Apertus 8B vs Mistral 7B on LangGraph, with A100 sections for 70B and Nemo 12B |
| `ensemble_divergence_social_science.ipynb` | Methods description analysis demo — applied to Settele (2022), a verified failure case from Kohler et al. (2026) |
| `ensemble_divergence_social_science.py` | Python script version of the social science demo |
| `methods_opacity_prompts.py` | Adapted opacity prompts for natural language methods descriptions, with the Settele test case |

---

## The core idea

AI execution frameworks solve real problems — provider normalization, context threading, type-driven routing. But they also carry **orphaned invariants**: contracts that components depend on but that no single component declares ownership of. Callers must satisfy these contracts without being told they exist.

The ensemble divergence methodology detects these orphaned invariants before execution begins. Five dimensions are scored per artifact:

| Dimension | What it measures |
|-----------|-----------------|
| Undeclared preconditions | What callers must satisfy beyond type signatures |
| Global state risk | Hidden shared mutable state |
| Implicit coupling | Undeclared dependencies on the environment |
| Contract completeness | How fully contracts are declared (inverted — 0 = fully declared) |
| Structural centrality | How much system correctness depends on this artifact invisibly |

Where multiple independent models name the same contract, it is treated as a confirmed finding. Where they disagree, the disagreement locates a structural boundary worth investigating.

---

## Key finding — LangGraph pregel/main.py

Four models across two training lineages analyzed `libs/langgraph/langgraph/pregel/main.py`:

| Model | Score | Training lineage |
|-------|-------|-----------------|
| Apertus 8B (Q5_K_M) | 78/100 HIGH | Swiss / Llama-family |
| Mistral 7B (Q5_K_M) | 60/100 HIGH | French / Mistral arch. |
| Apertus 70B (Q5_K_S) | 61/100 HIGH | Swiss / Llama-family |
| Mistral Nemo 12B (Q5_K_M) | 65/100 HIGH | French / Mistral arch. |

**4× convergent findings** (all four models agree):
- `structural_centrality`: 18/20
- `undeclared_preconditions`: 12/20

**3× confirmed load-bearing functions** (named by 3+ models):
`get_subgraphs`, `aget_subgraphs`, `get_input_schema`, `get_input_jsonschema`, `get_output_schema`, `get_output_jsonschema`

**The depth-of-read finding** — Apertus 70B names a specific orphaned invariant that all three smaller models miss:
> *"The `get_subgraphs` method assumes that subgraphs are stored in a nested structure with namespaces separated by `NS_SEP`, which is not declared in the type hints."*

This is the methodological point: the most valuable signal is not where models agree, but where a larger or differently-trained model finds what the others read past. That is where the orphaned invariants live.

---

## Extension — methods description analysis

The same methodology applies to empirical research methods descriptions. Applied to Settele (2022) — a verified failure case from Kohler et al. (2026) *"Read the Paper, Write the Code"* — both Apertus 8B and Mistral 7B independently flag the party identification coding as the primary ambiguity zone:

> **Mistral 7B primary signal:** *"The specific coding of the party identification variable"*

This is the exact failure node documented in Kohler et al. Table 3: agents tasked with reproducing the paper fail because the methods description does not specify which survey codes map to "Democrat." Both models locate this ambiguity before any re-implementation attempt.

The methodology detects underspecification where it lives — in the artifact — rather than discovering it through reproduction failure downstream.

---

## Running the demos

### Requirements

- Google Colab (free T4 tier for Section 1; any paid plan for A100 sections)
- HuggingFace account with access to [Apertus on the ActionPace profile](https://huggingface.co/actionpace)
- HF token added to Colab Secrets as `HF_TOKEN`

### Source code demo (Section 1 — free tier)

```
1. Open ensemble_divergence_demo.ipynb in Colab
2. Add HF_TOKEN to Colab Secrets (key icon, left sidebar)
3. Run Section 0 (hardware detection)
4. Run Section 1 cells in order — downloads ~10GB of models on first run
```

Sections 2 and 3 require A100 and will skip gracefully on T4.

### Methods description demo

```
1. Open ensemble_divergence_social_science.ipynb in Colab
2. Add HF_TOKEN to Colab Secrets
3. Run all cells — Section 1 runs on free T4
```

The Settele (2022) methods text is embedded in the notebook. No external data required.

---

## Models used

| Model | Source | Access |
|-------|--------|--------|
| Apertus 8B Instruct 2509 (Q5_K_M) | [unsloth/Apertus-8B-Instruct-2509-GGUF](https://huggingface.co/unsloth/Apertus-8B-Instruct-2509-GGUF) | Gated — request via ActionPace HF profile |
| Apertus 70B Instruct 2509 (Q5_K_S) | [unsloth/Apertus-70B-Instruct-2509-GGUF](https://huggingface.co/unsloth/Apertus-70B-Instruct-2509-GGUF) | Gated — request via ActionPace HF profile |
| Mistral 7B Instruct v0.3 (Q5_K_M) | [bartowski/Mistral-7B-Instruct-v0.3-GGUF](https://huggingface.co/bartowski/Mistral-7B-Instruct-v0.3-GGUF) | Public |
| Mistral Nemo Instruct 2407 (Q5_K_M) | [bartowski/Mistral-Nemo-Instruct-2407-GGUF](https://huggingface.co/bartowski/Mistral-Nemo-Instruct-2407-GGUF) | Public |

Apertus is developed by the [Swiss AI Initiative](https://www.swiss-ai.org/) (EPFL, ETH Zürich, CSCS) — a fully open model trained on compliant data with open weights, open training details, and EU AI Act alignment. Mistral models are from [Mistral AI](https://mistral.ai/) (France).

The pairing is deliberate: two architecturally independent model families from different countries and training lineages. Structural findings that converge across both are robust to model-specific training biases.

---

## Citation

```bibtex
@misc{king2026control,
  title={Toward a Control Architecture for Composable AI Execution Systems},
  author={King, David E.},
  year={2026},
  note={arXiv forthcoming · cs.SE / cs.AI},
  institution={ActionPace}
}
```

---

## Related work

- Kohler, Zollikofer, Einsiedler, Hoyle & Ash (2026). *Read the Paper, Write the Code: Agentic Reproduction of Social-Science Results.* arXiv:2604.21965
- Fernandez (2026). *From Admission to Invariants: Measuring Deviation in Delegated Agent Systems.* arXiv:2604.17517

---

**ActionPace · [third-place.space/ai](http://third-place.space/ai) · April 2026**
