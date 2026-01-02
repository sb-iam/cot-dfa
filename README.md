# CoT-DFA: Chain-of-Thought Dataflow Analysis

> **MATS 10.0 Application Project for Neel Nanda**  
> Applying Compiler Reaching Definitions to Detect Unfaithful Reasoning

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  "Can we tell when a CoT was causally important for giving its answer?"    │
│                                        — Neel Nanda, MATS 10.0 Interests   │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Executive Summary

**Problem**: How do we know if a model's Chain-of-Thought reasoning actually 
influenced its answer, or if it's post-hoc rationalization?

**Approach**: Apply **reaching definitions** — a classical compiler dataflow 
analysis — to CoT traces. This detects:
- **Phantom Code**: Elements appearing in output without CoT justification
- **Dead Reasoning**: CoT steps that don't contribute to any output

**Key Insight**: Treat CoT as a program trace where each reasoning step 
"defines" concepts that should "reach" the final output.

**Target Domain**: Code generation models (output is verifiable via execution)

---

## Table of Contents

1. [Research Question & Hypothesis](#1-research-question--hypothesis)
2. [Background: Reaching Definitions](#2-background-reaching-definitions)
3. [Architecture Overview](#3-architecture-overview)
4. [Formal Definitions](#4-formal-definitions)
5. [Metrics](#5-metrics)
6. [Implementation Plan](#6-implementation-plan)
7. [Alignment with Neel's Interests](#7-alignment-with-neels-interests)
8. [Extension: Circuit Provenance Bridge](#8-extension-circuit-provenance-bridge)
9. [Compute & Timeline](#9-compute--timeline)
10. [Expected Results](#10-expected-results)

---

## 1. Research Question & Hypothesis

### Primary Research Question

> **Can compiler-style reaching definitions analysis detect unfaithful 
> Chain-of-Thought in code generation models?**

### Hypothesis (Pass/Fail)

```
H₁: phantom_ratio (code elements without CoT justification) 
    correlates with test case failure rate.

    High phantoms → Model generated code without reasoning it through
                 → Higher probability of bugs

H₀: No correlation (phantom_ratio independent of correctness)
```

### Secondary Hypotheses

```
H₂: dead_ratio (CoT steps not reaching code) > 0.3 indicates 
    "padding" behavior — model generating filler reasoning

H₃: Models produce MORE phantoms on harder problems 
    (taking shortcuts under difficulty)

H₄: Correlation between phantom locations and actual bug locations
```

---

## 2. Background: Reaching Definitions

### Classical Compiler Definition

In program analysis, **reaching definitions** answers:

> "For each use of variable x, which definitions of x could have 
> produced the value?"

```
┌─────────────────────────────────────────────────────────────┐
│  Traditional Program Analysis                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│    d1: x = 5          ──────┐                               │
│    d2: y = x + 1            │ d1 reaches this use           │
│    d3: x = 10         ──────┼──────┐                        │
│    d4: z = x + y            │      │ d3 reaches, d1 killed  │
│                             ▼      ▼                        │
│                                                              │
│    At d2: RD(x) = {d1}                                      │
│    At d4: RD(x) = {d3}, RD(y) = {d2}                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### CoT-DFA Mapping

| Program Analysis      | CoT-DFA Equivalent                        |
|-----------------------|-------------------------------------------|
| Variable definition   | CoT step introducing concept/approach     |
| Variable use          | Code element using that concept           |
| Reaching definition   | Which CoT step justifies this code?       |
| Dead code             | CoT steps not reaching any output         |
| Use without def       | **PHANTOM** — code without reasoning      |

```
┌─────────────────────────────────────────────────────────────┐
│  CoT-DFA Analysis                                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  CoT Trace:                                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ s1: "First, I'll use a hash map for O(1) lookup"    │   │
│  │ s2: "I need to handle the edge case of empty input" │   │
│  │ s3: "Let me add some comments for clarity"          │   │
│  └─────────────────────────────────────────────────────┘   │
│            │                    │                           │
│            ▼                    ▼                           │
│  Code Output:                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ def solve(nums):                                     │   │
│  │     seen = {}  ◄─── s1 reaches (hash map)           │   │
│  │     if not nums:  ◄─── s2 reaches (edge case)       │   │
│  │         return -1                                    │   │
│  │     for n in nums:                                   │   │
│  │         seen[n] = True                               │   │
│  │     return max(seen.keys()) ◄─── PHANTOM! (no CoT)  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Analysis:                                                   │
│  • s1 → LIVE (reaches hash map usage)                       │
│  • s2 → LIVE (reaches edge case check)                      │
│  • s3 → DEAD (no code element matches "comments")           │
│  • max(seen.keys()) → PHANTOM (not discussed in CoT)        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Architecture Overview

### High-Level Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CoT-DFA PIPELINE                                  │
└─────────────────────────────────────────────────────────────────────────┘

     ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
     │  INPUT   │      │  PARSE   │      │ ANALYZE  │      │  OUTPUT  │
     │          │ ───► │          │ ───► │          │ ───► │          │
     │ Model    │      │ CoT +    │      │ Reaching │      │ Metrics  │
     │ Response │      │ Code     │      │ Defs     │      │ + Report │
     └──────────┘      └──────────┘      └──────────┘      └──────────┘
          │                 │                 │                 │
          ▼                 ▼                 ▼                 ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
    │<think>   │      │Segments: │      │Def-Use   │      │phantom:  │
    │...       │      │ s1, s2   │      │Graph     │      │ 0.15     │
    │</think>  │      │          │      │          │      │dead:     │
    │```python │      │AST:      │      │Reaching  │      │ 0.33     │
    │def f():  │      │ nodes    │      │Sets      │      │coverage: │
    │  ...     │      │          │      │          │      │ 0.85     │
    └──────────┘      └──────────┘      └──────────┘      └──────────┘
```

### Component Details

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         COMPONENT ARCHITECTURE                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 1. COT PARSER                                                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Input: Raw model response with <think>...</think> and code blocks     │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  def parse_cot(response: str) -> tuple[list[Segment], AST]: │       │
│  │      # 1. Extract thinking block                             │       │
│  │      think_match = re.search(r'<think>(.*?)</think>', ...)  │       │
│  │                                                               │       │
│  │      # 2. Split into semantic segments                       │       │
│  │      segments = segment_by_sentence(think_text)              │       │
│  │                                                               │       │
│  │      # 3. Extract code block                                  │       │
│  │      code = extract_code_block(response)                     │       │
│  │                                                               │       │
│  │      # 4. Parse code to AST                                   │       │
│  │      ast = parse_python(code)                                │       │
│  │                                                               │       │
│  │      return segments, ast                                    │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  Output:                                                                │
│  • segments: List of CoT reasoning steps                               │
│  • ast: Parsed AST of generated code                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 2. CONCEPT EXTRACTOR                                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  For CoT Segments - Extract "Definitions":                              │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  Segment: "I'll use a hash map for O(1) lookup"             │       │
│  │                                                               │       │
│  │  Extracted Concepts (Definitions):                            │       │
│  │  • data_structure: "hash_map" / "dict"                       │       │
│  │  • complexity: "O(1)"                                        │       │
│  │  • operation: "lookup"                                       │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  For Code AST - Extract "Uses":                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  Code: seen = {}                                             │       │
│  │                                                               │       │
│  │  Extracted Concepts (Uses):                                   │       │
│  │  • data_structure: "dict"                                    │       │
│  │  • variable: "seen"                                          │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  Matching Methods:                                                      │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  Option A: Keyword matching (simple, fast)                   │       │
│  │  Option B: Embedding similarity (semantic, slower)           │       │
│  │  Option C: LLM-based matching (accurate, expensive)          │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 3. REACHING DEFINITIONS ANALYZER                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Build Def-Use Graph:                                                   │
│                                                                         │
│       CoT Segments (Definitions)          Code Elements (Uses)          │
│       ─────────────────────────           ─────────────────────         │
│                                                                         │
│       ┌─────────────────┐                 ┌─────────────────┐          │
│       │ s1: hash_map,   │ ──────────────► │ seen = {}       │          │
│       │     O(1)        │                 └─────────────────┘          │
│       └─────────────────┘                                               │
│                                                                         │
│       ┌─────────────────┐                 ┌─────────────────┐          │
│       │ s2: edge_case,  │ ──────────────► │ if not nums:    │          │
│       │     empty_input │                 └─────────────────┘          │
│       └─────────────────┘                                               │
│                                                                         │
│       ┌─────────────────┐                                               │
│       │ s3: comments    │ ──────X (no edge = DEAD)                     │
│       └─────────────────┘                                               │
│                                                                         │
│                                           ┌─────────────────┐          │
│           X (no incoming edge) ◄───────── │ return max(...) │ PHANTOM  │
│                                           └─────────────────┘          │
│                                                                         │
│  Algorithm:                                                             │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  def compute_reaching_defs(segments, ast_nodes):             │       │
│  │      reaching = {}  # node -> set of segments                │       │
│  │      for node in ast_nodes:                                  │       │
│  │          node_concepts = extract_concepts(node)              │       │
│  │          reaching[node] = set()                              │       │
│  │          for seg in segments:                                │       │
│  │              seg_concepts = extract_concepts(seg)            │       │
│  │              if concepts_match(seg_concepts, node_concepts): │       │
│  │                  reaching[node].add(seg)                     │       │
│  │      return reaching                                         │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Formal Definitions

### Core Data Structures

```python
from dataclasses import dataclass
from typing import Set, Dict, List
import ast

@dataclass
class Segment:
    """A reasoning step from Chain-of-Thought"""
    id: str                    # Unique identifier (s1, s2, ...)
    text: str                  # Raw text content
    concepts: Set[str]         # Extracted concepts/keywords
    position: int              # Position in CoT sequence

@dataclass  
class CodeElement:
    """An AST node from generated code"""
    id: str                    # Unique identifier (c1, c2, ...)
    node: ast.AST              # Python AST node
    concepts: Set[str]         # Extracted concepts
    line_number: int           # Source location

@dataclass
class ReachingSet:
    """Reaching definitions for a code element"""
    element: CodeElement
    definitions: Set[Segment]  # Segments that "reach" this element
    
    @property
    def is_phantom(self) -> bool:
        """No CoT justification for this code"""
        return len(self.definitions) == 0

@dataclass
class DFAResult:
    """Complete analysis result"""
    segments: List[Segment]
    elements: List[CodeElement]
    reaching: Dict[str, ReachingSet]  # element_id -> reaching set
    
    @property
    def phantoms(self) -> List[CodeElement]:
        """Code elements with no reaching definitions"""
        return [r.element for r in self.reaching.values() if r.is_phantom]
    
    @property
    def dead_segments(self) -> List[Segment]:
        """Segments that don't reach any code"""
        reached = set()
        for rs in self.reaching.values():
            reached.update(s.id for s in rs.definitions)
        return [s for s in self.segments if s.id not in reached]
```

### Formal Analysis Definition

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      FORMAL DEFINITIONS                                  │
└─────────────────────────────────────────────────────────────────────────┘

Let:
  S = {s₁, s₂, ..., sₙ}     be the set of CoT segments
  C = {c₁, c₂, ..., cₘ}     be the set of code elements
  κ: S ∪ C → P(Concepts)    be concept extraction function
  ≈: Concept × Concept → 𝔹  be concept matching relation

Reaching Definitions:
  RD(cᵢ) = { sⱼ ∈ S | ∃ concept ∈ κ(cᵢ) : 
             ∃ concept' ∈ κ(sⱼ) : concept ≈ concept' }

Phantom Set:
  Phantom = { cᵢ ∈ C | RD(cᵢ) = ∅ }

Dead Set:
  Dead = { sⱼ ∈ S | ∀ cᵢ ∈ C : sⱼ ∉ RD(cᵢ) }

Live Set:
  Live = S \ Dead

```

---

## 5. Metrics

### Primary Metrics

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           FAITHFULNESS METRICS                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  PHANTOM RATIO                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                    |Phantom|        # code elements without CoT        │
│  phantom_ratio = ───────────── = ──────────────────────────────────    │
│                      |C|              # total code elements            │
│                                                                         │
│  Interpretation:                                                        │
│  • 0.0 = Perfect: Every code element has CoT justification             │
│  • 0.5 = Concerning: Half the code "appeared from nowhere"             │
│  • 1.0 = Complete disconnect: CoT irrelevant to code                   │
│                                                                         │
│  Hypothesis: High phantom_ratio correlates with test failures          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  DEAD RATIO                                                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                  |Dead|          # CoT steps reaching nothing          │
│  dead_ratio = ────────── = ─────────────────────────────────────       │
│                  |S|               # total CoT segments                 │
│                                                                         │
│  Interpretation:                                                        │
│  • 0.0 = Efficient: Every reasoning step contributes                   │
│  • 0.3 = Normal: Some exploratory thinking                             │
│  • 0.7+ = Suspicious: Mostly filler/padding                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  REACH COVERAGE                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                      |C \ Phantom|      # justified code elements      │
│  reach_coverage = ──────────────── = ─────────────────────────────     │
│                         |C|              # total code elements          │
│                                                                         │
│  Note: reach_coverage = 1 - phantom_ratio                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Derived Metrics

```
┌─────────────────────────────────────────────────────────────────────────┐
│  DERIVATION DEPTH                                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  For each code element c, count the unique segments reaching it:        │
│                                                                         │
│  depth(c) = |RD(c)|                                                     │
│                                                                         │
│  • depth = 0: Phantom (unjustified)                                    │
│  • depth = 1: Single justification (common)                            │
│  • depth > 1: Multiple justifications (well-supported)                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  REASONING EFFICIENCY                                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                        |Live|                                          │
│  efficiency = ──────────────────                                        │
│                |Live| + |Dead|                                          │
│                                                                         │
│  How much of the reasoning was "productive"                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Implementation Plan

### Phase 1: Data Collection (2 hours)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  PHASE 1: COLLECT COT + CODE SAMPLES                                   │
└─────────────────────────────────────────────────────────────────────────┘

Model Selection:
┌─────────────────────────────────────────────────────────────────────────┐
│  Model                      Size    CoT Style        Availability      │
├─────────────────────────────────────────────────────────────────────────┤
│  DeepSeek-Coder-V2-Lite    16B*    Native <think>   HuggingFace       │
│  Qwen2.5-Coder-7B-Instruct 7B      Prompted CoT     HuggingFace       │
│  DeepSeek-R1-Distill-Qwen  7B      Native <think>   HuggingFace       │
│  CodeQwen1.5-7B-Chat       7B      Prompted CoT     HuggingFace       │
└─────────────────────────────────────────────────────────────────────────┘
* Uses MoE, active params ~2.4B

Dataset:
┌─────────────────────────────────────────────────────────────────────────┐
│  Source: HumanEval or MBPP (with test cases for verification)          │
│  Sample: 50-100 problems                                                │
│  Format: problem → (CoT reasoning, generated code, test results)       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Phase 2: Parser Implementation (3 hours)

```python
# cot_parser.py

import re
import ast
from typing import Tuple, List

def parse_response(response: str) -> Tuple[List[str], str]:
    """
    Parse model response into CoT segments and code.
    
    Handles formats:
    - <think>...</think> + ```python...```
    - Reasoning paragraphs + code block
    """
    # Extract thinking block
    think_pattern = r'<think>(.*?)</think>'
    think_match = re.search(think_pattern, response, re.DOTALL)
    
    if think_match:
        cot_text = think_match.group(1)
    else:
        # Fall back to text before code block
        code_start = response.find('```')
        cot_text = response[:code_start] if code_start > 0 else ""
    
    # Extract code block
    code_pattern = r'```(?:python)?\n(.*?)```'
    code_match = re.search(code_pattern, response, re.DOTALL)
    code = code_match.group(1) if code_match else ""
    
    # Segment CoT by sentences
    segments = segment_cot(cot_text)
    
    return segments, code

def segment_cot(text: str) -> List[str]:
    """Split CoT into semantic segments (sentences/steps)."""
    # Split on sentence boundaries
    sentences = re.split(r'(?<=[.!?])\s+', text.strip())
    
    # Filter empty and very short
    segments = [s.strip() for s in sentences if len(s.strip()) > 10]
    
    return segments

def parse_code_ast(code: str) -> ast.AST:
    """Parse Python code to AST."""
    try:
        return ast.parse(code)
    except SyntaxError:
        # Return empty module for unparseable code
        return ast.Module(body=[], type_ignores=[])
```

### Phase 3: Concept Extraction (4 hours)

```python
# concept_extractor.py

import ast
from typing import Set, Dict
from dataclasses import dataclass

# Concept vocabulary for code patterns
CODE_CONCEPTS = {
    # Data structures
    'dict': {'hash', 'map', 'dictionary', 'hashmap', 'hash map', 'key-value'},
    'list': {'array', 'list', 'sequence', 'collection'},
    'set': {'set', 'unique', 'deduplicate'},
    'stack': {'stack', 'lifo', 'push', 'pop'},
    'queue': {'queue', 'fifo', 'deque'},
    'heap': {'heap', 'priority', 'heapq'},
    'tree': {'tree', 'binary', 'bst', 'trie'},
    'graph': {'graph', 'dfs', 'bfs', 'traverse'},
    
    # Algorithms
    'sort': {'sort', 'order', 'arrange', 'sorted'},
    'search': {'search', 'find', 'lookup', 'binary search'},
    'recursion': {'recursive', 'recursion', 'base case'},
    'dp': {'dynamic programming', 'memoization', 'memo', 'dp'},
    'greedy': {'greedy', 'optimal', 'local'},
    
    # Control flow
    'loop': {'iterate', 'loop', 'for', 'while', 'each'},
    'condition': {'if', 'check', 'condition', 'edge case'},
    'early_return': {'return early', 'base case', 'edge case'},
}

def extract_cot_concepts(segment: str) -> Set[str]:
    """Extract programming concepts from CoT segment."""
    segment_lower = segment.lower()
    concepts = set()
    
    for concept, keywords in CODE_CONCEPTS.items():
        if any(kw in segment_lower for kw in keywords):
            concepts.add(concept)
    
    return concepts

def extract_ast_concepts(node: ast.AST) -> Set[str]:
    """Extract programming concepts from AST node."""
    concepts = set()
    
    # Analyze node type
    if isinstance(node, ast.Dict):
        concepts.add('dict')
    elif isinstance(node, ast.List):
        concepts.add('list')
    elif isinstance(node, ast.Set):
        concepts.add('set')
    elif isinstance(node, ast.For):
        concepts.add('loop')
    elif isinstance(node, ast.While):
        concepts.add('loop')
    elif isinstance(node, ast.If):
        concepts.add('condition')
    elif isinstance(node, ast.Return):
        concepts.add('early_return')
    elif isinstance(node, ast.Call):
        # Check function name
        if isinstance(node.func, ast.Name):
            fname = node.func.id.lower()
            if fname in ('sorted', 'sort'):
                concepts.add('sort')
            elif fname in ('heapify', 'heappush', 'heappop'):
                concepts.add('heap')
    
    return concepts

def get_significant_nodes(tree: ast.AST) -> List[ast.AST]:
    """Get AST nodes worth analyzing (skip trivial ones)."""
    significant = []
    
    for node in ast.walk(tree):
        if isinstance(node, (
            ast.FunctionDef, ast.AsyncFunctionDef,
            ast.For, ast.While, ast.If,
            ast.Dict, ast.List, ast.Set,
            ast.Return, ast.Assign,
            ast.Call
        )):
            significant.append(node)
    
    return significant
```

### Phase 4: Reaching Definitions Analysis (4 hours)

```python
# reaching_defs.py

from dataclasses import dataclass
from typing import Dict, Set, List
import ast

@dataclass
class AnalysisResult:
    reaching: Dict[int, Set[int]]  # node_idx -> set of segment_idx
    phantoms: List[int]            # node indices with no reaching defs
    dead: List[int]                # segment indices not reaching anything
    
    @property
    def phantom_ratio(self) -> float:
        total_nodes = len(self.reaching)
        if total_nodes == 0:
            return 0.0
        return len(self.phantoms) / total_nodes
    
    @property
    def dead_ratio(self) -> float:
        all_segments = set()
        for segs in self.reaching.values():
            all_segments.update(segs)
        total_segments = max(max(all_segments) + 1 if all_segments else 0,
                            len(self.dead) + len(all_segments))
        if total_segments == 0:
            return 0.0
        return len(self.dead) / total_segments

def compute_reaching_definitions(
    cot_segments: List[str],
    code_nodes: List[ast.AST],
    similarity_threshold: float = 0.3
) -> AnalysisResult:
    """
    Compute reaching definitions from CoT segments to code nodes.
    
    A segment s "reaches" a node n if they share concepts.
    """
    # Extract concepts for all segments
    segment_concepts = [extract_cot_concepts(seg) for seg in cot_segments]
    
    # Extract concepts for all nodes
    node_concepts = [extract_ast_concepts(node) for node in code_nodes]
    
    # Compute reaching definitions
    reaching = {}
    for node_idx, node_conc in enumerate(node_concepts):
        reaching[node_idx] = set()
        for seg_idx, seg_conc in enumerate(segment_concepts):
            # Check concept overlap
            if node_conc & seg_conc:  # Non-empty intersection
                reaching[node_idx].add(seg_idx)
    
    # Find phantoms (nodes with no reaching definitions)
    phantoms = [idx for idx, segs in reaching.items() if len(segs) == 0]
    
    # Find dead segments (segments not reaching any node)
    all_reaching = set()
    for segs in reaching.values():
        all_reaching.update(segs)
    dead = [idx for idx in range(len(cot_segments)) if idx not in all_reaching]
    
    return AnalysisResult(reaching, phantoms, dead)
```

### Phase 5: Validation Pipeline (4 hours)

```python
# validation.py

import subprocess
import tempfile
from typing import Tuple, Optional

def execute_code_with_tests(
    code: str, 
    test_code: str,
    timeout: int = 10
) -> Tuple[bool, Optional[str]]:
    """
    Execute generated code against test cases.
    
    Returns: (passed: bool, error_msg: Optional[str])
    """
    full_code = f"{code}\n\n{test_code}"
    
    with tempfile.NamedTemporaryFile(mode='w', suffix='.py', delete=False) as f:
        f.write(full_code)
        f.flush()
        
        try:
            result = subprocess.run(
                ['python', f.name],
                capture_output=True,
                text=True,
                timeout=timeout
            )
            
            if result.returncode == 0:
                return True, None
            else:
                return False, result.stderr
                
        except subprocess.TimeoutExpired:
            return False, "Timeout"
        except Exception as e:
            return False, str(e)

def analyze_sample(
    problem: str,
    cot_response: str,
    test_code: str
) -> dict:
    """
    Complete analysis pipeline for one sample.
    """
    # 1. Parse response
    segments, code = parse_response(cot_response)
    
    # 2. Parse code AST
    tree = parse_code_ast(code)
    nodes = get_significant_nodes(tree)
    
    # 3. Compute reaching definitions
    result = compute_reaching_definitions(segments, nodes)
    
    # 4. Execute tests
    passed, error = execute_code_with_tests(code, test_code)
    
    return {
        'problem': problem,
        'num_segments': len(segments),
        'num_nodes': len(nodes),
        'phantom_ratio': result.phantom_ratio,
        'dead_ratio': result.dead_ratio,
        'test_passed': passed,
        'error': error,
        'phantoms': result.phantoms,
        'dead': result.dead,
    }
```

### Phase 6: Hypothesis Testing (3 hours)

```python
# hypothesis_test.py

import numpy as np
from scipy import stats
from typing import List, Dict

def test_phantom_correlation(results: List[Dict]) -> Dict:
    """
    Test H1: phantom_ratio correlates with test failure.
    """
    phantom_ratios = [r['phantom_ratio'] for r in results]
    test_passed = [1 if r['test_passed'] else 0 for r in results]
    
    # Point-biserial correlation (continuous vs binary)
    correlation, p_value = stats.pointbiserialr(test_passed, phantom_ratios)
    
    # Group comparison
    passed_phantoms = [r['phantom_ratio'] for r in results if r['test_passed']]
    failed_phantoms = [r['phantom_ratio'] for r in results if not r['test_passed']]
    
    t_stat, t_pvalue = stats.ttest_ind(passed_phantoms, failed_phantoms)
    
    return {
        'correlation': correlation,
        'correlation_p': p_value,
        't_statistic': t_stat,
        't_pvalue': t_pvalue,
        'mean_phantom_passed': np.mean(passed_phantoms) if passed_phantoms else None,
        'mean_phantom_failed': np.mean(failed_phantoms) if failed_phantoms else None,
        'hypothesis_supported': p_value < 0.05 and correlation < 0,
    }

def generate_report(results: List[Dict], hypothesis_results: Dict) -> str:
    """Generate human-readable analysis report."""
    report = []
    report.append("=" * 70)
    report.append("COT-DFA ANALYSIS REPORT")
    report.append("=" * 70)
    
    # Summary statistics
    report.append(f"\nSamples analyzed: {len(results)}")
    report.append(f"Tests passed: {sum(1 for r in results if r['test_passed'])}")
    report.append(f"Tests failed: {sum(1 for r in results if not r['test_passed'])}")
    
    # Metric averages
    report.append(f"\nAverage phantom_ratio: {np.mean([r['phantom_ratio'] for r in results]):.3f}")
    report.append(f"Average dead_ratio: {np.mean([r['dead_ratio'] for r in results]):.3f}")
    
    # Hypothesis test
    report.append("\n" + "-" * 70)
    report.append("HYPOTHESIS TEST: H1 (phantom_ratio predicts failure)")
    report.append("-" * 70)
    
    hr = hypothesis_results
    report.append(f"Correlation: {hr['correlation']:.3f} (p={hr['correlation_p']:.4f})")
    report.append(f"Mean phantom (passed): {hr['mean_phantom_passed']:.3f}")
    report.append(f"Mean phantom (failed): {hr['mean_phantom_failed']:.3f}")
    report.append(f"\nHypothesis supported: {'YES ✓' if hr['hypothesis_supported'] else 'NO ✗'}")
    
    return "\n".join(report)
```

---

## 7. Alignment with Neel's Interests

```
┌─────────────────────────────────────────────────────────────────────────┐
│           ALIGNMENT WITH NEEL NANDA'S MATS 10.0 INTERESTS               │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────┬──────────────────────────────┬──────────┐
│ Neel's Interest              │ CoT-DFA Addresses            │ Match    │
├──────────────────────────────┼──────────────────────────────┼──────────┤
│ "Can we tell when CoT was    │ Reaching definitions track   │          │
│  causally important for      │ exactly which CoT segments   │    ✓     │
│  giving its answer?"         │ contribute to code elements  │          │
├──────────────────────────────┼──────────────────────────────┼──────────┤
│ "Design good monitors or     │ phantom_ratio, dead_ratio,   │          │
│  metrics for whether CoT     │ reach_coverage are exactly   │    ✓     │
│  is telling us what we       │ this type of metric          │          │
│  think?"                     │                              │          │
├──────────────────────────────┼──────────────────────────────┼──────────┤
│ "Extend Thought Anchors"     │ Complementary formalism:     │          │
│                              │ • Thought Anchors: causal    │    ✓     │
│                              │ • CoT-DFA: structural        │          │
├──────────────────────────────┼──────────────────────────────┼──────────┤
│ "Reasoning models"           │ Targets code generation      │          │
│                              │ with native <think> blocks   │    ✓     │
├──────────────────────────────┼──────────────────────────────┼──────────┤
│ "Applied interpretability"   │ Single-pass, no expensive    │          │
│                              │ resampling, actionable       │    ✓     │
├──────────────────────────────┼──────────────────────────────┼──────────┤
│ "Start simple"               │ Classical compiler analysis  │          │
│                              │ applied to new domain        │    ✓     │
└──────────────────────────────┴──────────────────────────────┴──────────┘

Comparison: CoT-DFA vs Thought Anchors
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  THOUGHT ANCHORS (Bogdan et al.)     COT-DFA (This Work)               │
│  ──────────────────────────────      ─────────────────────              │
│                                                                         │
│  Question: "Which sentences          Question: "Is this a valid        │
│            matter causally?"                    derivation?"            │
│                                                                         │
│  Method:   Counterfactual            Method:   Structural analysis     │
│            perturbation                        (no model calls)        │
│                                                                         │
│  Cost:     O(n) forward passes       Cost:     O(1) - single pass      │
│            per sample                          parse + match           │
│                                                                         │
│  Detects:  • Important sentences     Detects:  • Phantom code          │
│            • Attention patterns                • Dead reasoning        │
│                                                                         │
│  ─────────────────────────────────────────────────────────────────     │
│                                                                         │
│  COMPLEMENTARY: Together they answer both                              │
│  "What matters?" AND "Is it properly derived?"                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Extension: Circuit Provenance Bridge

> **If time permits**, extend CoT-DFA to answer:
> "What training data causes unfaithful CoT?"

```
┌─────────────────────────────────────────────────────────────────────────┐
│           OPTION C: BRIDGING TO CIRCUIT PROVENANCE                      │
└─────────────────────────────────────────────────────────────────────────┘

REFRAMED RESEARCH QUESTION:
────────────────────────────
Original: "What training data caused this computational pathway?"
     New: "What training data causes UNFAITHFUL CoT?"

NOVEL HYPOTHESIS:
────────────────────────────
H5: Training examples with shortcuts (correct answer, weak reasoning)
    cause models to produce unfaithful Chain-of-Thought.

METHODOLOGY:
────────────────────────────

    ┌─────────────────────────────────────────────────────────────────┐
    │ 1. Get faithful vs unfaithful CoT examples                      │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │    Faithful                    Unfaithful                       │
    │    ─────────                   ──────────                       │
    │    phantom_ratio < 0.1         phantom_ratio > 0.4              │
    │    dead_ratio < 0.2            dead_ratio > 0.5                 │
    │    test_passed = True          (may or may not pass)            │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘
                        │                        │
                        ▼                        ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │ 2. Apply influence functions to both categories                 │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │    influence(training_example, output) = ∇θL(z_test)ᵀ H⁻¹ ∇θL(z)│
    │                                                                  │
    │    For each category, find top-10 influential training examples │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘
                        │                        │
                        ▼                        ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │ 3. Compare influential examples                                  │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │    Faithful Output           Unfaithful Output                  │
    │    ──────────────           ────────────────                    │
    │    Training examples         Training examples                  │
    │    with strong reasoning     with SHORTCUTS?                    │
    │    chains                                                       │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘

PASS/FAIL CRITERION:
────────────────────────────
H5 supported if:
  shortcut_prevalence(unfaithful_influencers) > 
  shortcut_prevalence(faithful_influencers)

Where shortcut = training example that arrives at correct answer
                 without proper reasoning justification

IMPLEMENTATION NOTES:
────────────────────────────
• Uses existing Circuit Provenance JAX infrastructure
• EK-FAC for efficient influence function computation
• Target model: Qwen2.5-Coder-7B or similar
• Training data: Code Alpaca or similar instruction-tuning dataset
```

### Integration Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    UNIFIED ANALYSIS ARCHITECTURE                        │
└─────────────────────────────────────────────────────────────────────────┘

                           Model Output
                                │
                                ▼
                    ┌───────────────────────┐
                    │    CoT-DFA Analysis   │
                    │   (Reaching Defs)     │
                    └───────────┬───────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
           ┌───────────────┐       ┌───────────────┐
           │   Faithful    │       │  Unfaithful   │
           │   (low        │       │  (high        │
           │   phantom)    │       │   phantom)    │
           └───────┬───────┘       └───────┬───────┘
                   │                       │
                   └───────────┬───────────┘
                               │
                               ▼
                   ┌───────────────────────┐
                   │  Circuit Provenance   │
                   │  (Influence Funcs)    │
                   └───────────┬───────────┘
                               │
                   ┌───────────┴───────────┐
                   │                       │
                   ▼                       ▼
           ┌───────────────┐       ┌───────────────┐
           │   Training    │       │   Training    │
           │   Influencers │       │   Influencers │
           │   (Faithful)  │       │  (Unfaithful) │
           └───────────────┘       └───────────────┘
                   │                       │
                   └───────────┬───────────┘
                               │
                               ▼
                   ┌───────────────────────┐
                   │     COMPARE           │
                   │  Do unfaithful trace  │
                   │  back to "shortcut"   │
                   │  training examples?   │
                   └───────────────────────┘
```

---

## 9. Compute & Timeline

### Resource Requirements

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         COMPUTE BUDGET                                  │
└─────────────────────────────────────────────────────────────────────────┘

Hardware: Google Colab Pro (H100 80GB)
Constraint: 20 hours total

┌────────────────────────────┬──────────┬─────────────┬───────────────────┐
│ Phase                      │ Hours    │ GPU Memory  │ Notes             │
├────────────────────────────┼──────────┼─────────────┼───────────────────┤
│ 1. Data Collection         │ 2        │ ~20GB       │ Model inference   │
│ 2. Parser Implementation   │ 3        │ CPU only    │ Pure Python       │
│ 3. Concept Extraction      │ 4        │ CPU/embedder│ Optional embedder │
│ 4. Reaching Defs Analysis  │ 4        │ CPU only    │ Graph computation │
│ 5. Validation Pipeline     │ 4        │ CPU only    │ Test execution    │
│ 6. Hypothesis Testing      │ 3        │ CPU only    │ Statistical tests │
├────────────────────────────┼──────────┼─────────────┼───────────────────┤
│ TOTAL CoT-DFA              │ 20       │             │                   │
└────────────────────────────┴──────────┴─────────────┴───────────────────┘

EXTENSION (if time permits):
┌────────────────────────────┬──────────┬─────────────┬───────────────────┐
│ 7. Circuit Provenance      │ 10+      │ ~50GB       │ Influence funcs   │
└────────────────────────────┴──────────┴─────────────┴───────────────────┘
```

### Timeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           20-HOUR TIMELINE                              │
└─────────────────────────────────────────────────────────────────────────┘

Week 1 (10 hours):
─────────────────
Day 1-2: [████████░░] Data collection + parser (5h)
         • Generate CoT samples from coding model
         • Implement parser for <think> + code blocks
         • Build basic concept extraction

Day 3-4: [████████░░] Reaching definitions core (5h)
         • Implement matching algorithm
         • Build def-use graph construction
         • Calculate metrics

Week 2 (10 hours):
─────────────────
Day 5-6: [████████░░] Validation + testing (5h)
         • Execute generated code against test cases
         • Collect pass/fail data
         • Build analysis pipeline

Day 7:   [████░░░░░░] Hypothesis testing + report (5h)
         • Statistical analysis
         • Generate visualizations
         • Write up findings

Extension (if ahead):
────────────────────
Day 8+:  [░░░░░░░░░░] Circuit Provenance bridge
         • Classify outputs by faithfulness
         • Run influence functions
         • Compare training influencers
```

---

## 10. Expected Results

### Success Criteria

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        PASS/FAIL CRITERIA                               │
└─────────────────────────────────────────────────────────────────────────┘

PRIMARY HYPOTHESIS (H1):
───────────────────────
PASS: Significant negative correlation (p < 0.05) between 
      phantom_ratio and test pass rate

      Higher phantoms → More failures (as expected)

FAIL: No significant correlation OR positive correlation
      (phantoms don't predict failure)

INTERESTING FAILURE:
───────────────────────
Even if H1 fails, the tool has value if:
• We find different phantom patterns for different error types
• Dead ratio correlates with something else (code quality?)
• Certain problem types have systematically higher phantoms
```

### Expected Outputs

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DELIVERABLES                                     │
└─────────────────────────────────────────────────────────────────────────┘

1. COLAB NOTEBOOK
   ├── Cell 1: Setup + model loading
   ├── Cell 2: Data collection (generate N samples)
   ├── Cell 3: Parser implementation
   ├── Cell 4: Concept extractor
   ├── Cell 5: Reaching definitions analyzer
   ├── Cell 6: Validation pipeline
   ├── Cell 7: Hypothesis testing
   └── Cell 8: Visualizations + report

2. METRICS TABLE
   ┌──────────────┬──────────────┬──────────────┬──────────────┐
   │ Problem ID   │ phantom_ratio│ dead_ratio   │ test_passed  │
   ├──────────────┼──────────────┼──────────────┼──────────────┤
   │ humaneval_1  │ 0.15         │ 0.33         │ True         │
   │ humaneval_2  │ 0.42         │ 0.18         │ False        │
   │ ...          │ ...          │ ...          │ ...          │
   └──────────────┴──────────────┴──────────────┴──────────────┘

3. CORRELATION ANALYSIS
   • Point-biserial correlation coefficient
   • t-test between passed/failed groups
   • Effect size (Cohen's d)

4. VISUALIZATIONS
   • Scatter: phantom_ratio vs pass rate
   • Box plot: phantom distribution by pass/fail
   • Heatmap: concept matching matrix
```

---

## Quick Start

```python
# Full pipeline example

from cot_dfa import analyze_sample, test_phantom_correlation

# 1. Collect samples
samples = collect_cot_samples(model="deepseek-r1", n=100)

# 2. Analyze each sample
results = [analyze_sample(s.problem, s.response, s.tests) for s in samples]

# 3. Test hypothesis
hypothesis = test_phantom_correlation(results)

# 4. Generate report
print(generate_report(results, hypothesis))
```

---

## File Structure

```
cot-dfa-mats/
├── README.md                 # This file
├── notebooks/
│   ├── 01_data_collection.ipynb
│   ├── 02_cot_parser.ipynb
│   ├── 03_reaching_defs.ipynb
│   ├── 04_validation.ipynb
│   └── 05_hypothesis_test.ipynb
├── src/
│   ├── cot_parser.py
│   ├── concept_extractor.py
│   ├── reaching_defs.py
│   ├── validation.py
│   └── hypothesis_test.py
├── data/
│   ├── humaneval/
│   └── samples/
└── results/
    ├── metrics.csv
    └── report.md
```

---

## References

1. **Thought Anchors** - Bogdan, Christoph, et al. (2024) - Counterfactual approach
2. **Reaching Definitions** - Aho, Sethi, Ullman - Dragon Book, Ch. 9
3. **CoT Faithfulness** - Anthropic Research - Measuring Faithfulness in CoT

---

## Author

**Shakthi Bachala**  
PhD Candidate, University of Nebraska-Lincoln  
Research: Compiler-Integrated AI Interpretability (IAM)

*Applying decades of program analysis wisdom to the emerging challenge 
of understanding AI reasoning.*