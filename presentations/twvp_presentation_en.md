---
marp: true
theme: theme_U
paginate: true
footer: "2026/05/24<span class='fcenter'>Uchida Lab Paper Review</span>"
size: 4:3
math: katex
---

<style>
section.fs20 { font-size: 20px; }
section.fs22 { font-size: 22px; }
section.fs24 { font-size: 24px; }
section.fs28 { font-size: 28px; }
section.fs32 { font-size: 32px; }
.h_center { text-align: center; }
.v_center { display: flex; align-items: center; justify-content: center; }
.split { display: flex; gap: 28px; align-items: stretch; }
.split_l, .split_r { flex: 1; }
.card { border: 2px solid #222; border-radius: 10px; padding: 16px 20px; margin: 10px 0; background: #fafafa; }
.memo { border-left: 8px solid #999; padding: 10px 14px; background: #f6f6f6; font-size: 0.82em; }
.small { font-size: 0.78em; }
.tiny { font-size: 0.66em; }
code { font-size: 0.85em; }
pre { font-size: 0.72em; }
</style>

<!-- _class: cover -->
<!-- _paginate: false -->
<!-- class: fs28 -->

# Thinking with<br>Visual Primitives

<span style="text-align: center; margin-top: 10px;">

## Multimodal Reasoning by Anchoring Thought in Points and Boxes

</span>

Paper Presentation  
Your Name

---

<!-- _class: fs32 -->

# Why I Chose This Paper

- Multimodal LLMs are getting better at **seeing**, but they still often fail at **referring precisely** to what they see.

- Tasks such as **counting, spatial reasoning, maze solving, and path tracing** are hard to solve with language-only chain-of-thought.

- This paper proposes a simple but powerful idea: let the model **think while pointing** by inserting visual primitives directly into the reasoning trace.

---

<!-- _class: fs32 -->

# Main Takeaway

- Standard multimodal reasoning mostly unfolds in **language space**.

- The paper identifies a core bottleneck: the **Reference Gap**.

- It introduces **visual primitives**—mainly **points** and **bounding boxes**—as the smallest reasoning units for grounding thought in image space.

- The full pipeline combines:
  - large-scale grounding pretraining,
  - specialized SFT and RL,
  - unified refinement,
  - and on-policy distillation.

- The result is strong performance on **counting, spatial reasoning, and topological reasoning** with efficient visual token usage.

---

<!-- _class: section_title -->
# Background: Why Multimodal CoT Still Fails

<style>
section.section_title h1 {
    /
    /* background-color: red; */
}
</style>

---

<!-- class: fs24 -->

# The Limitation of Language-Only Reasoning

Given an image $I$ and a question $q$, a reasoning model predicts an answer $a$ through latent reasoning steps $z$:

$$
p_\theta(a \mid I,q)
= \sum_z p_\theta(z \mid I,q)\, p_\theta(a \mid I,q,z)
$$

In many multimodal CoT systems, $z$ is expressed almost entirely in natural language.

<div class="card">
<b>Problem:</b> the model may say “the small object on the left” or “the line near the center,” but such textual references can drift or become ambiguous across multiple reasoning steps.
</div>

- This is manageable for simple scenes.
- It becomes brittle in dense scenes, object comparison, topological tasks, and long reasoning chains.

---

<!-- class: fs24 -->

# Perception Gap vs. Reference Gap

<div class="split">
<div class="split_l" style="flex-basis: 50%;">

## Perception Gap

- The model does **not perceive enough visual detail**.
- Typical causes:
  - low resolution,
  - insufficient cropping,
  - too few visual tokens.

$$
\text{image detail} \not\rightarrow \text{usable visual tokens}
$$

</div>

<div class="split_r" style="flex-basis: 50%;">

## Reference Gap

- The model sees the scene, but cannot **stably refer to the correct entity or location**.
- Typical failure cases:
  - dense counting,
  - multi-hop spatial reasoning,
  - maze navigation,
  - path tracing.

$$
\text{visual entity} \not\leftrightarrow \text{textual reference}
$$

</div>
</div>

<div class="memo">
The paper argues that frontier MLLMs have made progress on perception, but still lack a robust and unambiguous reference mechanism.
</div>

---

<!-- class: fs24 -->

# A Simple Formalization of the Reference Gap

Let the image coordinate space be

$$
\Omega = [0,W) \times [0,H)
$$

and let a natural-language reference $r$ ideally identify a target location or entity $z \in \Omega$.

In practice, the model maintains a distribution:

$$
p(z \mid r, I)
$$

Ambiguity can be measured by the conditional entropy:

$$
\mathrm{Gap}_{\mathrm{ref}}(r,I) = H(Z \mid r,I)
$$

- Large $H(Z \mid r,I)$ means the reference is ambiguous.
- Visual primitives reduce the gap by turning $z$ into an **explicit coordinate or region**.

---

<!-- _class: section_title -->
# Proposal: Thinking with Visual Primitives

---

<!-- class: fs24 -->

# What Is a Visual Primitive?

A visual primitive is the smallest explicit grounding unit used inside the reasoning process.

<div class="split">
<div class="split_l" style="flex-basis: 50%;">

## Point

$$
p = [x,y]
$$

- abstract location reference
- useful for paths, trajectories, and waypoints
- well suited for topology and path-following tasks

</div>

<div class="split_r" style="flex-basis: 50%;">

## Bounding Box

$$
b = [x_1,y_1,x_2,y_2]
$$

- captures object location and scale
- useful for grounding, counting, and relations
- contains more information than a single point

</div>
</div>

---

<!-- class: fs24 -->

# Interleaving Language and Visual Primitives

Traditional chain-of-thought:

$$
y = (w_1, w_2, \ldots, w_T)
$$

Proposed reasoning trace:

$$
y = (w_1,\; v_1,\; w_2,\; v_2,\; \ldots,\; w_T,\; a)
$$

where each visual step is a primitive:

$$
v_t \in \{\text{point},\text{box}\}
$$

<div class="card">
<b>Intuition:</b> this mimics how humans solve visual tasks—by pointing, tracing, marking, and checking while reasoning.
</div>

---

<!-- class: fs22 -->

# Output Formats

## Bounding-Box Grounding

<div class="card">
<code>&lt;|ref|&gt;TARGET&lt;|/ref|&gt;&lt;|box|&gt;[[x1,y1,x2,y2], ...]&lt;|/box|&gt;</code>

- <b>TARGET</b> is the object category or referring phrase.
- Multiple instances are listed in <b>left-to-right order</b>.
- Coordinates are normalized to <b>discrete integers in [0, 999]</b>.
</div>

## Point Grounding

<div class="card">
<code>&lt;|point|&gt;[[x1,y1], [x2,y2], ...]&lt;|/point|&gt;</code>

- The object name does not always need to be repeated.
- This format is especially useful for <b>paths, trajectories, and abstract spatial references</b>.
</div>

---

<!-- class: fs24 -->

# Coordinate Normalization

Let the original image width and height be $(W,H)$, and let the pixel coordinate be $(u,v)$.

The normalized 0–999 coordinates are:

$$
x = \operatorname{round}\left(999 \cdot \frac{u}{W}\right),
\qquad
y = \operatorname{round}\left(999 \cdot \frac{v}{H}\right)
$$

The approximate inverse transform is:

$$
u \approx \frac{x}{999}W, \qquad v \approx \frac{y}{999}H
$$

For a box $b=[x_1,y_1,x_2,y_2]$, its center is

$$
c(b)=\left[\frac{x_1+x_2}{2},\frac{y_1+y_2}{2}\right]
$$

This normalized representation lets the same reasoning format transfer across image resolutions.

---

<!-- _class: section_title -->
# Model Architecture

---

# Architecture and Training Pipeline

<div class="h_center v_center">
<img src="images/fig2_arch_pipeline.png" height="360px">
</div>

- A vision encoder extracts image features.
- A text tokenizer encodes the prompt.
- A multimodal LLM processes interleaved vision-language tokens.
- A de-tokenizer generates a response that may include visual primitives.

---

<!-- class: fs24 -->

# Visual Token Compression

Let the image size be $H \times W$ and the ViT patch size be $P=14$.

The number of patch tokens is

$$
N_{\mathrm{patch}} = \left\lceil \frac{H}{P} \right\rceil
\left\lceil \frac{W}{P} \right\rceil
$$

After $3\times 3$ spatial token compression,

$$
N_{\mathrm{LLM}} \approx \frac{N_{\mathrm{patch}}}{9}
$$

With a compressed sparse attention factor $m=4$,

$$
N_{\mathrm{KV}} \approx \frac{N_{\mathrm{LLM}}}{m}
= \frac{N_{\mathrm{patch}}}{36}
$$

---

<!-- class: fs24 -->

# Compression Example

For a $756 \times 756$ image,

$$
\frac{756}{14} = 54
$$

$$
N_{\mathrm{patch}} = 54^2 = 2916
$$

After $3 \times 3$ compression,

$$
N_{\mathrm{LLM}} = 18^2 = 324
$$

After compressed sparse attention,

$$
N_{\mathrm{KV}} = \frac{324}{4} = 81
$$

So the compression ratio from pixels to KV entries is

$$
\frac{756 \times 756}{81} = 7056
$$

---

# Token Efficiency

<div class="h_center v_center">
<img src="images/fig1_token_efficiency.png" height="420px">
</div>

- The paper shows strong performance with relatively few visual tokens.
- The key point is not only accuracy, but also **efficient use of compute and KV cache**.

---

<!-- _class: section_title -->
# Pretraining Data

---

<!-- class: fs24 -->

# Building Large-Scale Box Grounding Data

The paper constructs box-grounding supervision from large-scale web data.

<div class="card">
<b>Data funnel</b>

$$
97{,}984\;\text{sources}
\rightarrow 43{,}141
\rightarrow 31{,}701
\rightarrow 40\text{M+ samples}
$$
</div>

## Filtering Step I: Semantic Review

- remove gibberish and meaningless labels
- remove private entities and overly specific labels
- remove ambiguous abbreviations or subjective labels

## Filtering Step II: Visual-Geometric Review

- missing annotations
- severe truncation or offset
- very low-quality or extremely oversized boxes

---

<!-- class: fs24 -->

# Why Emphasize Box Data?

<div class="split">
<div class="split_l" style="flex-basis: 48%;">

## 1. Determinism

- A box tightly encloses an object.
- A point can be placed almost anywhere inside the object, so the target is less deterministic.

</div>

<div class="split_r" style="flex-basis: 52%;">

## 2. Generalizability

A box can be written as two corner points:

$$
b = [(x_1,y_1),(x_2,y_2)]
$$

and a center point can be derived:

$$
p = c(b)
$$

</div>
</div>

<div class="memo">
Boxes encode not only location but also scale. That makes them particularly useful for counting and object comparison.
</div>

---

<!-- class: fs22 -->

# Unified Pretraining

The model is pretrained jointly on **general multimodal data** and **visual-primitive data**.

## Box Prompt Example

<div class="card">
<code>Locate TARGET in this image and report its bounding box coordinates.</code>

Expected response:

<code>&lt;|ref|&gt;TARGET&lt;|/ref|&gt;&lt;|box|&gt;[[x1,y1,x2,y2], ...]&lt;|/box|&gt;</code>
</div>

## Point Prompt Example

<div class="card">
<code>Help me find TARGET. Give me the center point for each instance.</code>

Expected response:

<code>&lt;|point|&gt;[[x1,y1], [x2,y2], ...]&lt;|/point|&gt;</code>
</div>

In total, pretraining consumes **trillions of multimodal tokens**.

---

<!-- _class: section_title -->
# Cold-Start Task Design

---

<!-- class: fs24 -->

# Four Cold-Start Task Families

- **Counting**
  - dense scenes and fine-grained constraints
  - object correspondence stabilized through boxes

- **Spatial Reasoning & General VQA**
  - grounding plus relational inference
  - based on GQA and CLEVR

- **Maze Navigation**
  - search and backtracking expressed as point sequences

- **Path Tracing**
  - tangled curves followed via ordered points

<div class="memo">
The cold-start data provides explicit intermediate supervision and is verified as much as possible with automatic checkers.
</div>

---

<!-- class: fs24 -->

# Counting with Boxes

Suppose the question $q$ specifies a target category and attribute constraints. Let the detected candidate boxes be

$$
B = \{b_1, b_2, \ldots, b_m\}
$$

Then the count is

$$
C(q,I) = \sum_{i=1}^{m}
\mathbf{1}\left[o_i \models \mathrm{target}(q)\right]
\mathbf{1}\left[o_i \models \mathrm{constraint}(q)\right]
$$

Coarse-grained counting:

$$
C = \sum_i \mathbf{1}[\mathrm{class}(o_i)=c]
$$

Fine-grained counting:

$$
C = \sum_i \mathbf{1}[\mathrm{class}=c]\mathbf{1}[\mathrm{attr}=a]\mathbf{1}[\mathrm{relation}=r]
$$

---

# Counting Example

<div class="h_center v_center">
<img src="images/fig3_counting_examples.png" height="500px">
</div>

- decompose the target condition
- ground the relevant objects
- verify each candidate systematically
- sum the valid instances

---

<!-- class: fs24 -->

# Spatial Reasoning and General VQA

## Natural Scenes: GQA

- use scene-graph metadata
- identify objects through multi-attribute constraints
- reasoning chain: intent analysis → grounding → relational inference

## Synthetic Scenes: CLEVR

- controllable object density
- programmatically generated execution traces
- object IDs projected to 2D boxes

Example relation predicates:

$$
\mathrm{left}(b_i,b_j) =
\mathbf{1}\left[ c_x(b_i) < c_x(b_j) \right]
$$

$$
\mathrm{same\_size}(b_i,b_j) =
\mathbf{1}\left[ |s(b_i)-s(b_j)| < \epsilon \right]
$$

---

# Spatial Reasoning Example

<div class="h_center v_center">
<img src="images/fig4_spatial_reasoning.png" height="500px">
</div>

- first locate the gray metallic object
- infer its size
- search for a purple rubber object of the same size
- avoid hallucinating an object if no such match exists

---

<!-- class: fs24 -->

# Maze Navigation as Reachability

Treat the maze as a graph

$$
G = (V,E)
$$

where:

- $V$ = cells or junctions,
- $E$ = valid moves not blocked by walls,
- $s \in V$ = start,
- $t \in V$ = target.

Solvability is a reachability problem:

$$
\exists P=(v_0,\ldots,v_L)
\quad \mathrm{s.t.}\quad
v_0=s,\; v_L=t,\; (v_{\ell},v_{\ell+1})\in E
$$

Visual primitives represent route waypoints:

$$
\langle p_0,p_1,\ldots,p_L \rangle
$$

---

# Maze Navigation Example

<div class="h_center v_center">
<img src="images/fig5_maze_navigation.png" height="500px">
</div>

- anchor the start and destination with points
- explore like DFS
- detect dead ends
- backtrack when necessary
- output the verified final path as a point sequence

---

<!-- class: fs24 -->

# Path Tracing by Geometric Continuity

Given an image with multiple crossing curves, the goal is to identify the correct endpoint from a marked start.

For example, a cubic Bézier curve can be written as

$$
B(t)=\sum_{k=0}^{3}{3 \choose k}(1-t)^{3-k}t^k P_k,
\qquad t\in[0,1]
$$

The model predicts a trajectory

$$
\hat{P} = (\hat{p}_1,\hat{p}_2,\ldots,\hat{p}_T)
$$

At an intersection, the model should follow local geometric continuity:

$$
\arg\min_j \angle\left(\hat{p}_t-\hat{p}_{t-1},\; p^{(j)}_{t+1}-\hat{p}_t\right)
$$

---

# Path Tracing Example

<div class="h_center v_center">
<img src="images/fig10_pointing_showcases.png" height="430px">
</div>

- identify the start marker
- produce intermediate points along the curve
- increase point density at difficult turns or crossings
- determine the final endpoint label

---

<!-- _class: section_title -->
# Post-Training Pipeline

---

<!-- class: fs24 -->

# Specialized SFT and RL

The paper first trains specialized models for grounding and pointing.

$$
\mathrm{FTwG}: \text{thinking with grounding}
$$

$$
\mathrm{FTwP}: \text{thinking with pointing}
$$

## Specialized SFT

- data mixture: roughly 70% general multimodal or text data, 30% specialized visual-primitive data
- box reasoning and point reasoning are trained separately at first

## Specialized RL

- RL is then applied independently to FTwG and FTwP
- separate reward models are designed for different task families

---

<!-- class: fs24 -->

# Reward Model Structure

For RL, the total reward can be written conceptually as

$$
R = \lambda_f R_{\mathrm{format}}
+ \lambda_q R_{\mathrm{quality}}
+ \lambda_a R_{\mathrm{accuracy}}
$$

## Format RM

- checks syntax of visual primitives
- discourages malformed outputs, duplicates, and looping behavior

## Quality RM

- checks consistency between reasoning trace and final answer
- penalizes self-contradiction or reward hacking

## Accuracy RM

- task-specific
- designed separately for counting, maze navigation, path tracing, etc.

---

<!-- class: fs24 -->

# Accuracy Reward for Counting

Exact match alone is too sparse, so the paper also uses a smooth reward based on relative error:

$$
R(\hat{y}, y) = \alpha \cdot
\exp\left(
-\beta \cdot \frac{|\hat{y}-y|}{|y|+1}
\right)
$$

- $\hat{y}$: predicted count
- $y$: ground-truth count
- $|y|+1$: relative-error normalization
- the paper reports $\alpha=0.7$ and $\beta=3$

<div class="memo">
When the true count is large, an error of 1 should be penalized less harshly than when the true count is small.
</div>

---

<!-- class: fs24 -->

# Accuracy Reward for Maze and Path Tasks

For maze navigation, the reward can be written as a weighted sum:

$$
R_{\mathrm{maze}} =
\lambda_1 R_{\mathrm{progress}}
+\lambda_2 R_{\mathrm{complete}}
+\lambda_3 R_{\mathrm{wall}}
+\lambda_4 R_{\mathrm{path}}
+\lambda_5 R_{\mathrm{answer}}
$$

If the first illegal move occurs at

$$
T^* = \min\{t \mid (v_t,v_{t+1})\notin E\}
$$

then later steps can be causally truncated.

A valid final path satisfies

$$
R_{\mathrm{path}} = \mathbf{1}\left[v_0=s,\;v_L=t,\;\forall \ell,(v_\ell,v_{\ell+1})\in E\right]
$$

For path tracing, a bidirectional geometric score is used:

$$
R_{\mathrm{traj}} = \frac{1}{2}\left(S(d(\hat{P}\rightarrow P)) + S(d(P\rightarrow \hat{P}))\right)
$$

---

<!-- class: fs24 -->

# Unified RFT and On-Policy Distillation

After specialized RL, the resulting expert models are

$$
\mathrm{ETwG}, \qquad \mathrm{ETwP}
$$

## Unified RFT

The paper builds a refined training set using rollout data:

$$
\mathcal{D}_{\mathrm{RFT}} =
\mathcal{D}_{\mathrm{normal}} \cup
0.05\cdot \mathcal{D}_{\mathrm{easy}}
$$

- keep all normal-level samples
- subsample 5% of easy-level samples
- reduce catastrophic forgetting

## On-Policy Distillation

The unified model is further trained with expert distributions:

$$
\mathcal{L}_{\mathrm{OPD}}(\theta)=
\sum_{i=1}^{N} w_i \cdot
D_{\mathrm{KL}}\left(\pi_{\theta} \,\Vert\, \pi_{E_i}\right)
$$

---

<!-- _class: section_title -->
# Experiments

---

<!-- class: fs24 -->

# Evaluation Setup

## Public Benchmarks

- Counting: CountQA, Pixmo-Count
- Spatial reasoning / VQA: SpatialMQA, CV-Bench, EmbSpatial, OmniSpatial, MIHBench

## In-House Benchmarks

- DS_Finegrained_Counting: 600 manually verified test cases
- DS_Spatial_Reasoning: multi-hop reasoning derived from CLEVR validation
- DS_Maze_Navigation: 2,000 instances
- DS_Path_Tracing: 2,000 instances

## Fair Evaluation Settings

- same prompts across compared APIs
- low-resolution images upscaled to 640,000 pixels when needed
- thinking budget normalized for models such as GPT and Gemini Flash

---

# Comparison with Frontier Models

<div class="h_center v_center">
<img src="images/table1_results.png" height="460px">
</div>

- The main gains are especially visible in counting, spatial reasoning, and topological reasoning.
- The strongest advantages appear on the topology-heavy tasks.

---

<!-- class: fs24 -->

# Key Experimental Observations

## Counting

- Pixmo-Count: 89.2 EM
- DS_Finegrained_Counting: 88.7 EM

## Spatial Reasoning / General VQA

- MIHBench: 85.3 ACC
- SpatialMQA: 69.4 ACC
- DS_Spatial_Reasoning: 98.7 ACC

## Topological Reasoning

- DS_Maze_Navigation: 66.9 ACC
- DS_Path_Tracing: 56.7 ACC

<div class="memo">
Even very strong frontier models remain relatively weak on topological reasoning, which supports the paper’s claim that point-based reasoning is particularly valuable there.
</div>

---

<!-- _class: section_title -->
# Qualitative Results

---

# Boxes as Visual Primitives

<div class="h_center v_center">
<img src="images/fig10_pointing_showcases.png" height="500px">
</div>

- object grounding
- dense counting
- grounded reasoning with explicit visual anchors

---

# Counting and Grounded Reasoning Examples

<div class="h_center v_center">
<img src="images/fig3_counting_examples.png" height="500px">
</div>

- candidate selection through boxes
- step-by-step verification
- less ambiguity than language-only counting

---

# Spatial and World-Knowledge Grounding

<div class="h_center v_center">
<img src="images/fig4_spatial_reasoning.png" height="500px">
</div>

- grounded object recognition
- relation-sensitive reasoning
- support for language answers that remain spatially grounded

---

# Points as Visual Primitives

<div class="h_center v_center">
<img src="images/fig5_maze_navigation.png" height="500px">
</div>

- maze navigation through explicit route traces
- path following through ordered point sequences
- topology handled as coordinates, not just text descriptions

---

<!-- _class: section_title -->
# Limitations and Discussion

---

<!-- class: fs24 -->

# Limitations

## 1. Input Resolution Still Matters

- In fine-grained scenes, primitives can still be inaccurate if the perceptual input is weak.
- Better perception modules could further help.

## 2. Reliance on Explicit Triggers

- The current system often needs explicit prompting to invoke visual-primitive reasoning.
- Ideally, the model should learn to activate this behavior automatically when needed.

## 3. Generalization of Point Reasoning

- The method works well for maze and path tasks.
- But broader cross-scenario topological generalization remains challenging.

---

<!-- class: fs24 -->

# What Is New Here?

<div class="card">
<b>Language-only CoT:</b> reasoning proceeds through phrases like “the small object in the upper left” or “the line next to it.” If the reference drifts once, later reasoning may completely collapse.
</div>

<div class="card">
<b>Thinking with visual primitives:</b> the reasoning state is anchored explicitly as “this box” or “this point.” This connects abstract reasoning in language to concrete structure in image space.
</div>

---

<!-- class: fs24 -->

# Summary

- **Problem:** MLLMs often fail not only in seeing, but also in **referencing**.

- **Core idea:** insert **points** and **bounding boxes** directly into the reasoning trace.

- **Why it works:** it gives the model an unambiguous visual anchor while it thinks.

- **Training pipeline:**
  - normalized coordinate outputs,
  - large-scale grounding pretraining,
  - task-specific cold-start data,
  - specialized SFT/RL,
  - unified RFT,
  - on-policy distillation.

- **Conclusion:** for multimodal System-2-like reasoning, a precise reference mechanism may matter as much as—or more than—simply adding more pixels.

---

<!-- _class: fs32 -->

# Takeaway

## Think in language, but anchor in vision.

Visual primitives make multimodal reasoning less like describing an image from memory, and more like actively pointing, tracing, checking, and counting inside the image.

---

<!-- _class: section_title -->
# Appendix: Useful Equations

---

<!-- class: fs24 -->

# Appendix: Visual Primitive Sequence

A visual reasoning trace can be written as

$$
\tau = \{(s_t, v_t)\}_{t=1}^{T}
$$

where

- $s_t$: language state or step description
- $v_t$: visual primitive

$$
v_t =
\begin{cases}
[x_t,y_t] & \text{point} \\
[x_{1,t},y_{1,t},x_{2,t},y_{2,t}] & \text{box}
\end{cases}
$$

The final answer is conditioned on both language and primitive history:

$$
a \sim p_\theta(a \mid I,q,s_{1:T},v_{1:T})
$$

---

<!-- class: fs24 -->

# Appendix: Primitive-Grounded Counting and Paths

Given grounded boxes $B=\{b_i\}_{i=1}^{m}$ and a target predicate $\phi_q(\cdot)$,

$$
\hat{y}=\sum_{i=1}^{m}\mathbf{1}[\phi_q(b_i,I)=1]
$$

Exact-match correctness is

$$
\mathrm{EM}=\mathbf{1}[\hat{y}=y]
$$

Smooth counting reward:

$$
R(\hat{y}, y) = \alpha \exp\left(-\beta \frac{|\hat{y}-y|}{|y|+1}\right)
$$

For path quality, the bidirectional discrepancy is

$$
D_{\mathrm{bi}}(\hat{P},P)=
\frac{1}{2}\left[d(\hat{P}\rightarrow P)+d(P\rightarrow \hat{P})\right]
$$

A smaller $D_{\mathrm{bi}}$ means the predicted path stays close to the true trajectory without skipping key segments.
