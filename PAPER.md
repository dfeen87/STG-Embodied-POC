# Deterministic Spiral-Time Governance for Hallucination Suppression in LLM-Controlled Climbing and Walking Robots

**Historical manuscript record — clarification dated 29 October 2026.** The historical title, numerical tables, equations, and illustrations are retained with interpretation limits at each claim. This is not the latest journal Word draft. The 11.4/4.7, 12.4/5.0, and 13.24/4.87 reports have not been reconciled as the same protocol or seed population. None establishes an independently verified physical-safety benefit. See [Figure 3 provenance](docs/figure3/README.md) for the source mapping and unmodified original wording in the pinned source revision.


**Marcel Krüger¹ · Don Michael Feeney Jr.²**

¹ Independent Researcher, Germany

² Independent Researcher, USA

Corresponding author: marcelkrueger092@gmail.com

ORCID (M.K.): 0009-0002-5709-9729 

ORCID (D.M.F.): 0009-0003-1350-4160

*Journal of Climbing and Walking Robots · 2026 · Vol. XX(XX) · pp. 1–XX*
DOI: *(assigned by journal)*

---

## Abstract

Large Language Models (LLMs) are increasingly integrated into robotic autonomy stacks for semantic planning and high-level decision support. In climbing and walking robots, long-horizon deployment is constrained by hallucination drift, retrieval instability, and non-deterministic reasoning variance, which can propagate into physically unsafe locomotion behavior.

We introduce a deterministic external governance layer based on a Spiral-Time operator formalism. Interaction history is embedded into a triadic state ψ(t) = t + iϕ(t) + jχ(t), where ϕ(t) encodes contextual coherence and χ(t) = ∂ₜϕ(t) captures temporal torsion associated with abrupt divergence. A scalar instability functional ΔΦ(t) deterministically regulates memory writes, retrieval widening, verification mode, and safe fallback switching.

We define hallucination operationally as disagreement between generated claims and simulator-derived checks and specify a deterministic supervisory algorithm. The historical Lyapunov argument concerns an assumed auxiliary linear model; it has not been proved to bound the implemented hybrid governor or robot dynamics. Archived protocols, intervals, and ablations are reported with unresolved original run provenance.

The archive includes a synthetic testbed and a MuJoCo quadruped proof of concept. Their differing diagnostic summaries are not interchangeable, and the mode-conditioned counter does not independently measure unsafe executed motion. No verified reduction in physical hazards, unchanged hallucination generation, or transfer benefit follows from these historical aggregates.

The design leaves LLM weights unchanged and provides inspectable mode-selection logic. Deployment suitability, fallback safety, robustness, and independent physical-safety benefit remain unestablished.

**Keywords:** legged robotics · LLM-based control · hallucination detection · deterministic supervision · MuJoCo proof of concept

---

## 1 · Introduction

Climbing and walking robots operate under strong contact constraints, terrain discontinuities, and safety-critical failure modes (falls, self-collision, uncontrolled impacts). The stability perspective adopted here is conceptually aligned with barrier-function and Lyapunov-based safety formulations in control theory [1], although the present governor operates at the **semantic decision layer** rather than continuous torque control.

Recent robotics research increasingly integrates LLMs as high-level planners, semantic interfaces, and vision-language-action controllers. While such models enable flexible task abstraction and semantic reasoning, their generative nature introduces instability risks: hallucinated environment claims, inconsistent tool outputs, and non-deterministic subgoal sequences can propagate into physically unsafe locomotion behavior — especially under partial observability and long-horizon deployment.

Confidence scoring, retrieval augmentation, post-hoc verification, and established runtime-assurance methods address different interfaces. The present threshold logic is deterministic, but this alone does not establish bounded hybrid dynamics or robot safety, nor a superiority claim over those methods.

This paper targets a specific failure class: hallucinated assertions or action recommendations that are inconsistent with verified robot state and constraints. We propose a **deterministic Spiral-Time Governor** that wraps a black-box LLM and enforces auditable threshold-based mode switching.

<!-- FIGURE: Mode Switching Overview -->
<svg width="100%" viewBox="0 0 680 160" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arr1" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>
  <!-- EXECUTE box -->
  <rect x="40" y="30" width="170" height="90" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1.2"/>
  <text x="125" y="60" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#1b5e20">EXECUTE</text>
  <text x="125" y="80" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#2e7d32">Normal operation</text>
  <text x="125" y="97" text-anchor="middle" font-family="monospace" font-size="10" fill="#388e3c">ΔΦ(t) &lt; τ₁</text>
  <!-- VERIFY box -->
  <rect x="255" y="30" width="170" height="90" rx="10" fill="#fff8e1" stroke="#f57f17" stroke-width="1.2"/>
  <text x="340" y="60" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#e65100">VERIFY</text>
  <text x="340" y="80" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#f57f17">Cross-check &amp; validate</text>
  <text x="340" y="97" text-anchor="middle" font-family="monospace" font-size="10" fill="#fb8c00">τ₁ ≤ ΔΦ(t) &lt; τ₂</text>
  <!-- SAFE box -->
  <rect x="470" y="30" width="170" height="90" rx="10" fill="#ffebee" stroke="#c62828" stroke-width="1.2"/>
  <text x="555" y="60" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#b71c1c">SAFE</text>
  <text x="555" y="80" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#c62828">Freeze + fallback</text>
  <text x="555" y="97" text-anchor="middle" font-family="monospace" font-size="10" fill="#e53935">ΔΦ(t) ≥ τ₂</text>
  <!-- arrows -->
  <line x1="210" y1="75" x2="253" y2="75" stroke="#888" stroke-width="1.2" marker-end="url(#arr1)"/>
  <line x1="425" y1="75" x2="468" y2="75" stroke="#888" stroke-width="1.2" marker-end="url(#arr1)"/>
  <!-- ΔΦ axis label -->
  <text x="340" y="148" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#555">ΔΦ(t) increases →  instability grows →  more conservative mode</text>
</svg>

*Illustration A: Three threshold-selected software modes. SAFE names a fallback mode; it does not certify the robot state or the fallback controller. Thresholds satisfy 0 < τ₁ < τ₂ < 1.*

### Contributions

1. **Operational hallucination definition** for robotics: falsifiable mismatch between LLM claims and a ground-truth oracle derived from simulator state.
2. **Deterministic governance layer**: triadic Spiral-Time embedding ψ(t) = t + iϕ(t) + jχ(t) and instability functional ΔΦ(t) with transparent thresholds and auditable switching logic.
3. **Limited theoretical scope**: normalization bounds the supervisory variables; the archived Lyapunov argument assumes a separate linear model and is not a proved governor or robot-safety certificate.
4. **Historical evaluation records**: separate synthetic and MuJoCo proxy summaries, including reported seeds, ablations, statistics, and overhead. Their original provenance and independent safety interpretation remain unresolved.

---

## 2 · Problem Setup and Operational Hallucination Definition

### 2.1 · Robot Autonomy Stack Abstraction

Let the physical robot (or simulator) have state sₜ ∈ 𝒮, measured outputs yₜ ∈ 𝒴, and low-level controller uₜ ∈ 𝒰. A classical autonomy stack produces high-level intents/goals gₜ and low-level actions uₜ subject to safety constraints 𝒞 (contacts, friction, joint/torque limits).

An LLM-based module proposes high-level outputs â_t (tool calls, subgoals, textual claims, or command suggestions). The governor 𝒢 is a **deterministic supervisor** that decides whether â_t is admissible or must be verified/blocked.

<!-- FIGURE: Robot Autonomy Stack -->
<svg width="100%" viewBox="0 0 680 300" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arr2" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>
  <!-- outer container -->
  <rect x="30" y="15" width="620" height="265" rx="14" fill="none" stroke="#b0bec5" stroke-width="1" stroke-dasharray="5 4"/>
  <text x="50" y="36" font-family="sans-serif" font-size="11" fill="#78909c">Physical Robot / Simulator</text>
  <!-- Robot state box -->
  <rect x="55" y="50" width="160" height="56" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="1"/>
  <text x="135" y="73" text-anchor="middle" font-family="monospace" font-size="12" font-weight="600" fill="#0d47a1">sₜ ∈ 𝒮</text>
  <text x="135" y="92" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#1565c0">Robot state</text>
  <!-- Measured outputs -->
  <rect x="255" y="50" width="160" height="56" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="1"/>
  <text x="335" y="73" text-anchor="middle" font-family="monospace" font-size="12" font-weight="600" fill="#0d47a1">yₜ ∈ 𝒴</text>
  <text x="335" y="92" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#1565c0">Measured outputs</text>
  <!-- Low-level controller -->
  <rect x="455" y="50" width="160" height="56" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="1"/>
  <text x="535" y="73" text-anchor="middle" font-family="monospace" font-size="12" font-weight="600" fill="#0d47a1">uₜ ∈ 𝒰</text>
  <text x="535" y="92" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#1565c0">Low-level controller</text>
  <!-- arrows top row -->
  <line x1="215" y1="78" x2="253" y2="78" stroke="#1565c0" stroke-width="1.2" marker-end="url(#arr2)"/>
  <line x1="415" y1="78" x2="453" y2="78" stroke="#1565c0" stroke-width="1.2" marker-end="url(#arr2)"/>
  <!-- LLM Module -->
  <rect x="155" y="165" width="170" height="60" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="1"/>
  <text x="240" y="188" text-anchor="middle" font-family="sans-serif" font-size="12" font-weight="600" fill="#4a148c">LLM Module</text>
  <text x="240" y="206" text-anchor="middle" font-family="monospace" font-size="10" fill="#6a1b9a">â_t claims, subgoals</text>
  <text x="240" y="220" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#6a1b9a">action proposals</text>
  <!-- Governor -->
  <rect x="355" y="165" width="180" height="60" rx="8" fill="#fff3e0" stroke="#e65100" stroke-width="1.5"/>
  <text x="445" y="188" text-anchor="middle" font-family="sans-serif" font-size="12" font-weight="700" fill="#bf360c">Governor 𝒢</text>
  <text x="445" y="206" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#e64a19">Deterministic supervisor</text>
  <text x="445" y="220" text-anchor="middle" font-family="monospace" font-size="10" fill="#e64a19">admissible?</text>
  <!-- arrow: yₜ down to LLM -->
  <line x1="335" y1="106" x2="285" y2="163" stroke="#888" stroke-width="1" stroke-dasharray="4 3" marker-end="url(#arr2)"/>
  <!-- arrow: LLM to Governor -->
  <line x1="325" y1="195" x2="353" y2="195" stroke="#6a1b9a" stroke-width="1.2" marker-end="url(#arr2)"/>
  <!-- arrow: Governor to uₜ -->
  <line x1="535" y1="163" x2="535" y2="108" stroke="#e65100" stroke-width="1.2" marker-end="url(#arr2)"/>
  <line x1="445" y1="163" x2="480" y2="130" stroke="#e65100" stroke-width="1" stroke-dasharray="3 3" marker-end="url(#arr2)"/>
  <!-- label -->
  <text x="340" y="280" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#555">Illustration B: Robot autonomy stack with LLM module and deterministic Governor 𝒢.</text>
</svg>

### 2.2 · Oracle-Based Hallucination Definition

In simulation we have a ground-truth oracle 𝒪(sₜ) providing verifiable predicates (pose, contact set, terrain class, feasibility flags). Let the LLM output at time t include claims Kₜ = {kₜ,₁, …, kₜ,mₜ} and proposed action â_t.

Define a verification map **ver(kₜ,ⱼ) ∈ {0, 1}** — equals 1 iff kₜ,ⱼ is consistent with 𝒪(sₜ) and the specified checks. Certification of monitors or physical sensing is an assumption for a future deployment, not an outcome of this proof of concept.

**Hallucination rate over horizon T:**

$$H_T := \frac{1}{\sum_{t=1}^{T} m_t} \sum_{t=1}^{T}\sum_{j=1}^{m_t} \bigl(1 - \text{ver}(k_{t,j})\bigr)$$

Thus hallucination is defined as a **testable inconsistency** — not a probabilistic confidence estimate.

---

## 3 · Spiral-Time Governor: State, Instability, and Gating

### 3.1 · Triadic Spiral-Time Embedding

We define the triadic state: **ψ(t) = t + iϕ(t) + jχ(t)**, where χ(t) := ϕ(t) − ϕ(t−1).

Here ϕ(t) ∈ [0, 1] is a deterministic coherence score and χ(t) captures abrupt coherence changes (temporal torsion).

<!-- FIGURE: Spiral-Time State Space -->
<svg width="100%" viewBox="0 0 680 300" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arr3" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>
  <!-- Background panel -->
  <rect x="30" y="20" width="380" height="255" rx="12" fill="#fafafa" stroke="#e0e0e0" stroke-width="1"/>
  <!-- Axes -->
  <!-- t axis (diagonal = depth illusion) -->
  <line x1="120" y1="230" x2="60" y2="270" stroke="#555" stroke-width="1.5" marker-end="url(#arr3)"/>
  <text x="42" y="280" font-family="serif" font-size="13" font-style="italic" fill="#333">t</text>
  <!-- iϕ axis (horizontal right) -->
  <line x1="120" y1="230" x2="340" y2="230" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arr3)"/>
  <text x="348" y="234" font-family="serif" font-size="13" font-style="italic" fill="#1565c0">iϕ(t)</text>
  <!-- jχ axis (vertical up) -->
  <line x1="120" y1="230" x2="120" y2="50" stroke="#6a1b9a" stroke-width="1.5" marker-end="url(#arr3)"/>
  <text x="96" y="44" font-family="serif" font-size="13" font-style="italic" fill="#6a1b9a">jχ(t)</text>
  <!-- Origin dot -->
  <circle cx="120" cy="230" r="3" fill="#333"/>
  <text x="100" y="248" font-family="sans-serif" font-size="10" fill="#555">origin</text>
  <!-- Sample ψ(t) point -->
  <circle cx="248" cy="118" r="7" fill="#e65100" stroke="#fff" stroke-width="2"/>
  <text x="260" y="113" font-family="serif" font-size="13" font-style="italic" fill="#e65100">ψ(t)</text>
  <!-- dashed projection lines -->
  <line x1="248" y1="118" x2="248" y2="230" stroke="#1565c0" stroke-width="1" stroke-dasharray="4 3"/>
  <line x1="120" y1="118" x2="248" y2="118" stroke="#6a1b9a" stroke-width="1" stroke-dasharray="4 3"/>
  <line x1="120" y1="230" x2="248" y2="118" stroke="#e65100" stroke-width="1.5" stroke-dasharray="3 2"/>
  <!-- coordinate labels -->
  <text x="184" y="245" font-family="monospace" font-size="10" fill="#1565c0">ϕ(t)</text>
  <text x="68" y="175" font-family="monospace" font-size="10" fill="#6a1b9a">χ(t)</text>
  <!-- formula panel -->
  <rect x="430" y="30" width="225" height="235" rx="10" fill="#fff8e1" stroke="#f9a825" stroke-width="1"/>
  <text x="542" y="58" text-anchor="middle" font-family="sans-serif" font-size="12" font-weight="700" fill="#e65100">Triadic State</text>
  <text x="542" y="82" text-anchor="middle" font-family="serif" font-size="13" font-style="italic" fill="#333">ψ(t) = t + iϕ(t) + jχ(t)</text>
  <line x1="448" y1="95" x2="637" y2="95" stroke="#f9a825" stroke-width="0.8"/>
  <text x="448" y="116" font-family="sans-serif" font-size="11" font-weight="600" fill="#333">Components:</text>
  <text x="448" y="136" font-family="monospace" font-size="10" fill="#333">t      — real time</text>
  <text x="448" y="156" font-family="monospace" font-size="10" fill="#1565c0">iϕ(t) — coherence</text>
  <text x="448" y="176" font-family="monospace" font-size="10" fill="#6a1b9a">jχ(t) — torsion</text>
  <line x1="448" y1="190" x2="637" y2="190" stroke="#f9a825" stroke-width="0.8"/>
  <text x="448" y="210" font-family="sans-serif" font-size="11" font-weight="600" fill="#333">Torsion:</text>
  <text x="448" y="230" font-family="serif" font-size="12" font-style="italic" fill="#6a1b9a">χ(t) = ϕ(t) − ϕ(t−1)</text>
  <text x="448" y="252" font-family="sans-serif" font-size="10" fill="#555">captures abrupt divergence</text>
  <!-- caption -->
  <text x="340" y="288" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#555">Illustration C: Triadic Spiral-Time state space. ψ(t) traces a path as coherence and torsion evolve.</text>
</svg>

### 3.2 · Deterministic Coherence Components

We use deterministic deviations inspired by the Structure–Information–Coherence (S–I–C) framework:

| Component | Symbol | Range | Captures |
|---|---|---|---|
| Structure deviation | ΔR(t) | [0, 1] | Constraint/plan mismatch, inadmissible tools, feasibility failures |
| Information deviation | ΔI(t) | [0, 1] | Claim mismatch against oracle/telemetry |
| Coherence deviation | ΔC(t) | [0, 1] | Contradiction score vs. verified memory window |

**Information deviation:** ΔI(t) := (1/mₜ) · Σⱼ (1 − ver(kₜ,ⱼ))

**Coherence score:** ϕ(t) := 1 − (wR·ΔR(t) + wI·ΔI(t) + wC·ΔC(t)),   where wR + wI + wC = 1

### 3.3 · Instability Functional

$$\boxed{\Delta\Phi(t) := \alpha\,\Delta R(t) + \beta\,\Delta I(t) + \gamma\,\Delta C(t) + \delta\,|\chi(t)|, \quad \alpha+\beta+\gamma+\delta = 1}$$

<!-- FIGURE: Instability Functional Decomposition -->
<svg width="100%" viewBox="0 0 680 220" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arr4" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>
  <!-- ΔR component -->
  <rect x="40" y="30" width="118" height="64" rx="8" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1"/>
  <text x="99" y="54" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#1b5e20">ΔR(t)</text>
  <text x="99" y="72" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#2e7d32">Structure</text>
  <text x="99" y="86" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#388e3c">weight α</text>
  <!-- ΔI component -->
  <rect x="185" y="30" width="118" height="64" rx="8" fill="#e3f2fd" stroke="#1565c0" stroke-width="1"/>
  <text x="244" y="54" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#0d47a1">ΔI(t)</text>
  <text x="244" y="72" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#1565c0">Information</text>
  <text x="244" y="86" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#1976d2">weight β</text>
  <!-- ΔC component -->
  <rect x="330" y="30" width="118" height="64" rx="8" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="1"/>
  <text x="389" y="54" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#4a148c">ΔC(t)</text>
  <text x="389" y="72" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#6a1b9a">Coherence</text>
  <text x="389" y="86" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#7b1fa2">weight γ</text>
  <!-- |χ(t)| component -->
  <rect x="475" y="30" width="118" height="64" rx="8" fill="#fff3e0" stroke="#e65100" stroke-width="1"/>
  <text x="534" y="54" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#bf360c">|χ(t)|</text>
  <text x="534" y="72" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#e65100">Torsion</text>
  <text x="534" y="86" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#f57c00">weight δ</text>
  <!-- arrows down to ΔΦ -->
  <line x1="99"  y1="94" x2="310" y2="158" stroke="#2e7d32" stroke-width="1" marker-end="url(#arr4)"/>
  <line x1="244" y1="94" x2="320" y2="158" stroke="#1565c0" stroke-width="1" marker-end="url(#arr4)"/>
  <line x1="389" y1="94" x2="340" y2="158" stroke="#6a1b9a" stroke-width="1" marker-end="url(#arr4)"/>
  <line x1="534" y1="94" x2="360" y2="158" stroke="#e65100" stroke-width="1" marker-end="url(#arr4)"/>
  <!-- ΔΦ output box -->
  <rect x="265" y="158" width="150" height="44" rx="8" fill="#212121" stroke="#424242" stroke-width="1.5"/>
  <text x="340" y="177" text-anchor="middle" font-family="monospace" font-size="13" font-weight="700" fill="#fff">ΔΦ(t)</text>
  <text x="340" y="194" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#bdbdbd">instability scalar</text>
  <text x="340" y="214" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#777">Illustration D: ΔΦ(t) is a weighted sum of four deterministic deviation signals (α+β+γ+δ=1).</text>
</svg>

### 3.4 · Deterministic Mode Switching

Thresholds 0 < τ₁ < τ₂ < 1 define:

| Condition | Mode |
|---|---|
| ΔΦ(t) < τ₁ | **EXECUTE** |
| τ₁ ≤ ΔΦ(t) < τ₂ | **VERIFY** |
| ΔΦ(t) ≥ τ₂ | **SAFE** |



---

## 4 · Algorithm

**Algorithm 1: Deterministic Spiral-Time Governor (STG)**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Require: Telemetry yₜ, oracle 𝒪 (sim); validated real monitors remain future work
 Require: LLM proposal (â_t, Kₜ)
 Require: Fixed (wR, wI, wC), (α, β, γ, δ), thresholds (τ₁, τ₂)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  1:  Compute ΔR(t)  ◀  feasibility/constraint checks
  2:  Compute ΔI(t)  ◀  claim verification ver(kₜ,ⱼ) vs oracle
  3:  Compute ΔC(t)  ◀  contradiction detector on memory window
  4:  ϕ(t)  ←  1 − (wR·ΔR + wI·ΔI + wC·ΔC)
  5:  χ(t)  ←  ϕ(t) − ϕ(t − 1)
  6:  ΔΦ(t) ←  α·ΔR + β·ΔI + γ·ΔC + δ·|χ(t)|
  7:  if ΔΦ(t) < τ₁  →  M(t) ← EXECUTE ; allow â_t if constraints pass
  8:  elif ΔΦ(t) < τ₂ →  M(t) ← VERIFY  ; cross-check; block irreversible
  9:  else            →  M(t) ← SAFE    ; freeze writes; trigger fallback
 10:  Log (t, ϕ(t), χ(t), ΔΦ(t), M(t))  →  immutable audit trail
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 5 · Deterministic Governance Table

**Table 1:** Intended gate actions. Deterministic mode selection does not establish physical safety or certification of the listed behavior.

| Condition | Mode | Deterministic Action |
|---|:---:|---|
| ΔΦ(t) < τ₁ | **EXECUTE** | Allow LLM-issued tool calls/subgoals only if constraints pass; normal operation. |
| τ₁ ≤ ΔΦ(t) < τ₂ | **VERIFY** | Run planner feasibility + monitor agreement; widen retrieval; block irreversible commands until verified. |
| ΔΦ(t) ≥ τ₂ | **SAFE** | Freeze memory writes; request predefined fallback. Physical safety and certification of any halt, gait, or return behavior require separate validation. |

---

## 6 · Architecture

<!-- FIGURE: System Architecture -->
<svg width="100%" viewBox="0 0 680 360" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <marker id="arr5" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>
  <!-- Telemetry box -->
  <rect x="40" y="50" width="150" height="80" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="1.2"/>
  <text x="115" y="78" text-anchor="middle" font-family="sans-serif" font-size="12" font-weight="700" fill="#0d47a1">Telemetry</text>
  <text x="115" y="97" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#1565c0">Perception / SLAM</text>
  <text x="115" y="113" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#1565c0">Contacts / IMU</text>
  <!-- Spiral-Time Governor (central) -->
  <rect x="240" y="30" width="200" height="130" rx="12" fill="#fff8e1" stroke="#f9a825" stroke-width="2"/>
  <text x="340" y="58" text-anchor="middle" font-family="sans-serif" font-size="13" font-weight="700" fill="#e65100">Spiral-Time Governor</text>
  <line x1="258" y1="68" x2="422" y2="68" stroke="#f9a825" stroke-width="0.8"/>
  <text x="340" y="87" text-anchor="middle" font-family="monospace" font-size="11" fill="#555">ϕ(t)  coherence score</text>
  <text x="340" y="103" text-anchor="middle" font-family="monospace" font-size="11" fill="#555">χ(t)  torsion</text>
  <text x="340" y="119" text-anchor="middle" font-family="monospace" font-size="11" fill="#555">ΔΦ(t) instability</text>
  <rect x="262" y="131" width="156" height="20" rx="5" fill="#212121"/>
  <text x="340" y="145" text-anchor="middle" font-family="monospace" font-size="10" fill="#fff">EXECUTE · VERIFY · SAFE</text>
  <!-- LLM Agent -->
  <rect x="490" y="50" width="150" height="80" rx="10" fill="#f3e5f5" stroke="#6a1b9a" stroke-width="1.2"/>
  <text x="565" y="78" text-anchor="middle" font-family="sans-serif" font-size="12" font-weight="700" fill="#4a148c">LLM Agent</text>
  <text x="565" y="97" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#6a1b9a">black-box</text>
  <text x="565" y="113" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#6a1b9a">claims + proposals</text>
  <!-- Planner/Supervisor -->
  <rect x="340" y="225" width="160" height="60" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1.2"/>
  <text x="420" y="250" text-anchor="middle" font-family="sans-serif" font-size="12" font-weight="700" fill="#1b5e20">Planner / Supervisor</text>
  <text x="420" y="272" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#2e7d32">goal refinement</text>
  <!-- Low-level Controller -->
  <rect x="340" y="305" width="160" height="44" rx="10" fill="#ffebee" stroke="#c62828" stroke-width="1.2"/>
  <text x="420" y="326" text-anchor="middle" font-family="sans-serif" font-size="12" font-weight="700" fill="#b71c1c">Low-level Controller</text>
  <text x="420" y="342" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#c62828">PID / torque</text>
  <!-- Arrows -->
  <!-- Telemetry → Governor -->
  <line x1="190" y1="90" x2="238" y2="90" stroke="#1565c0" stroke-width="1.5" marker-end="url(#arr5)"/>
  <!-- Governor ↔ LLM -->
  <line x1="440" y1="80" x2="488" y2="80" stroke="#f9a825" stroke-width="1.5" marker-end="url(#arr5)"/>
  <line x1="490" y1="100" x2="442" y2="100" stroke="#6a1b9a" stroke-width="1.5" stroke-dasharray="4 3" marker-end="url(#arr5)"/>
  <!-- Governor → Planner -->
  <line x1="340" y1="160" x2="390" y2="223" stroke="#f9a825" stroke-width="1.5" marker-end="url(#arr5)"/>
  <!-- Planner → Controller -->
  <line x1="420" y1="285" x2="420" y2="303" stroke="#2e7d32" stroke-width="1.5" marker-end="url(#arr5)"/>
  <!-- Controller feedback → Telemetry (curved) -->
  <path d="M340 327 Q180 330 115 132" fill="none" stroke="#c62828" stroke-width="1" stroke-dasharray="5 3" marker-end="url(#arr5)"/>
  <text x="190" y="340" font-family="sans-serif" font-size="10" fill="#c62828">feedback</text>
  <!-- Legend labels -->
  <text x="340" y="355" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#555">Illustration E: Governor-centered architecture. The STG gates all LLM outputs before execution. (See Figure 1 in paper.)</text>
</svg>

---

## 7 · Historical Auxiliary Linear-Model Argument (No Robot-Safety Certificate)

### 7.1 · Unproved Link to an Auxiliary Linear Model

The historical notation identifies **x(t) := (ϕ(t), χ(t))ᵀ ∈ ℝ²** and writes the linear relation below. Normalization/clamping does not derive that relation for the implemented hybrid governor. The matrices A and B must therefore be treated as assumptions of a separate auxiliary model, not as an established envelope for governor or robot trajectories:

$$x(t+1) = A\,x(t) + B\,\eta(t), \qquad \|\eta(t)\| \leq \bar{\eta}$$

Assume **ρ(A) < 1** (spectral radius strictly less than 1).

### 7.2 · Archived Lyapunov Construction and Its Limits

For an auxiliary linear model satisfying the assumed **ρ(A) < 1**, choose **Q ≻ 0** (e.g., Q = I). A unique **P ≻ 0** then solves **P − AᵀPA = Q**. This construction does not establish the assumed model's relationship to the implemented governor.

Let **V(x) = xᵀPx**.

> **Historical bound claim (not a proved governor or robot theorem).** The following spectral-radius-rate expression is retained for provenance; its general validity and application to the hybrid governor are not established here.
> The archived argument assumes ρ(A) < 1 and ‖η(t)‖ ≤ η̄ and asserts constants c₀, c₁ > 0 in:
>
> $$\boxed{\|x(t)\| \leq c_0\,\rho(A)^t\,\|x(0)\| + c_1\,\bar{\eta}, \qquad \forall\, t \geq 0}$$

*Archived proof sketch.* Expand V(x(t+1)) for the assumed auxiliary linear system and substitute P − AᵀPA = Q. The sketch does not derive that system from the governor or justify the displayed spectral-radius-rate expression in general. Only the elementary bounds of normalized supervisory variables follow directly from their definitions; neither those bounds nor this sketch certify plant stability, physical safety, or fallback performance.

<!-- FIGURE: Lyapunov Stability Decay -->
<svg width="100%" viewBox="0 0 680 240" xmlns="http://www.w3.org/2000/svg">
  <title>Historical illustration with unverified original provenance</title>
  <desc>Archived values or assumed auxiliary-model behavior only. This illustration does not establish independently measured physical-safety improvement, component necessity, robustness, or generator equivalence.</desc>
  <defs>
    <marker id="arr6" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>
  <!-- Plot area -->
  <rect x="60" y="20" width="500" height="175" rx="8" fill="#fafafa" stroke="#e0e0e0" stroke-width="1"/>
  <!-- Axes -->
  <line x1="80" y1="180" x2="540" y2="180" stroke="#333" stroke-width="1.5" marker-end="url(#arr6)"/>
  <line x1="80" y1="180" x2="80" y2="30"  stroke="#333" stroke-width="1.5" marker-end="url(#arr6)"/>
  <text x="548" y="184" font-family="serif" font-size="13" font-style="italic" fill="#333">t</text>
  <text x="70"  y="26"  font-family="serif" font-size="13" font-style="italic" fill="#333">‖x‖</text>
  <!-- y-axis ticks -->
  <line x1="76" y1="50"  x2="84" y2="50"  stroke="#555" stroke-width="1"/>
  <line x1="76" y1="120" x2="84" y2="120" stroke="#555" stroke-width="1"/>
  <line x1="76" y1="155" x2="84" y2="155" stroke="#555" stroke-width="1"/>
  <text x="58" y="54"  font-family="monospace" font-size="10" fill="#555" text-anchor="end">‖x(0)‖·c₀</text>
  <text x="58" y="124" font-family="monospace" font-size="10" fill="#555" text-anchor="end">mid</text>
  <text x="58" y="159" font-family="monospace" font-size="10" fill="#c62828" text-anchor="end">c₁·η̄</text>
  <!-- Exponential decay curve -->
  <polyline points="80,50 120,72 160,92 200,108 240,120 280,130 320,138 360,144 400,149 440,153 480,156 520,157" fill="none" stroke="#1565c0" stroke-width="2.2"/>
  <!-- Residual bound line -->
  <line x1="80" y1="155" x2="530" y2="155" stroke="#c62828" stroke-width="1.5" stroke-dasharray="6 4"/>
  <!-- Shaded safe zone -->
  <rect x="80" y="155" width="450" height="25" fill="#ffebee" opacity="0.5"/>
  <!-- Annotations -->
  <text x="210" y="90" font-family="sans-serif" font-size="11" fill="#1565c0">c₀ · ρ(A)ᵗ · ‖x(0)‖</text>
  <text x="210" y="104" font-family="sans-serif" font-size="10" fill="#1565c0">assumed auxiliary decay</text>
  <text x="380" y="150" font-family="sans-serif" font-size="10" fill="#c62828">c₁·η̄  (bounded residual)</text>
  <!-- Description -->
  <text x="310" y="218" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#555">Illustration F: Archived auxiliary-model sketch; no governor or robot-safety certificate.</text>
</svg>

---

## 8 · Simulation Environment

The historical primary evaluation uses a synthetic stochastic testbed described as a deterministic JavaScript implementation with a parametric noise model in place of physical dynamics. Section 8.2 records a separate MuJoCo proxy. Neither software determinism nor plotted aggregates establish exact reconstruction of missing historical observations or independently verified physical-safety improvement.

> **Synthetic Testbed Disclaimer.** These are historical synthetic diagnostics. The separate MuJoCo report in §8.2 does not retroactively validate them or establish transfer of a physical-safety benefit.

The historical protocol reports five experimental conditions (Baseline LLM, LLM+RAG, LLM+Governor, Ablation A with δ = 0, and Ablation B always-execute), three tasks (T1: Climb, T2: Stairs, T3: Gap), 120 steps per episode, N = 30 independent random seeds per condition/task, 90 episodes per condition, and 54,000 total simulated steps. These claimed populations have not been authenticated against complete original records; the LLM+RAG label does not establish a functioning retrieval pipeline.

### 8.1 · Statistical Analysis

The historical analysis reports **α = 0.05** (two-tailed), bootstrap percentile CI (B = 2,000, seed 77), Mann–Whitney U tests, Holm step-down, rank-biserial r, and Cliff's δ. Original observations and intervals have not been independently reconstructed here; these summaries do not establish statistical or physical-safety conclusions for the current implementation.

**Table 2:** Archived synthetic metrics (reported mean [95% CI], claimed N = 90 per condition; provenance and intervals unverified). Lower recorded H_T may reflect claim-stream selection or sampling and does not establish altered LLM generation. The diagnostic counter is not an independently measured physical outcome.

| Condition | Archived H_T | Archived diagnostic counter | Recorded success (%) | Archived action variance |
|---|---|---|---|---|
| Baseline LLM | 0.4595 [0.4491, 0.4702] | 13.24 [12.59, 13.96] | 34.4 [24.4, 44.4] | 0.0472 [0.0465, 0.0480] |
| LLM+RAG | 0.3679 [0.3585, 0.3775] | 12.56 [11.94, 13.18] | 38.9 [28.9, 48.9] | 0.0470 [0.0463, 0.0477] |
| **LLM+Governor** | **0.2200 [0.2139, 0.2266]** | **12.59 [11.82, 13.38]** | **41.1 [31.1, 51.1]** | **0.0474 [0.0466, 0.0481]** |
| Ablation A (δ=0) | 0.2422 [0.2350, 0.2497] | 13.79 [13.03, 14.49] | 30.0 [21.1, 38.9] | 0.0475 [0.0468, 0.0482] |
| Ablation B (always-exec) | 0.4369 [0.4270, 0.4473] | 24.68 [23.80, 25.57] | 0.0 [0.0, 0.0] | 0.0477 [0.0470, 0.0485] |

<!-- FIGURE: H_T Bar Chart -->
<svg width="100%" viewBox="0 0 680 260" xmlns="http://www.w3.org/2000/svg">
  <title>Historical illustration with unverified original provenance</title>
  <desc>Archived values or assumed auxiliary-model behavior only. This illustration does not establish independently measured physical-safety improvement, component necessity, robustness, or generator equivalence.</desc>
  <!-- Y-axis -->
  <line x1="80" y1="20" x2="80" y2="200" stroke="#333" stroke-width="1.5"/>
  <line x1="80" y1="200" x2="640" y2="200" stroke="#333" stroke-width="1.5"/>
  <!-- Y gridlines and labels -->
  <line x1="78" y1="200" x2="82" y2="200" stroke="#333" stroke-width="1.2"/>
  <line x1="78" y1="163" x2="82" y2="163" stroke="#333" stroke-width="1"/>
  <line x1="78" y1="126" x2="82" y2="126" stroke="#333" stroke-width="1"/>
  <line x1="78" y1="89"  x2="82" y2="89"  stroke="#333" stroke-width="1"/>
  <line x1="78" y1="52"  x2="82" y2="52"  stroke="#333" stroke-width="1"/>
  <text x="74" y="204" text-anchor="end" font-family="monospace" font-size="10" fill="#555">0.00</text>
  <text x="74" y="167" text-anchor="end" font-family="monospace" font-size="10" fill="#555">0.10</text>
  <text x="74" y="130" text-anchor="end" font-family="monospace" font-size="10" fill="#555">0.20</text>
  <text x="74" y="93"  text-anchor="end" font-family="monospace" font-size="10" fill="#555">0.30</text>
  <text x="74" y="56"  text-anchor="end" font-family="monospace" font-size="10" fill="#555">0.40</text>
  <!-- subtle gridlines -->
  <line x1="80" y1="163" x2="640" y2="163" stroke="#eee" stroke-width="1"/>
  <line x1="80" y1="126" x2="640" y2="126" stroke="#eee" stroke-width="1"/>
  <line x1="80" y1="89"  x2="640" y2="89"  stroke="#eee" stroke-width="1"/>
  <line x1="80" y1="52"  x2="640" y2="52"  stroke="#eee" stroke-width="1"/>
  <!-- Bars: H_T values scaled: 0.50 → y=20, 0.00 → y=200. scale = 360px per unit -->
  <!-- Baseline: 0.4595 → height=165.4, top=34.6 -->
  <rect x="105" y="35" width="70" height="165" fill="#ef5350"/>
  <text x="140" y="30" text-anchor="middle" font-family="monospace" font-size="10" fill="#c62828">0.460</text>
  <!-- RAG: 0.3679 → height=132.4, top=67.6 -->
  <rect x="210" y="68" width="70" height="132" fill="#ff8f00"/>
  <text x="245" y="63" text-anchor="middle" font-family="monospace" font-size="10" fill="#e65100">0.368</text>
  <!-- Governor: 0.2200 → height=79.2, top=120.8 -->
  <rect x="315" y="121" width="70" height="79" fill="#43a047"/>
  <text x="350" y="116" text-anchor="middle" font-family="monospace" font-size="10" fill="#1b5e20" font-weight="700">0.220 ★</text>
  <!-- Ablation A: 0.2422 → height=87.2, top=112.8 -->
  <rect x="420" y="113" width="70" height="87" fill="#7e57c2"/>
  <text x="455" y="108" text-anchor="middle" font-family="monospace" font-size="10" fill="#4527a0">0.242</text>
  <!-- Ablation B: 0.4369 → height=157.3, top=42.7 -->
  <rect x="525" y="43" width="70" height="157" fill="#90a4ae"/>
  <text x="560" y="38" text-anchor="middle" font-family="monospace" font-size="10" fill="#455a64">0.437</text>
  <!-- X axis labels -->
  <text x="140" y="218" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#333">Baseline</text>
  <text x="245" y="218" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#333">RAG</text>
  <text x="350" y="218" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#1b5e20" font-weight="700">Governor</text>
  <text x="455" y="218" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#333">Ablation A</text>
  <text x="560" y="218" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#333">Ablation B</text>
  <!-- Y axis label -->
  <text x="32" y="115" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#555" transform="rotate(-90,32,115)">Archived H_T</text>
  <!-- Caption -->
  <text x="340" y="244" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#555">Illustration G: Archived H_T; neither generation improvement nor physical safety is verified.</text>
</svg>

**Table 3:** Archived pairwise statistics (reported Mann–Whitney U and Holm correction). Original p-values and effect sizes are unverified; diagnostic comparisons are not tests of independent physical safety.

| Comparison | Endpoint | U | Z | Holm p | r | Sig. |
|---|---|---|---|---|---|---|
| Gov vs. Baseline | H_T | 8100 | +11.587 | < 10⁻¹⁰ | 0.864 | *** |
| Gov vs. RAG | H_T | 8084 | +11.541 | < 10⁻¹⁰ | 0.860 | *** |
| Abl-A vs. Gov | H_T | 2663 | −3.970 | < 0.001 | 0.296 | *** |
| Abl-B vs. Gov | H_T | 0 | −11.587 | < 10⁻¹⁰ | 0.864 | *** |
| Gov vs. Baseline | Violations | 4473 | +1.210 | 0.226 | 0.090 | ns |

*Archived notation: \*\*\* p < 0.001 · ns = not significant. These labels reproduce the historical report, not independently verified significance.*

**Table 4:** Archived per-step overhead figures. Original timing records and a deployment-level real-time budget have not been independently verified.

| Component | Latency (ms) | CPU (%) | Mem (MB) | Energy (mJ) |
|---|---|---|---|---|
| Claim verification ΔI | 1.2 ± 0.2 | 2.1 | 4.2 | ~0.8 |
| Constraint checks ΔR | 0.9 ± 0.1 | 1.8 | 3.1 | ~0.6 |
| Contradiction score ΔC | 2.4 ± 0.4 | 3.5 | 8.6 | ~1.4 |
| Governor update (ϕ, χ, ΔΦ) | 0.3 ± 0.1 | 0.4 | 1.1 | ~0.2 |
| Logging / audit trail | 0.6 ± 0.1 | 0.9 | 2.3 | ~0.3 |
| **TOTAL** | **5.4 ± 0.7** | **8.7** | **19.3** | **~3.3** |

> The archive reports **~5.4 ms/step** and compares it with 50–200 Hz control periods (5–20 ms). This does not establish a validated latency bound, deadline compliance, or safe real-time integration.

### 8.2 · Historical MuJoCo Proxy Report (Companion PoC)

**Historical report.** The following numerical comparison and intervals are retained as recorded, with unauthenticated original run provenance. They do not establish an independently measured physical-safety benefit.

The archive describes a **dm_control quadruped domain** (MuJoCo backend) implementation using the "escape" task. Contact simulation, actuation, and proprioceptive observations make it a physics-based proxy; they do not establish independent physical-safety validation or transfer to broader locomotion tasks.

**Reported protocol.** The manuscript states that parameters (wR, wI, wC, α, β, γ, δ, τ₁, τ₂, ϕ₀) were not retuned, identifies primary seeds 0–9 and additional seeds 40–49, and specifies B = 2000 bootstrap resamples. Complete original records do not authenticate those populations for every numerical summary, establish interval validity, or confirm robustness.

**Reported agent model.** The historical primary comparison uses a deterministic mock proposal generator. The supervisor leaves model weights unchanged and gates downstream actions. This design property does not establish identical measured H_T across protocols, an unchanged executed-action distribution, or independent physical-safety improvement.

**Historical quantitative report.** The manuscript records H_T = 0.65 in both conditions and diagnostic counts of 11.4 and 4.7 per episode (approximately 59% contrast). It also reports non-overlapping 95% bootstrap intervals and EXECUTE fractions of 100% and 30%. Their original run/interval provenance is unresolved. The reviewed mode-conditioned counting rule is not an independent executed-action endpoint; neither the counter contrast nor non-overlapping archived intervals establishes a statistically verified physical-safety benefit or a validated compute bound.

**Table 5:** Historical manuscript summary — reported mean and 95% bootstrap CI, claimed seeds 0–9. Values and intervals remain unverified and are not independent physical-safety measurements.

| Condition | Archived H_T | Archived 95% CI (unverified) | Archived diagnostic counter | Archived 95% CI (unverified) | Recorded EXECUTE (%) |
|---|---|---|---|---|---|
| Baseline LLM | 0.65 | [0.47, 0.83] | 11.4 | [9.7, 13.1] | 100 |
| **LLM+Governor** | **0.65** | **[0.47, 0.83]** | **4.7** | **[3.0, 6.5]** | **30** |

<!-- ILLUSTRATION H: MuJoCo Violations comparison -->
<svg width="100%" viewBox="0 0 680 270" xmlns="http://www.w3.org/2000/svg">
  <title>Historical illustration with unverified original provenance</title>
  <desc>Archived values or assumed auxiliary-model behavior only. This illustration does not establish independently measured physical-safety improvement, component necessity, robustness, or generator equivalence.</desc>
  <defs>
    <marker id="arr8" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>
  <!-- LEFT PANEL: Violations comparison -->
  <rect x="30" y="15" width="290" height="220" rx="10" fill="#fafafa" stroke="#e0e0e0" stroke-width="1"/>
  <text x="175" y="38" text-anchor="middle" font-family="sans-serif" font-size="12" font-weight="700" fill="#333">Archived diagnostic counter</text>
  <line x1="70" y1="55" x2="70" y2="195" stroke="#555" stroke-width="1.2"/>
  <line x1="70" y1="195" x2="300" y2="195" stroke="#555" stroke-width="1.2"/>
  <text x="64" y="59"  text-anchor="end" font-family="monospace" font-size="9" fill="#555">12</text>
  <text x="64" y="105" text-anchor="end" font-family="monospace" font-size="9" fill="#555">8</text>
  <text x="64" y="150" text-anchor="end" font-family="monospace" font-size="9" fill="#555">4</text>
  <text x="64" y="196" text-anchor="end" font-family="monospace" font-size="9" fill="#555">0</text>
  <line x1="70" y1="59"  x2="300" y2="59"  stroke="#eee" stroke-width="1"/>
  <line x1="70" y1="105" x2="300" y2="105" stroke="#eee" stroke-width="1"/>
  <line x1="70" y1="150" x2="300" y2="150" stroke="#eee" stroke-width="1"/>
  <!-- Baseline bar: 11.4 -->
  <rect x="105" y="62" width="65" height="133" fill="#ef5350"/>
  <line x1="137" y1="47" x2="137" y2="80" stroke="#c62828" stroke-width="1.5"/>
  <line x1="127" y1="47" x2="147" y2="47" stroke="#c62828" stroke-width="1.2"/>
  <line x1="127" y1="80" x2="147" y2="80" stroke="#c62828" stroke-width="1.2"/>
  <text x="137" y="220" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#333">Baseline</text>
  <text x="137" y="56"  text-anchor="middle" font-family="monospace" font-size="10" fill="#c62828">11.4 [9.7–13.1]</text>
  <!-- Governor bar: 4.7 -->
  <rect x="210" y="140" width="65" height="55" fill="#43a047"/>
  <line x1="242" y1="125" x2="242" y2="160" stroke="#1b5e20" stroke-width="1.5"/>
  <line x1="232" y1="125" x2="252" y2="125" stroke="#1b5e20" stroke-width="1.2"/>
  <line x1="232" y1="160" x2="252" y2="160" stroke="#1b5e20" stroke-width="1.2"/>
  <text x="242" y="220" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#1b5e20" font-weight="700">Governor</text>
  <text x="242" y="134" text-anchor="middle" font-family="monospace" font-size="10" fill="#1b5e20" font-weight="700">4.7 [3.0–6.5]</text>
  <text x="175" y="240" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#e65100" font-weight="700">≈ 59% counter contrast; unverified CIs</text>
  <!-- RIGHT PANEL: EXECUTE mode comparison -->
  <rect x="350" y="15" width="300" height="220" rx="10" fill="#fafafa" stroke="#e0e0e0" stroke-width="1"/>
  <text x="500" y="38" text-anchor="middle" font-family="sans-serif" font-size="12" font-weight="700" fill="#333">EXECUTE Mode Fraction</text>
  <text x="370" y="68" font-family="sans-serif" font-size="11" fill="#333">Governor (STG active):</text>
  <rect x="370" y="78" width="84" height="30" fill="#43a047" rx="3"/>
  <text x="412" y="98" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#fff" font-weight="700">EXECUTE 30%</text>
  <rect x="454" y="78" width="196" height="30" fill="#b0bec5" rx="3"/>
  <text x="552" y="98" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#37474f" font-weight="700">other modes 70%</text>
  <text x="370" y="140" font-family="sans-serif" font-size="11" fill="#333">Baseline mode (shared checker):</text>
  <rect x="370" y="150" width="280" height="30" fill="#ef9a9a" rx="3"/>
  <text x="510" y="170" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#b71c1c" font-weight="700">EXECUTE 100% (archived)</text>
  <text x="500" y="205" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#555">Recorded mode difference only</text>
  <text x="500" y="220" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#555">physical safety unverified (Table 5)</text>
  <text x="340" y="258" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#555">Illustration H: Archived counters, intervals and modes; no verified physical-safety effect.</text>
</svg>

### Figure 3 — Descriptive archived diagnostics

![Figure 3: descriptive archived diagnostic summaries](docs/figure3/figure3_archived_diagnostics.png)

**Figure 3. Archived diagnostic summaries.** The plotted values are descriptive aggregates with unauthenticated original run provenance and sample counts; they do not establish independently measured physical-safety improvement. Panel B error bars are archived standard deviations. *real_llm is a historical source-file label; original model/run records are unauthenticated.* See the [proposed manuscript caption and source mapping](docs/figure3/README.md).

<details>
<summary>Historical four-panel illustration and claims (retained for provenance; superseded by Figure 3)</summary>



<!-- FIGURE 2 (paper): Robustness Summary — 4-panel SVG recreation -->
<svg width="100%" viewBox="0 0 680 590" xmlns="http://www.w3.org/2000/svg">
  <title>Historical illustration with unverified original provenance</title>
  <desc>Archived values or assumed auxiliary-model behavior only. This illustration does not establish independently measured physical-safety improvement, component necessity, robustness, or generator equivalence.</desc>
  <defs>
    <marker id="arr9" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>
  <!-- Title -->
  <text x="340" y="22" text-anchor="middle" font-family="sans-serif" font-size="13" font-weight="700" fill="#333">Figure 2 — Historical diagnostics</text>

  <!-- ═══════════════ PANEL A ═══════════════ -->
  <rect x="20" y="32" width="315" height="230" rx="8" fill="#fafafa" stroke="#ddd" stroke-width="1"/>
  <text x="177" y="52" text-anchor="middle" font-family="sans-serif" font-size="11" font-weight="700" fill="#333">(A) Archived counter and H_T</text>
  <text x="177" y="66" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#555">Original runs and H_T provenance unresolved</text>
  <!-- Left y-axis: violations 0–16 -->
  <line x1="58" y1="80" x2="58" y2="220" stroke="#555" stroke-width="1"/>
  <line x1="58" y1="220" x2="280" y2="220" stroke="#555" stroke-width="1"/>
  <text x="30" y="84" font-family="sans-serif" font-size="8" fill="#4472c4" text-anchor="middle" transform="rotate(-90,30,150)">Archived counter / ep.</text>
  <!-- y ticks violations (0–16, scale = 140/16 = 8.75px per unit) -->
  <text x="54" y="222" text-anchor="end" font-family="monospace" font-size="8" fill="#555">0</text>
  <text x="54" y="187" text-anchor="end" font-family="monospace" font-size="8" fill="#555">4</text>
  <text x="54" y="152" text-anchor="end" font-family="monospace" font-size="8" fill="#555">8</text>
  <text x="54" y="117" text-anchor="end" font-family="monospace" font-size="8" fill="#555">12</text>
  <text x="54" y="83" text-anchor="end" font-family="monospace" font-size="8" fill="#555">16</text>
  <line x1="56" y1="187" x2="280" y2="187" stroke="#eee" stroke-width="0.8"/>
  <line x1="56" y1="152" x2="280" y2="152" stroke="#eee" stroke-width="0.8"/>
  <line x1="56" y1="117" x2="280" y2="117" stroke="#eee" stroke-width="0.8"/>
  <!-- violation bars: Baseline=13, Governor=4.7, RealLLM=5.7, NoisyOracle=6 -->
  <!-- scale: 220 - val*8.75 -->
  <rect x="68"  y="106" width="28" height="114" fill="#4472c4"/>
  <rect x="118" y="179" width="28" height="41"  fill="#4472c4"/>
  <rect x="168" y="170" width="28" height="50"  fill="#4472c4"/>
  <rect x="218" y="167" width="28" height="53"  fill="#4472c4"/>
  <!-- H_T orange bars (right axis, 0–1, scale=140): Baseline≈0.8, Gov≈0.77, Real≈0.82, Noisy≈0.8 -->
  <rect x="96"  y="108" width="20" height="112" fill="#ed7d31" opacity="0.85"/>
  <rect x="146" y="112" width="20" height="108" fill="#ed7d31" opacity="0.85"/>
  <rect x="196" y="105" width="20" height="115" fill="#ed7d31" opacity="0.85"/>
  <rect x="246" y="108" width="20" height="112" fill="#ed7d31" opacity="0.85"/>
  <!-- right axis label -->
  <text x="295" y="84" font-family="sans-serif" font-size="8" fill="#ed7d31" text-anchor="middle" transform="rotate(90,295,150)">Hallucination rate H_T</text>
  <!-- x labels -->
  <text x="82"  y="232" text-anchor="middle" font-family="sans-serif" font-size="8" fill="#333">Baseline</text>
  <text x="132" y="232" text-anchor="middle" font-family="sans-serif" font-size="8" fill="#333">Governor</text>
  <text x="182" y="232" text-anchor="middle" font-family="sans-serif" font-size="8" fill="#333">Real LLM</text>
  <text x="232" y="232" text-anchor="middle" font-family="sans-serif" font-size="8" fill="#333">Noisy Oracle</text>
  <!-- legend -->
  <rect x="62" y="244" width="10" height="8" fill="#4472c4"/>
  <text x="76" y="251" font-family="sans-serif" font-size="8" fill="#333">Counter</text>
  <rect x="148" y="244" width="10" height="8" fill="#ed7d31"/>
  <text x="162" y="251" font-family="sans-serif" font-size="8" fill="#333">Hallucination rate H_T</text>

  <!-- ═══════════════ PANEL B ═══════════════ -->
  <rect x="345" y="32" width="315" height="230" rx="8" fill="#fafafa" stroke="#ddd" stroke-width="1"/>
  <text x="502" y="52" text-anchor="middle" font-family="sans-serif" font-size="11" font-weight="700" fill="#333">(B) Ablation Study</text>
  <text x="502" y="66" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#555">component necessity unverified</text>
  <line x1="378" y1="80" x2="378" y2="220" stroke="#555" stroke-width="1"/>
  <line x1="378" y1="220" x2="635" y2="220" stroke="#555" stroke-width="1"/>
  <!-- dashed reference line at full governor = 4.87 -->
  <!-- scale: 220 - val*(140/10)=14px per unit, 0–10 -->
  <line x1="378" y1="152" x2="635" y2="152" stroke="#888" stroke-width="1" stroke-dasharray="5 3"/>
  <text x="637" y="155" font-family="sans-serif" font-size="7" fill="#888">ref. 4.87</text>
  <!-- y ticks 0–10 -->
  <text x="374" y="222" text-anchor="end" font-family="monospace" font-size="8" fill="#555">0</text>
  <text x="374" y="180" text-anchor="end" font-family="monospace" font-size="8" fill="#555">3</text>
  <text x="374" y="138" text-anchor="end" font-family="monospace" font-size="8" fill="#555">6</text>
  <text x="374" y="96"  text-anchor="end" font-family="monospace" font-size="8" fill="#555">9</text>
  <line x1="376" y1="180" x2="635" y2="180" stroke="#eee" stroke-width="0.8"/>
  <line x1="376" y1="138" x2="635" y2="138" stroke="#eee" stroke-width="0.8"/>
  <!-- bars: FullGov=4.87→y=152, δ=0→9→y=94, removeI→7.3→y=118, removeC→6.8→y=125 -->
  <rect x="393" y="152" width="42" height="68"  fill="#43a047"/>
  <!-- error bars -->
  <line x1="414" y1="138" x2="414" y2="166" stroke="#1b5e20" stroke-width="1.5"/>
  <line x1="408" y1="138" x2="420" y2="138" stroke="#1b5e20" stroke-width="1"/>
  <line x1="408" y1="166" x2="420" y2="166" stroke="#1b5e20" stroke-width="1"/>
  <rect x="453" y="94"  width="42" height="126" fill="#e74c3c"/>
  <line x1="474" y1="78" x2="474" y2="110" stroke="#c0392b" stroke-width="1.5"/>
  <line x1="468" y1="78" x2="480" y2="78" stroke="#c0392b" stroke-width="1"/>
  <line x1="468" y1="110" x2="480" y2="110" stroke="#c0392b" stroke-width="1"/>
  <rect x="513" y="118" width="42" height="102" fill="#7e57c2"/>
  <line x1="534" y1="102" x2="534" y2="134" stroke="#4527a0" stroke-width="1.5"/>
  <line x1="528" y1="102" x2="540" y2="102" stroke="#4527a0" stroke-width="1"/>
  <line x1="528" y1="134" x2="540" y2="134" stroke="#4527a0" stroke-width="1"/>
  <rect x="573" y="125" width="42" height="95"  fill="#d4ac0d"/>
  <line x1="594" y1="109" x2="594" y2="141" stroke="#9a7d0a" stroke-width="1.5"/>
  <line x1="588" y1="109" x2="600" y2="109" stroke="#9a7d0a" stroke-width="1"/>
  <line x1="588" y1="141" x2="600" y2="141" stroke="#9a7d0a" stroke-width="1"/>
  <text x="414"  y="232" text-anchor="middle" font-family="sans-serif" font-size="8" fill="#333">Full Gov</text>
  <text x="474"  y="232" text-anchor="middle" font-family="sans-serif" font-size="8" fill="#333">δ=0</text>
  <text x="534"  y="232" text-anchor="middle" font-family="sans-serif" font-size="8" fill="#333">remove_I</text>
  <text x="594"  y="232" text-anchor="middle" font-family="sans-serif" font-size="8" fill="#333">remove_C</text>
  <text x="490" y="248" text-anchor="middle" font-family="sans-serif" font-size="8" fill="#555">Archived counter differences; significance unverified</text>

  <!-- ═══════════════ PANEL C ═══════════════ -->
  <rect x="20" y="290" width="315" height="240" rx="8" fill="#fafafa" stroke="#ddd" stroke-width="1"/>
  <text x="177" y="310" text-anchor="middle" font-family="sans-serif" font-size="11" font-weight="700" fill="#333">(C) Archived weight-factor sweep</text>
  <text x="177" y="324" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#555">±30% variation; robustness unverified</text>
  <line x1="58" y1="338" x2="58" y2="488" stroke="#555" stroke-width="1"/>
  <line x1="58" y1="488" x2="300" y2="488" stroke="#555" stroke-width="1"/>
  <!-- left axis: success/violation 0–40 -->
  <text x="374" y="338" font-family="sans-serif" font-size="7" fill="#888" text-anchor="middle">0.945</text>
  <text x="54" y="342" text-anchor="end" font-family="monospace" font-size="8" fill="#555">40</text>
  <text x="54" y="380" text-anchor="end" font-family="monospace" font-size="8" fill="#555">30</text>
  <text x="54" y="415" text-anchor="end" font-family="monospace" font-size="8" fill="#555">20</text>
  <text x="54" y="450" text-anchor="end" font-family="monospace" font-size="8" fill="#555">10</text>
  <text x="54" y="490" text-anchor="end" font-family="monospace" font-size="8" fill="#555">0</text>
  <!-- x axis: weight factors 0.7–1.3 -->
  <text x="68"  y="500" text-anchor="middle" font-family="monospace" font-size="8" fill="#555">0.7</text>
  <text x="108" y="500" text-anchor="middle" font-family="monospace" font-size="8" fill="#555">0.8</text>
  <text x="148" y="500" text-anchor="middle" font-family="monospace" font-size="8" fill="#555">0.9</text>
  <text x="188" y="500" text-anchor="middle" font-family="monospace" font-size="8" fill="#555">1.0</text>
  <text x="228" y="500" text-anchor="middle" font-family="monospace" font-size="8" fill="#555">1.1</text>
  <text x="268" y="500" text-anchor="middle" font-family="monospace" font-size="8" fill="#555">1.2</text>
  <text x="300" y="500" text-anchor="middle" font-family="monospace" font-size="8" fill="#555">1.3</text>
  <text x="177" y="512" text-anchor="middle" font-family="sans-serif" font-size="8" fill="#555">Weight Factor</text>
  <!-- gridlines -->
  <line x1="56" y1="380" x2="300" y2="380" stroke="#eee" stroke-width="0.8"/>
  <line x1="56" y1="415" x2="300" y2="415" stroke="#eee" stroke-width="0.8"/>
  <line x1="56" y1="450" x2="300" y2="450" stroke="#eee" stroke-width="0.8"/>
  <!-- nominal line -->
  <line x1="188" y1="338" x2="188" y2="490" stroke="#bbb" stroke-width="1" stroke-dasharray="4 3"/>
  <!-- success rate line (blue dots) — nearly flat ~40% -->
  <polyline points="68,343 108,340 148,340 188,340 228,340 268,343 300,343" fill="none" stroke="#4472c4" stroke-width="1.5"/>
  <circle cx="68"  cy="343" r="3" fill="#4472c4"/>
  <circle cx="108" cy="340" r="3" fill="#4472c4"/>
  <circle cx="148" cy="340" r="3" fill="#4472c4"/>
  <circle cx="188" cy="340" r="3" fill="#4472c4"/>
  <circle cx="228" cy="340" r="3" fill="#4472c4"/>
  <circle cx="268" cy="343" r="3" fill="#4472c4"/>
  <circle cx="300" cy="343" r="3" fill="#4472c4"/>
  <!-- violation rate line (red squares) — flat ~5 -->
  <polyline points="68,483 108,483 148,483 188,483 228,483 268,483 300,483" fill="none" stroke="#e74c3c" stroke-width="1.5"/>
  <rect x="65"  y="480" width="6" height="6" fill="#e74c3c"/>
  <rect x="105" y="480" width="6" height="6" fill="#e74c3c"/>
  <rect x="145" y="480" width="6" height="6" fill="#e74c3c"/>
  <rect x="185" y="480" width="6" height="6" fill="#e74c3c"/>
  <rect x="225" y="480" width="6" height="6" fill="#e74c3c"/>
  <rect x="265" y="480" width="6" height="6" fill="#e74c3c"/>
  <rect x="297" y="480" width="6" height="6" fill="#e74c3c"/>
  <!-- ΔΦ stability line (green triangles) — peaks at 1.0 -->
  <polyline points="68,465 108,438 148,400 188,338 228,368 268,420 300,468" fill="none" stroke="#27ae60" stroke-width="1.5"/>
  <polygon points="65,468 68,462 71,468" fill="#27ae60"/>
  <polygon points="105,441 108,435 111,441" fill="#27ae60"/>
  <polygon points="145,403 148,397 151,403" fill="#27ae60"/>
  <polygon points="185,341 188,335 191,341" fill="#27ae60"/>
  <polygon points="225,371 228,365 231,371" fill="#27ae60"/>
  <polygon points="265,423 268,417 271,423" fill="#27ae60"/>
  <polygon points="297,471 300,465 303,471" fill="#27ae60"/>
  <!-- legend C -->
  <rect x="62" y="516" width="8" height="8" fill="#4472c4" rx="4"/>
  <text x="74" y="523" font-family="sans-serif" font-size="8" fill="#333">Success rate (%)</text>
  <rect x="140" y="516" width="8" height="8" fill="#e74c3c"/>
  <text x="152" y="523" font-family="sans-serif" font-size="8" fill="#333">Counter</text>
  <polygon points="198,520 202,514 206,520" fill="#27ae60"/>
  <text x="210" y="523" font-family="sans-serif" font-size="8" fill="#333">ΔΦ index</text>

  <!-- ═══════════════ PANEL D ═══════════════ -->
  <rect x="345" y="290" width="315" height="240" rx="8" fill="#fafafa" stroke="#ddd" stroke-width="1"/>
  <text x="502" y="310" text-anchor="middle" font-family="sans-serif" font-size="11" font-weight="700" fill="#333">(D) Historical generator labels</text>
  <text x="502" y="324" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#555">Original model/run identity unverified</text>
  <line x1="378" y1="338" x2="378" y2="460" stroke="#555" stroke-width="1"/>
  <line x1="378" y1="460" x2="635" y2="460" stroke="#555" stroke-width="1"/>
  <!-- left axis: success % 0–60 -->
  <text x="374" y="342" text-anchor="end" font-family="monospace" font-size="8" fill="#555">60</text>
  <text x="374" y="380" text-anchor="end" font-family="monospace" font-size="8" fill="#555">40</text>
  <text x="374" y="420" text-anchor="end" font-family="monospace" font-size="8" fill="#555">20</text>
  <text x="374" y="462" text-anchor="end" font-family="monospace" font-size="8" fill="#555">0</text>
  <line x1="376" y1="380" x2="635" y2="380" stroke="#eee" stroke-width="0.8"/>
  <line x1="376" y1="420" x2="635" y2="420" stroke="#eee" stroke-width="0.8"/>
  <!-- Mock LLM bars: success~41%, violation~5 -->
  <!-- success bar (blue): 41/60 * 122 = 83px, top=460-83=377 -->
  <rect x="400" y="377" width="45" height="83" fill="#4472c4"/>
  <!-- violation bar (orange): 5/60*122=10px approx, right axis ~0-8 -->
  <rect x="450" y="450" width="45" height="10" fill="#ed7d31"/>
  <!-- Real LLM bars: success~39%, violation~5.2 -->
  <rect x="530" y="380" width="45" height="80" fill="#4472c4"/>
  <rect x="580" y="449" width="45" height="11" fill="#ed7d31"/>
  <!-- right axis label -->
  <text x="637" y="342" font-family="sans-serif" font-size="7" fill="#ed7d31" text-anchor="start">8</text>
  <text x="637" y="462" font-family="sans-serif" font-size="7" fill="#ed7d31" text-anchor="start">0</text>
  <text x="422" y="472" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#333">Mock LLM</text>
  <text x="552" y="472" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#333">Real LLM</text>
  <!-- legend D -->
  <rect x="390" y="484" width="10" height="8" fill="#4472c4"/>
  <text x="404" y="491" font-family="sans-serif" font-size="8" fill="#333">Success rate (%)</text>
  <rect x="510" y="484" width="10" height="8" fill="#ed7d31"/>
  <text x="524" y="491" font-family="sans-serif" font-size="8" fill="#333">Counter</text>
  <text x="502" y="508" text-anchor="middle" font-family="sans-serif" font-size="8" fill="#555">Archived aggregates do not establish generator equivalence</text>

  <!-- Overall figure caption -->
  <text x="340" y="552" text-anchor="middle" font-family="sans-serif" font-size="10" font-weight="700" fill="#333">Figure 2: Legacy illustration; Figure 3 records the source-matched values.</text>
  <text x="340" y="572" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#555">Physical safety, necessity, robustness and generator equivalence remain unverified.</text>
</svg>

</details>

Inspectable implementation: **https://github.com/dfeen87/stg-embodied-poc**. Available code and fixed configuration support prospective execution, not authentication of every historical result.

---

## 9 · Limitations

**Conservative supervision.** Thresholds select software modes. They do not establish reduced physical hazards, semantic correctness, or task validity. SAFE remains a mode name, not a certified safety state.

**Threshold sensitivity.** Behavior depends on τ₁ and τ₂. Archived ablation and sensitivity aggregates do not establish tuning robustness, component necessity, an optimal operating range, or generalization; principled threshold selection remains future work.

**Simulator-to-hardware transfer.** The MuJoCo implementation is a proxy, and the archived counter is not an independently validated safety measure. Hardware deployment would require separately validated sensing, state estimation, monitors, fallback controllers, and plant/environment assumptions.

**LLM abstraction.** The primary code path uses a deterministic mock generator. Archived external-LLM labels do not authenticate the original model or run records and do not establish equivalent performance, robustness, or safety under stochastic generation.

---

## 10 · Conclusion

We present a **deterministic Spiral-Time Governor** as an inspectable supervisory design with an instability functional ΔΦ, fixed thresholds, and EXECUTE / VERIFY / SAFE mode selection. The archive does not establish suppression of upstream hallucination generation or an independently verified reduction in physically unsafe execution.

The design leaves model weights unchanged and provides explicit mode-selection logic. Normalization bounds supervisory variables directly; the archived Lyapunov argument assumes an auxiliary linear model and is not a proved hybrid-governor, plant-stability, or robot-safety certificate.

Historical synthetic and MuJoCo summaries document distinct diagnostic reports with unresolved original provenance. They do not establish a 59% physical-safety improvement, deployment readiness, robustness, or component necessity. Future controlled evaluation must use matched monitoring access and independent state/action outcomes, with separately identified data and protocols.

<!-- FIGURE: Summary Visual -->
<svg width="100%" viewBox="0 0 680 130" xmlns="http://www.w3.org/2000/svg">
  <title>Historical illustration with unverified original provenance</title>
  <desc>Archived values or assumed auxiliary-model behavior only. This illustration does not establish independently measured physical-safety improvement, component necessity, robustness, or generator equivalence.</desc>
  <!-- Three pillars of the contribution -->
  <rect x="40"  y="20" width="175" height="88" rx="10" fill="#e3f2fd" stroke="#1565c0" stroke-width="1"/>
  <text x="127" y="46" text-anchor="middle" font-family="sans-serif" font-size="12" font-weight="700" fill="#0d47a1">Auxiliary model</text>
  <text x="127" y="64" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#1565c0">Assumed linear model</text>
  <text x="127" y="80" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#1565c0">No governor / plant</text>
  <text x="127" y="96" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#1565c0">safety certificate</text>
  <rect x="252" y="20" width="175" height="88" rx="10" fill="#fff8e1" stroke="#f9a825" stroke-width="1"/>
  <text x="340" y="46" text-anchor="middle" font-family="sans-serif" font-size="12" font-weight="700" fill="#e65100">Deterministic Gating</text>
  <text x="340" y="64" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#f57f17">ΔΦ functional with</text>
  <text x="340" y="80" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#f57f17">auditable thresholds</text>
  <text x="340" y="96" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#f57f17">model-agnostic</text>
  <rect x="464" y="20" width="175" height="88" rx="10" fill="#e8f5e9" stroke="#2e7d32" stroke-width="1"/>
  <text x="551" y="46" text-anchor="middle" font-family="sans-serif" font-size="12" font-weight="700" fill="#1b5e20">Archived diagnostics</text>
  <text x="551" y="64" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#2e7d32">Synthetic testbed +</text>
  <text x="551" y="80" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#2e7d32">MuJoCo PoC</text>
  <text x="551" y="96" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#2e7d32">59% counter; unverified</text>
  <text x="340" y="122" text-anchor="middle" font-family="sans-serif" font-size="11" fill="#555">Illustration I: Design and archival diagnostics; physical-safety benefit unverified.</text>
</svg>

---

## Acknowledgements

Repository scaffolding and documentation were developed with assistance from Claude (Sonnet 4.6) by Anthropic and Google (Jules Pro). All scientific content, mathematical formulations, and experimental design originate from the authors.

---

## Funding

This research received no external funding. The work was conducted independently by the authors.

---

## Data Availability

Implementation code, configuration files, seed lists, and aggregate analysis artifacts are available at: **https://github.com/dfeen87/stg-embodied-poc**. Complete original seed-level observations, state/action trajectories, and external-model metadata needed to authenticate the historical figures and intervals are not available in the audited snapshot. Code availability does not independently verify the reported safety effects.

---

## AI Usage Statement

The historical manuscript described LLMs as research objects and AI assistance with language and formatting; repository acknowledgements also describe AI-assisted scaffolding and documentation. Neither AI assistance nor author responsibility independently verifies the historical mathematical or experimental safety claims. The authors remain responsible for resolving the evidence limitations before any final scientific submission.

---

## Conflict of Interest

The authors declare no conflict of interest.

---

## Author Contributions

**M.K.** is recorded in the historical authorship statement as originator of the Spiral-Time concept and primary manuscript author, with responsibility for theory, methodology, and system design. These recorded roles do not independently validate the archived stability argument or physical-safety interpretation.

**D.M.F.** is recorded in the historical authorship statement as responsible for implementation, experimental execution, data generation, statistical evaluation, and technical feedback. These recorded roles do not authenticate missing original run records or establish independently verified physical-safety findings.

*The archived manuscript records prior author approval. It does not record approval of this clarification or resolve the outstanding verification limits.*

---

## References

[1] D. Driess, F. Xia, M. T. Knoop, et al., "PaLM-E: An Embodied Multimodal Language Model," arXiv:2303.03378, 2023. doi:10.48550/arXiv.2303.03378

[2] P. Xu, "Embodied AI: Bridging Simulation and Reality in Robotics," Proceedings of the 4th International Symposium on Robotics, Artificial Intelligence and Information Engineering (RAIIE), 2025. doi:10.1109/RAIIE65740.2025.11140070

[3] X. Zhou et al., "Large Language Models for Robotics: Opportunities and Challenges," arXiv:2308.14455, 2023.

[4] S. Basu, M. H. Kim, S. Tatlidil, T. Williams, S. Sloman, and R. I. Bahar, "Augmenting large language models with psychologically grounded models of causal reasoning for planning under uncertainty," *Frontiers in Artificial Intelligence*, vol. 8, Art. 1730614, Jan. 2026. doi: 10.3389/frai.2025.1730614.

[5] A. D. Ames, X. Xu, J. W. Grizzle, and P. Tabuada, "Control Barrier Function Based Quadratic Programs for Safety-Critical Systems," *IEEE Transactions on Automatic Control*, vol. 62, no. 8, pp. 3861–3876, Aug. 2017. doi: 10.1109/TAC.2016.2638961.

[6] W. Huang et al., "Inner Monologue: Embodied Reasoning through Planning with Language Models," arXiv preprint arXiv:2207.05608, 2022.

[7] A. Z. Ren, B. Govil, T.-Y. Yang, K. Narasimhan, and A. Majumdar, "Robots That Ask for Help: Uncertainty-Aligned LLM Planning," arXiv preprint arXiv:2307.01928, 2023.

[8] Z. Ji et al., "Survey of Hallucination in Natural Language Generation," *ACM Computing Surveys*, vol. 55, no. 12, pp. 1–38, 2023. doi: 10.1145/3571730.

[9] L. Huang et al., "A Survey on Hallucination in Large Language Models: Principles, Taxonomy, Challenges, and Open Questions," arXiv preprint arXiv:2311.05232, 2023.

[10] N. Shinn, F. Cassano, A. Gopinath, K. Narasimhan, and S. Yao, "Reflexion: Language Agents with Iterative Design Learning," in *Advances in Neural Information Processing Systems (NeurIPS)*, vol. 36, 2023.

[11] A. D. Ames, S. Coogan, M. Egerstedt, G. Notomista, K. Sreenath, and P. Tabuada, "Control Barrier Functions: Theory and Application," in *Proc. IEEE 18th European Control Conference (ECC)*, Naples, Italy, 2019, pp. 3420–3431. doi: 10.23919/ECC.2019.8796030.

[12] R. Cheng, G. Orosz, R. M. Murray, and J. W. Burdick, "Safe Control with Learned Models: Optimality and Runtime Guarantees," *IEEE Transactions on Automatic Control*, 2023. doi: 10.1109/TAC.2023.3247173.

[13] S. Gu, J. Grudzien Kuba, Y. Chen, Y. Du, L. Yang, A. Knoll, and Y. Yang, "Safe multi-agent reinforcement learning for multi-robot control," *Artificial Intelligence*, vol. 319, 103905, 2023. doi:10.1016/j.artint.2023.103905.

[14] H. K. Khalil, *Nonlinear Systems*, 3rd ed. Upper Saddle River, NJ, USA: Prentice Hall, 2002.

[15] J.-J. E. Slotine and W. Li, *Applied Nonlinear Control*. Englewood Cliffs, NJ, USA: Prentice Hall, 1991.

[16] E. D. Sontag, "Input-to-State Stability: Basic Concepts and Results," in *Nonlinear Dynamics and Operational Control*, P. Nistri and G. Stefani, Eds. Berlin, Germany: Springer, 1989, pp. 163–220. doi:10.1007/978-3-540-79026-86.

[17] Y. Tassa, Y. Doron, A. Muldal, T. Erez, Y. Li, S. de Freitas, N. Heess, and M. Riedmiller, "dm_control: Software and Tasks for Continuous Control," arXiv:1801.00690, 2018. doi:10.48550/arXiv.1801.00690.

[18] P. Lewis, E. Perez, A. Piktus, F. Petroni, V. Karpukhin, N. Goyal, H. Küttler, M. Lewis, W. Yih, T. Rocktäschel, S. Riedel, and D. Kiela, "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks," *Advances in Neural Information Processing Systems (NeurIPS)*, 2020. doi:10.48550/arXiv.2005.11401.

[19] J. Liang, W. Huang, F. Xia, P. Florence, A. Zeng, "Code as Policies: Language Model Programs for Embodied Control," in *Proc. IEEE International Conference on Robotics and Automation (ICRA)*, 2023. doi:10.1109/ICRA48891.2023.10160591.

[20] A. Brohan, N. Brown, B. Zitkovich et al., "RT-2: Vision-Language-Action Models Transfer Web Knowledge to Robotic Control," arXiv:2307.15818, 2023. doi:10.48550/arXiv.2307.15818
