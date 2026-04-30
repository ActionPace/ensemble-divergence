# Ensemble Divergence Applied to Research Methods Descriptions

## Extension: Detecting Underspecification Before Re-implementation

This document reports findings from applying the ensemble divergence methodology
to empirical research methods descriptions — a domain extension beyond the
source code analysis in the main paper.

The central question: can ensemble divergence identify underspecification zones
in a research methods description *before* any re-implementation attempt is made?

---

## Background

Kohler, Zollikofer, Einsiedler, Hoyle & Ash (2026) — *"Read the Paper, Write
the Code"* (arXiv:2604.21965) — evaluated LLM agents tasked with reproducing
social science results from methods descriptions and original data alone, without
access to the original code.

Their key finding: the primary barrier to reproduction is not agent capability
but **paper underspecification** — methods descriptions that omit implementation
details, forcing agents to infer choices the original authors made implicitly.
Table 3 of that paper documents specific verified failure cases, including:

> *Settele (2022), Table 2: The paper does not disclose which specific categorical
> survey codes (A1–A5) indicate "Democrat" support. Agents must infer the mapping.
> Incorrect inferences (A1/A2 = Democrat) produce grade D reproduction;
> correct inferences (A4/A5 = Democrat) produce grade B.*

This is structurally identical to an **orphaned invariant** in source code:
a contract that the system depends on but that no component declares ownership of.
The methods description is the artifact; the re-implementer is the caller;
the undeclared party coding convention is the orphaned invariant.

---

## The Adapted Methodology

The opacity scoring methodology was adapted from source code to methods descriptions
by replacing the five code-specific dimensions with five specification-specific ones:

| Original (source code) | Adapted (methods description) |
|------------------------|-------------------------------|
| Undeclared preconditions | Undeclared preconditions |
| Global state risk | Software default risk |
| Implicit coupling | Implicit data coupling |
| Contract completeness (inverted) | Contract completeness (inverted) |
| Structural centrality | Result centrality |

**Software default risk** captures the analog of hidden shared state in code:
reliance on software-specific defaults (e.g., which F-statistic Stata uses,
which standard error variant R defaults to) that are not declared in the methods text.

**Implicit data coupling** captures undeclared dependencies on the raw data structure —
column names, coding conventions, categorical mappings that only exist in the
codebook, not in the paper.

**Result centrality** captures how load-bearing the underspecified elements are:
does guessing wrong on an undeclared choice change the sign or significance of
a primary result, or only affect a robustness check?

---

## Test Case: Settele (2022)

**Paper:** *How Do Beliefs about the Gender Wage Gap Affect the Demand for
Public Policy?* — Sonja Settele, AEJ: Economic Policy (2022)

**Why this paper:** Kohler et al. Table 3 documents it as a verified failure case
with a known root cause. The party identification coding ambiguity (A4/A5 = Democrat)
is confirmed ground truth — making it an ideal test of whether the methodology
can detect the failure zone in advance.

**Ensemble:** Four models across two training lineages.

| Model | Score | Training lineage |
|-------|-------|-----------------|
| Apertus 8B Instruct 2509 (Q5_K_M) | 60/100 HIGH | Swiss / Llama-family |
| Mistral 7B Instruct v0.3 (Q5_K_M) | 48/100 MEDIUM | French / Mistral arch. |
| Apertus 70B Instruct 2509 (Q4_K_M) | 60/100 HIGH | Swiss / Llama-family |
| Mistral Nemo Instruct 2407 (Q5_K_M) | 57/100 MEDIUM | French / Mistral arch. |

---

## Primary Finding: 4× Convergence on Party Identification Coding

All four models independently flagged the party identification coding as the
primary structural signal:

```
[MATCH] Apertus 8B      "The ANES five-point scale is not fully specified,
                         and the coding of party identification may differ
                         across software environments."

[MATCH] Mistral 7B      "The specific coding of the party identification
                         variable"

[MATCH] Apertus 70B     "The exact coding of the party identification variable
                         (e.g., whether 'lean' categories are included or
                         excluded) is not specified."

[MATCH] Mistral Nemo    "The exact coding scheme for the party identification
                         variable is not specified."
```

**Confirmed at 4× confidence** — all four models, two training lineages,
independently located the same underspecification zone. This meets and exceeds
the paper's 3× confirmation threshold for high-confidence findings.

This is the exact failure node documented in Kohler et al. Table 3.
The methodology detected it **before any re-implementation attempt**,
from the methods description alone.

---

## Ambiguity Zone Counts

Beyond the primary signal, the ensemble identified a consistent set of
secondary ambiguity zones:

```
3x  party identification coding
3x  policy support index construction
2x  treatment assignment specification
2x  software defaults
1x  clustering / standard errors
1x  data filtering / missing values
```

The **3× confirmation on policy support index construction** is a secondary
finding worth noting: three of four models independently flagged the
construction of the outcome variable (average of five items — but which
items? weighted or unweighted?) as underspecified. This is not documented
as a failure node in Kohler et al. but represents an additional orphaned
invariant that a detection system would surface for human review.

---

## The Depth-of-Read Pattern

The 70B model produces the most specific primary signal and the most detailed
implementer burden analysis, with five UNSTATED items including treatment
wording, control variable measurement, and index construction method.

The 8B-class models (Apertus 8B, Mistral 7B, Mistral Nemo 12B) identify
the party coding ambiguity but characterize the implementer burden as
ALL STATED. The 70B model reads deeper and finds five specific unstated
requirements.

This replicates the depth-of-read pattern from the source code analysis
(Appendix A of the main paper): larger models find orphaned invariants
that smaller models read past. The pattern holds across domains.

---

## Quantization Effects: Q4_K_M vs Q5_K_S

The Apertus 70B model was run at two quantization levels across separate
sessions:

| Run | Quantization | Hardware | Primary signal |
|-----|-------------|----------|----------------|
| Run 1 | Q4_K_M (partial layer offload) | A100 40GB | *"whether 'lean' categories are included or excluded"* |
| Run 2 | Q5_K_S (full load) | A100 80GB | *"whether 'strong Democrat' is coded as 1 or 2 in the ANES scale"* |

Both runs converge on party identification coding as the primary signal.
The core finding is robust to quantization level.

The Q5_K_S run produces a more actionable signal — naming the specific
numeric coding question (1 or 2 in the scale) rather than the category
inclusion question. A researcher reading the Q5_K_S output knows exactly
what to look for in the codebook; the Q4_K_M output requires an additional
inference step to arrive at the same lookup.

The Q5_K_S run also shows higher consistency on software default risk
across models (3× vs 2× in ambiguity zone counts), suggesting the
higher-quality quantization surfaces that dimension more reliably.

**Practical implication:** For production use of this methodology,
Q5_K_S or equivalent full-quality quantization is preferred for
the 70B model. The Q4_K_M result is still directionally correct
and suitable for demonstration on standard A100 hardware.

---

## Methodological Interpretation

The ensemble divergence methodology detects underspecification where it
lives — in the artifact — rather than discovering it through reproduction
failure downstream.

Applied to the Settele methods description, the methodology would function
as a **pre-submission audit**: run the ensemble against the methods section
during peer review, surface the ambiguity zones, and give the authors the
opportunity to declare the party coding convention before the paper is
published. The same orphaned invariant that caused documented reproduction
failures would be flagged — and potentially resolved — before it enters
the scientific record.

This is the pre-execution governance framing from the main paper applied
to a new domain: composition seam governance for research methods
descriptions, where the "composition seam" is the boundary between what
the paper specifies and what the re-implementer must infer.

---

## Relationship to the Main Paper

The main paper's theoretical model proposes that implicit contracts should
be **surfaced, owned, and testable** rather than silently assumed. In source
code, this means naming the orphaned invariant and giving it a declared home
in an interface. In research methods descriptions, this means declaring the
party coding convention, the index construction method, and the software
defaults in the methods section rather than leaving them to the reader's
inference.

The detection methodology is the same in both domains. The artifact differs
(source code vs. methods description), the consumer differs (calling code vs.
re-implementing agent or researcher), but the structural problem is identical:
a contract that the system depends on but that no component declares.

---

## Reproduction

The full four-model ensemble run is demonstrated in:

| File | Links | What it is |
|------|-------|-----------|
| `ensemble_divergence_social_science.ipynb` | [Colab](https://colab.research.google.com/github/ActionPace/ensemble-divergence/blob/main/ensemble_divergence_social_science.ipynb) | Methods description analysis — applied to Settele (2022), a verified failure case from Kohler et al. (2026) |

The Settele methods text is embedded in the notebook. No external data or
API access to the original paper is required. The ground truth (A4/A5 = Democrat)
is documented in comments alongside the methods excerpt, verified against
Kohler et al. (2026) Table 3.

---

## Reference

Kohler, B., Zollikofer, D., Einsiedler, J., Hoyle, A., & Ash, E. (2026).
*Read the Paper, Write the Code: Agentic Reproduction of Social-Science Results.*
arXiv:2604.21965 [cs.AI]
