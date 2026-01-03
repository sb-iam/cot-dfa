# CoT-DFA: Chain-of-Thought Dataflow Analysis

**Applying Compiler Reaching Definitions to Detect Unfaithful Reasoning in Code Generation**

---

## Results Summary

**H₁ SUPPORTED:** Phantom ratio (code without CoT justification) correlates negatively with execution success.

| Metric | r | p | Cohen's d | Direction |
|--------|---|---|-----------|-----------|
| Phantom Ratio | −0.202 | 0.013* | −0.46 | ↑ → failure |
| Dead Ratio | −0.210 | 0.010* | −0.48 | ↑ → failure |
| Faithfulness | +0.250 | 0.002** | +0.58 | ↑ → success |
| Semantic Coherence | +0.206 | 0.012* | +0.47 | ↑ → success |

*n=150 samples from OpenThoughts-114k (DeepSeek-R1). All bootstrap 95% CIs exclude zero.*

---

## Research Question

**Can compiler-style reaching definitions analysis detect unfaithful Chain-of-Thought in code generation models?**

---

## The Idea

In compilers, **reaching definitions** answers: *"For each use of variable x, which definitions could have produced the value?"*

I apply this to CoT → Code:

| Compiler Concept | CoT-DFA Mapping |
|------------------|-----------------|
| Variable definition | CoT segment introducing a concept |
| Variable use | Code element using that concept |
| Use without definition | **PHANTOM** — code without reasoning |
| Dead code | CoT steps not reaching any output |

### Example

**CoT Trace:**
```
s1: "I'll use a hash map for O(1) lookup"
s2: "Handle the edge case of empty input"  
s3: "Add some comments for clarity"
```

**Code Output:**
```python
def solve(nums):
    seen = {}           # ← s1 reaches (hash map → dict)
    if not nums:        # ← s2 reaches (edge case)
        return -1
    for n in nums:
        seen[n] = True
    return max(seen)    # ← PHANTOM! (not discussed in CoT)
```

**Analysis:**
- `seen = {}` ← s1 reaches (hash map → dict) ✓
- `if not nums:` ← s2 reaches (edge case) ✓
- `return max(seen)` ← **PHANTOM** (no CoT justification)
- s3 is **DEAD** (reaches nothing)

**Result:** phantom_ratio = 1/4 = 0.25

**Key insight:** Code elements without any reaching CoT segment are *phantoms*—potentially post-hoc rationalization.

---

## Results

### Primary Finding

Successful code has **lower phantom ratios** than failing code:
- Success mean: μ = 0.247
- Failure mean: μ = 0.363  
- Bootstrap 95% CI for difference: [−0.202, −0.029] (excludes zero)

### Key Statistics

**Point-biserial correlations with execution success:**

```
phantom_ratio:      r = -0.202, p = 0.013*,  d = -0.46
dead_ratio:         r = -0.210, p = 0.010*,  d = -0.48
faithfulness:       r = +0.250, p = 0.002**, d = +0.58
semantic_coherence: r = +0.206, p = 0.012*,  d = +0.47
```

All four metrics significant at p < 0.05. All bootstrap 95% CIs exclude zero.

---

## Methodology

### Data
- **Source:** OpenThoughts-114k (DeepSeek-R1 CoT traces)
- **Domain:** Competitive programming (TACO, APPS, CodeContests)
- **Sample:** n=150 (42 success, 108 failure)
- **Ground truth:** Execution success (code runs without error)

### Pipeline
1. **Segment CoT** into reasoning steps (sentence boundaries + numbered steps)
2. **Parse code** to AST, extract significant nodes (assignments, calls, returns, loops, conditionals)
3. **Tag both** with 22-concept vocabulary:
   - Data structures: dict, list, set, stack, queue, heap, tree, graph
   - Algorithms: sort, search, recursion, dp, greedy, two_pointer, backtrack
   - Control flow: loop, condition, early_return
   - Operations: count, sum, max_min, string_op
4. **Compute reaching sets** via tiered matching:
   - Tier 1: ≥2 shared concepts (IDF-weighted)
   - Tier 2: 1 rare concept (IDF ≥ 2.0) + semantic threshold 0.20
   - Tier 3: Any shared concept + UniXcoder cosine ≥ 0.75

### Metrics

```
phantom_ratio = |{code elements with no reaching CoT}| / |code elements|

dead_ratio    = |{CoT segments reaching nothing}| / |segments|

faithfulness  = 0.7 × (1 - phantom_ratio) × (1 - 0.5 × dead_ratio) + 0.3 × semantic_sim
```

### Statistics
- Point-biserial correlation (continuous vs binary outcome)
- Mann-Whitney U test
- Bootstrap 95% CIs (9,999 resamples, percentile method)
- Cohen's d effect sizes

---

## Limitations

1. **Weak ground truth:** "Success" = executes without error. Only 1/150 had functional tests. Conflates "runs" with "correct."

2. **Single dataset/model:** OpenThoughts-114k + DeepSeek-R1 only. Generalization to other models/domains untested.

3. **Coarse vocabulary:** 22 concepts is a proxy. Finer semantic parsing might catch subtler unfaithfulness.

4. **Structural ≠ causal:** High phantom *suggests* unfaithfulness but doesn't *prove* it. Would need ablation to confirm causality.

5. **Modest effect sizes:** d ≈ 0.46–0.58. Other factors clearly dominate task success.

### Future Work
- Human faithfulness annotations (gold standard)
- Functional test suites (HumanEval/MBPP-style)
- Shuffle-control: randomize CoT↔code pairing, verify correlation collapses
- Covariate control: LOC, #segments as confounds
- Multi-model replication
- Combine with causal methods (Thought Anchors) for structural + causal validation

---

## Comparison with Related Work

### CoT-DFA vs Thought Anchors

| | Thought Anchors | CoT-DFA |
|---|----------------|---------|
| **Question** | "Which sentences matter causally?" | "Is this a valid derivation?" |
| **Method** | Counterfactual perturbation | Structural analysis |
| **Cost** | O(n) forward passes per sample | O(1) single-pass parse + match |
| **Detects** | Important sentences | Phantom code, dead reasoning |

**Complementary:** Together they answer "What matters?" AND "Is it properly derived?"

### Prior Work on CoT Faithfulness

| Study | Finding |
|-------|---------|
| Chen et al. (Anthropic, 2025) | Claude 3.7 only 25% faithful on hint verbalization |
| Arcuschin et al. (2025) | 13% implicit post-hoc rationalization in GPT-4o-mini |
| Lanham et al. (Anthropic, 2023) | Larger models produce less faithful reasoning |

**CoT-DFA contribution:** Lightweight, single-pass structural analysis that complements expensive causal methods.

---

## Technical Details

### Reaching Definitions Formalization

```
Let:
  S = {s₁, s₂, ..., sₙ}     be the set of CoT segments
  C = {c₁, c₂, ..., cₘ}     be the set of code elements
  κ: S ∪ C → P(Concepts)    be concept extraction function

Reaching Definitions:
  RD(cᵢ) = { sⱼ ∈ S | concepts_match(κ(sⱼ), κ(cᵢ)) }

Phantom Set:
  Phantom = { cᵢ ∈ C | RD(cᵢ) = ∅ }

Dead Set:
  Dead = { sⱼ ∈ S | ∀ cᵢ ∈ C : sⱼ ∉ RD(cᵢ) }
```

### Concept Matching (Tiered)

```python
def concepts_match(cot_concepts, code_concepts, cot_text, code_text):
    shared = cot_concepts & code_concepts
    
    # Tier 1: Multiple concept overlap
    if len(shared) >= 2:
        return True
    
    # Tier 2: Rare concept match
    if len(shared) == 1:
        concept = list(shared)[0]
        if idf_score(concept) >= 2.0:
            return True
    
    # Tier 3: Semantic similarity fallback
    if shared and semantic_similarity(cot_text, code_text) >= 0.75:
        return True
    
    return False
```

---

## References

1. Lanham et al. (2023). "Measuring Faithfulness in Chain-of-Thought Reasoning." Anthropic.
2. Arcuschin et al. (2025). "Chain-of-Thought Unfaithfulness." arXiv:2503.08679.
3. Chen et al. (2025). "Reasoning Models Don't Always Say What They Think." Anthropic.
4. Aho, Sethi, Ullman. "Compilers: Principles, Techniques, and Tools" (Dragon Book), Ch. 9.
