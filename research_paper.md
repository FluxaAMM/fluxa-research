# Logarithmic Number Representation for Decentralized Exchange Systems

**Srivatsav Erramilli**

_Independent Researcher_

srivatsaverramilli@gmail.com

December 2025 | Version 1.0

---

## Abstract

Decentralized exchanges face a fundamental tension between price representation precision, computational efficiency, and the unification of heterogeneous trading mechanisms. Current approaches—whether tick-based concentrated liquidity market makers (CLMMs) or fixed-point arithmetic systems—impose architectural constraints that fragment liquidity across separate spot, limit order, and derivatives venues, while struggling to achieve centralized exchange-grade numerical precision. This paper introduces a **logarithmic number representation** for on-chain price coordinates, replacing traditional linear or tick-based encodings with a continuous log-sqrt-price axis $l = \log_2 \sqrt{P}$ that serves as the sole source of truth for all price-dependent operations.

We develop a **three-layer architecture**: Layer 0 provides a continuous mathematical model where CLMM liquidity is a piecewise-constant density and limit orders are atomic measures on the same $l$-axis; Layer 1 defines a global slot lattice for efficient on-chain indexing without constraining economic prices; and Layer 2 constructs adaptive, trade-specific meshes that include all liquidity boundaries and order prices as nodes, enabling exact segment-wise integration of CLMM dynamics. This architecture naturally extends to a **spot-anchored derivative layer** where perpetual swaps, futures, and options derive all pricing from the unified spot process $P(l(t)) = 2^{2l(t)}$, eliminating basis risk and the need for funding rate mechanisms.

We establish rigorous **fixed-point specifications** with provable error bounds: the total numerical error in token deltas is bounded by $C(\frac{1}{\sigma} + \varepsilon_{\exp_2} + \varepsilon_{\log_2} + \varepsilon_{\mathrm{arith}})$, where $\sigma = 2^F$ is the log-scaling precision, and the approximation errors can be made arbitrarily small by increasing fractional bit-lengths. With $F = 48$ and Q64.64 formats, global relative price errors reach $O(10^{-14})$—comparable to IEEE 754 double precision. We prove that the spot-anchored mark price is the **unique no-arbitrage choice**, that protocol-level delta hedging achieves bounded residual risk under discrete execution, and that liquidations respect monotone price paths with non-negative reserves. Additional results include deterministic compositionality of concurrent liquidations, insurance fund solvency conditions, and leverage constraints that provably prevent unbounded liquidation cascades.

The unified semantics—where CLMM liquidity, central limit order book depth, and derivative positions all evolve on a single log-price coordinate with deterministic node execution ordering—enables precision and capital efficiency comparable to modern centralized exchange engines, while preserving the transparency and composability of decentralized systems.

---

## 1. Introduction

### 1.1 Motivation

The architecture of decentralized exchanges has evolved rapidly over the past several years. Constant-product automated market makers (AMMs) [@adams2020uniswap] gave way to concentrated liquidity market makers (CLMMs), exemplified by Uniswap v3 [@adams2021uniswap], which allow liquidity providers to allocate capital to specific price ranges rather than the entire $(0, \infty)$ interval. Concurrently, on-chain central limit order books (CLOBs) [@openbook2022] have emerged on high-throughput chains, offering price-time priority matching familiar from traditional finance. Most recently, perpetual swap protocols [@bitmex2016perpetual] have brought leveraged derivatives on-chain, complete with margin systems, funding rates, and liquidation engines.

These three paradigms—CLMMs, CLOBs, and perpetuals—currently exist as largely separate systems, each with its own price representation, execution logic, and liquidity pools. A trader wishing to execute a large order may need to route across multiple venues; a liquidity provider must choose between providing to an AMM pool, posting limit orders, or supplying collateral to a perp market. This fragmentation imposes capital inefficiency, increases smart contract complexity, and makes formal reasoning about system-wide behavior difficult.

At the heart of this fragmentation lies a representation problem. CLMMs typically encode prices via "ticks"—discrete points on a logarithmic scale chosen for computational convenience rather than mathematical elegance. CLOBs store limit order prices in various fixed-point formats. Perpetual protocols maintain separate "mark prices" derived from spot via funding rate mechanisms designed to keep perp prices anchored. Each system makes different tradeoffs between precision, gas cost, and expressiveness, and the interfaces between them are inherently lossy.

### 1.2 Problem Statement

Existing decentralized exchange designs suffer from several interrelated limitations:

**Ad-hoc discretization.** Tick-based systems (e.g., Uniswap v3's tick spacing [@adams2021uniswap]) impose a rigid grid that does not arise from any principled continuous model. The relationship between tick indices, sqrt-price values, and actual prices involves protocol-specific constants (e.g., 1.0001 as the tick base) and implicit rounding that obscure error analysis.

**Opaque fixed-point arithmetic.** On-chain implementations rely on fixed-point representations (e.g., Uniswap v3's `sqrtPriceX96` [@adams2021uniswap]) with precision choices made for historical or gas-optimization reasons. The propagation of rounding errors through CLMM formulas, CLOB matching, and derivative valuations is rarely analyzed formally.

**Fragmented liquidity and execution.** Because CLMMs, CLOBs, and perps use incompatible price representations, unifying them requires conversion layers that introduce additional error and complexity. There is no single "source of truth" price that all components reference.

**Derivative basis and funding complexity.** Traditional on-chain perpetuals define a separate "perp price" distinct from spot, necessitating funding rate mechanisms to keep them aligned. This introduces basis risk, funding arbitrage, and additional state variables that complicate both implementation and formal analysis.

**Lack of formal guarantees.** Perhaps most critically, existing systems provide no rigorous bounds on numerical error, no formal specification of execution semantics across liquidity types, and no proofs that the discrete on-chain implementation converges to any well-defined continuous model.

### 1.3 Key Idea

This paper proposes a unified framework based on a single **log-sqrt-price axis**:

$$
l = \log_2 S = \log_2 \sqrt{P} = \frac{1}{2} \log_2 P
$$

where $P$ is the spot price and $S = \sqrt{P}$ is the sqrt-price. All prices, liquidity distributions, limit orders, and derivative valuations are expressed as functions of this one-dimensional coordinate $l \in \mathbb{R}$.

We organize the system into **three layers**:

1. **Layer 0 (Continuous):** A purely mathematical model where the log-sqrt-price $l$ takes values in $\mathbb{R}$, CLMM liquidity is a piecewise-constant density $\lambda(l)$, and CLOB orders are point masses (Dirac measures) on the $l$-axis. This layer defines the "true" economic semantics independent of any discretization.

2. **Layer 1 (Global Slots):** A persistent lattice $\{l_s = l_{\mathrm{ref}} + s \cdot \Delta l : s \in \mathbb{Z}\}$ used for on-chain indexing. Liquidity provider positions and limit orders are mapped to slot indices for efficient storage and lookup, but their exact $l$-coordinates are preserved. The slot grid is an implementation convenience that does not constrain economic prices.

3. **Layer 2 (Local Mesh):** For each trade, we construct an adaptive mesh over the relevant price interval, including all LP band boundaries and limit order prices as nodes. Execution proceeds segment-by-segment with exact CLMM integration on each segment and deterministic CLOB matching at each node.

On top of this spot architecture, we define a **spot-anchored derivative layer** where perpetuals, futures, and options derive all pricing from the unified spot process $P(l(t)) = 2^{2l(t)}$. There is no separate "perp price," no basis, and no funding rate—the derivative layer is economically equivalent to holding (or shorting) the underlying, with leverage and margin handled as pure accounting.

Finally, we provide a **formal fixed-point specification** showing that the on-chain implementation converges to the continuous model with explicit error bounds that can be made arbitrarily small by increasing numerical precision.

### 1.4 Contributions

This paper makes the following contributions:

- **A continuous log-sqrt-price formulation** that places CLMM liquidity, CLOB orders, and perpetual positions on a single mathematical axis $l \in \mathbb{R}$, with CLMM dynamics expressed as explicit differential equations $dA(l), dB(l)$ in the log-coordinate.

- **A three-layer architecture** with precise semantics: Layer 0 defines the continuous economic model; Layer 1 provides global slot-based indexing for on-chain storage; Layer 2 constructs trade-specific adaptive meshes that preserve all structural points (band boundaries, order prices) and enable exact piecewise integration.

- **Unified execution semantics** specifying deterministic interaction between CLMM and CLOB liquidity: at each mesh node, all marketable CLOB volume is consumed before the price moves into the adjacent CLMM segment, ensuring no arbitrage between order types at the same price level.

- **Fixed-point and numerical error analysis** with global convergence bounds. We prove that the total error in token deltas is bounded by $C(\frac{1}{\sigma} + \varepsilon_{\exp_2} + \varepsilon_{\log_2} + \varepsilon_{\mathrm{arith}})$, where $\sigma = 2^F$ is the log-scaling precision and the $\varepsilon$ terms capture exp/log approximation and arithmetic rounding. With practical parameters ($F = 48$, Q64.64 formats), global relative price errors reach $O(10^{-14})$.

- **A spot-anchored derivative layer** in which perpetual swaps have mark price identically equal to spot price, eliminating basis and funding rates by construction. We prove this is the unique no-arbitrage choice and show that protocol-level delta hedging achieves bounded residual risk.

- **Formal guarantees for the derivative layer**, including: uniqueness of spot-anchored mark price under no-arbitrage; bounded residual hedging error under discrete execution; monotone and reserve-preserving liquidation execution; deterministic compositionality of concurrent liquidations; insurance fund solvency conditions; and leverage constraints that provably prevent unbounded liquidation cascades.

- **LP risk isolation**: we prove that liquidity providers in the spot layer do not warehouse derivative directional risk—from their perspective, derivative activity appears only as additional spot order flow, and their wealth depends solely on the spot price path and aggregate order flow.

### 1.5 Paper Roadmap

The remainder of this paper is organized as follows:

**Section 2** introduces the basic setup and notation, defining the token pair, spot price, sqrt-price, and the log-sqrt-price coordinate that will serve as our fundamental representation.

**Section 3** develops the logarithmic representation in its continuous, idealized form, explaining the choice of base-2 logarithms and the relationship between log-sqrt-price and economic price.

**Section 4** specifies the fixed-point encoding of log-sqrt-price, including scaling factors, encode/decode maps, and the Q-format conventions used throughout.

**Section 5** presents the three-layer architecture in detail: Layer 0's continuous model with CLMM liquidity density and CLOB order measure; Layer 1's global slot lattice for indexing; Layer 2's adaptive local mesh construction; and the CLMM dynamics expressed in log-coordinates with explicit finite-move formulas.

**Section 6** provides the formal fixed-point specification, including abstract specifications of $\exp_2$ and $\log_2$ routines, monotonicity requirements, error propagation analysis, and the main convergence theorem showing that on-chain execution approaches the continuous model as precision increases.

**Section 7** defines unified execution semantics for CLMM, CLOB, and perpetuals on the log-axis, including the deterministic node execution ordering rule, price path continuity, and the formal treatment of crossing CLOB orders.

**Section 8** introduces the spot-anchored derivative layer, covering perpetual swap definitions, mark-to-market equity, the elimination of funding rates, protocol-level delta hedging, LP exposure analysis, and margin/liquidation mechanics.

**Section 9** establishes formal guarantees for the derivative layer: no-arbitrage uniqueness of spot-anchored marks, residual hedging risk bounds, execution validity under hedge flow, liquidation correctness, fee allocation invariants, computational complexity, funding-rate elimination theorem, margin dynamics, insurance fund solvency, compositionality of concurrent liquidations, and dynamic stability under leverage constraints.

---

**Figure 1: Fragmented vs. Unified DEX Architecture**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CURRENT: FRAGMENTED ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐                │
│   │    CLMM     │      │    CLOB     │      │   PERPS     │                │
│   │   (Spot)    │      │   (Spot)    │      │(Derivatives)│                │
│   ├─────────────┤      ├─────────────┤      ├─────────────┤                │
│   │ Price: √P   │      │ Price: P    │      │ Mark ≠ Spot │                │
│   │ Ticks: 1bp  │      │ Discrete    │      │ Funding ≠ 0 │                │
│   │ Liquidity A │      │ Liquidity B │      │ Liquidity C │                │
│   └──────┬──────┘      └──────┬──────┘      └──────┬──────┘                │
│          │                    │                    │                       │
│          └────────────────────┼────────────────────┘                       │
│                               ▼                                            │
│                    ╔═══════════════════╗                                   │
│                    ║ PRICE CONVERSIONS ║  ◄── Arbitrage gaps               │
│                    ║ FRAGMENTED DEPTH  ║  ◄── Capital inefficiency         │
│                    ╚═══════════════════╝                                   │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                        PROPOSED: UNIFIED ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                    ╔═════════════════════════════════╗                     │
│                    ║   SINGLE LOG-SQRT-PRICE AXIS    ║                     │
│                    ║         l = log₂(√P)            ║                     │
│                    ╚═════════════════════════════════╝                     │
│                                   │                                        │
│              ┌────────────────────┼────────────────────┐                   │
│              ▼                    ▼                    ▼                   │
│      ┌─────────────┐      ┌─────────────┐      ┌─────────────┐            │
│      │    CLMM     │      │    CLOB     │      │   PERPS     │            │
│      │  λ(l) dl    │      │  μ(dl)      │      │ Mark = Spot │            │
│      │ Continuous  │      │ Atomic      │      │ Funding = 0 │            │
│      └─────────────┘      └─────────────┘      └─────────────┘            │
│              │                    │                    │                   │
│              └────────────────────┴────────────────────┘                   │
│                                   │                                        │
│                                   ▼                                        │
│                         ┌─────────────────┐                                │
│                         │ UNIFIED DEPTH   │ ◄── Single liquidity pool      │
│                         │ SHARED PRICING  │ ◄── No arbitrage gaps          │
│                         │ ATOMIC HEDGING  │ ◄── Delta-neutral perps        │
│                         └─────────────────┘                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

_Figure 1: Comparison of current fragmented DEX architecture (top) versus our proposed unified log-domain architecture (bottom). The fragmented approach requires price conversions between incompatible representations and splits liquidity across isolated venues. Our unified approach places all trading activity—CLMM, CLOB, and derivatives—on a single log-sqrt-price axis, enabling shared liquidity, atomic hedging, and elimination of basis/funding._

<!-- LaTeX/TikZ version for PDF:
\begin{figure}[h]
\centering
\begin{tikzpicture}
  % Top: Three separate boxes for CLMM, CLOB, Perps with different price representations
  % Arrow pointing down to "Price Conversions / Fragmented Depth"
  % Bottom: Single unified log-axis with all three sharing it
  % Arrow pointing down to "Unified Depth / Shared Pricing / Atomic Hedging"
\end{tikzpicture}
\caption{Fragmented vs. Unified DEX Architecture}
\label{fig:architecture-comparison}
\end{figure}
-->

---

## 2. Background and Notation

Throughout this paper, we consider a market for two tokens, which we denote $A$ and $B$. We write the spot price of $B$ in terms of $A$ as:

$$
P = P_{A,B} > 0
$$

so that $1$ unit of $B$ is worth $P$ units of $A$.

### 2.1 Square-root Price

Define the square-root price:

$$
S = \sqrt P = P^{1/2} > 0
$$

This is analogous to Uniswap v3's **$sqrtPriceX64 / sqrtPriceX96$** [@adams2021uniswap], but we are not committing to any particular scaling yet.

The standard CLMM formulas are usually written in terms of $S$, not $P$. We are going to change the **representation** of $S$.

### 2.2 Notation Summary

For reference, we summarize the principal symbols used throughout this paper:

| Symbol            | Meaning                                                         | Section     |
| ----------------- | --------------------------------------------------------------- | ----------- |
| $P$               | Spot price of token $B$ in terms of token $A$                   | 2           |
| $S = \sqrt{P}$    | Square-root price                                               | 2.1         |
| $l = \log_2 S$    | Log-Sqrt-Price (continuous)                                     | 3.1         |
| $L$               | Fixed-point integer encoding of $l$                             | 4.1         |
| $\sigma = 2^F$    | Scaling factor for fixed-point encoding ($F$ = fractional bits) | 4.1         |
| $\lambda(l)$      | CLMM liquidity density function at log-price $l$                | 5.1.2       |
| $L_i$             | Liquidity amount for CLMM position $i$                          | 5.1.2       |
| $q_j$             | Quantity for CLOB order $j$                                     | 5.1.3       |
| $q_m$, $q_i$      | Signed perp position size for account $m$ or $i$                | 7.4.1, 9.12 |
| $M_i$             | Margin balance for account $i$                                  | 9.12.1      |
| $E_i$             | Equity (net account value) for account $i$                      | 9.12.1      |
| $\ell_i$          | Effective leverage for account $i$                              | 9.15.1      |
| $\mathfrak{L}(l)$ | Aggregate leverage measure                                      | 9.15.1      |
| $Q_S$, $Q_P$      | Fractional bits in Q-format for $S$, $P$ (fixed-point notation) | 6.2         |

**Notation conventions:**

- $L$ (no subscript) is the fixed-point integer encoding of log-sqrt-price $l$.
- $L_i$ (with subscript) denotes CLMM liquidity amounts per position; $q_i$ denotes perp position sizes.
- $\lambda(l)$ is the aggregate liquidity density (sum of per-position $L_i$); $\ell_i(l)$ is per-account leverage; $\mathfrak{L}(l)$ is aggregate leverage.
- In Section 6, $Q_S$, $Q_P$, $Q_Q$ follow standard Q-format notation for fixed-point fractional bits, unrelated to liquidity or positions.

## 3. Logarithmic Representation (Continuous, Idealized)

Instead of storing $S$ directly, we store its **logarithm**.

We choose **$base 2$** for implementation convenience and to match binary integer arithmetic.

### 3.1 Log-Sqrt-Price (Continuous)

Define the **log-sqrt-price coordinate:**

$$
l = \log_2 S = \log_2 \sqrt P
$$

Since $S > 0$, we have $l \in \mathbb{R}$.

Equivalently, in terms of price $P$:

$$
l = \frac{1}{2} \log_2 P
$$

and conversely:

$$
S = 2^l, \quad P = S^2 = 2^{2l}
$$

So, the "true" economic price is recovered by:

$$
P(l) = 2^{2l}
$$

This is the **continuous** log-domain representation of the square-root price.

**Figure 2: Log-Sqrt-Price Coordinate Transformation**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PRICE COORDINATE TRANSFORMATION                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STEP 1: Spot Price → Sqrt-Price        STEP 2: Sqrt-Price → Log-Sqrt-Price│
│                                                                             │
│       P ──────────────► S                    S ──────────────► l            │
│           S = √P                                 l = log₂(S)                │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  EXAMPLE: Price doubling in each coordinate system                          │
│                                                                             │
│  Linear Price (P):     $100 ────────► $200 ────────► $400                  │
│                              ×2            ×2                               │
│                                                                             │
│  Sqrt-Price (S):       10 ─────────► 14.14 ────────► 20                    │
│                             ×√2           ×√2                               │
│                                                                             │
│  Log-Sqrt-Price (l):   3.32 ───────► 3.82 ─────────► 4.32                  │
│                             +0.5          +0.5                              │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  KEY INSIGHT: Multiplicative changes become ADDITIVE in log-space          │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │     P-axis (linear):   |----×4----|----×4----|----×4----|          │   │
│  │                        $25       $100      $400     $1600          │   │
│  │                                                                     │   │
│  │     l-axis (log):      |----+1----|----+1----|----+1----|          │   │
│  │                       2.32      3.32      4.32      5.32           │   │
│  │                                                                     │   │
│  │     Equal intervals in l ⟺ Equal percentage changes in P           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  FORMULAS:                                                                  │
│    Forward:   l = ½ log₂(P) = log₂(√P)                                     │
│    Inverse:   P = 2^(2l)    = (2^l)²                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

_Figure 2: The log-sqrt-price transformation converts multiplicative price changes into additive movements on the l-axis. A doubling of price P corresponds to adding 0.5 to the log-sqrt-price l. This property makes percentage-based operations (like fee tiers, slippage tolerances, and tick spacing) uniform across all price levels._

<!-- LaTeX/TikZ version for PDF:
\begin{figure}[h]
\centering
\begin{tikzpicture}
  \draw[->] (0,0) -- (10,0) node[right] {$l$};
  \draw[->] (0,0) -- (0,5) node[above] {$P$};
  \draw[domain=0:4, smooth, variable=\l, blue, thick]
    plot ({\l*2.5}, {0.5*exp(1.386*\l)});
  \node at (5,-0.5) {$P = 2^{2l}$};
\end{tikzpicture}
\caption{Log-Sqrt-Price Coordinate Transformation}
\label{fig:log-sqrt-price}
\end{figure}
-->

## 4. Fixed-Point Encoding of Log-Sqrt-Price

On-chain, we don't store real numbers, only integers. So we encode $l$ as a fixed-point integer.

### 4.1 Scaling Factor

Pick a positive scaling factor $\sigma \in \mathbb{R_{>0}}$. In practice, $\sigma$ will be a power of $2$ to facilitate bit-shifting.

- $\sigma = 2^F$ for some integer $F > 0$ (number of fractional bits) (e.g. $F = 32$, $F = 48$, $F = 64$ or $F = 96$)

We define the **fixed-point representation** of $l$ (log-sqrt-price):

$$
L \in \mathbb{Z}
$$

with the interpretation:

$$
l = \frac{L}{\sigma}
$$

So:

- $L$ is the **on-chain integer representation** of the log-sqrt-price
- $l$ is the **ideal real-valued log-sqrt-price**
- The mapping is determined by the scaling factor $\sigma$

### 4.2 Encode and Decode Maps

Formally, define:

- **Encode (real $\to$ integer)**:

$$
\text{encodeLogSqrtPrice}(S) = L = \lfloor \sigma \cdot \log_2 S \rceil
$$

where $\lfloor . \rceil$ denotes some rounding rule (e.g. round to nearest, round down, round up)

- **Decode (integer $\to$ approximate real sqrt price)**:

$$
\text{decodeLogSqrtPrice}(L) = \hat{S} = 2^{L / \sigma}
$$

The approximate price decoded from $L$ is:

$$
\hat{P} = \hat{S}^2 = 2^{2L / \sigma}
$$

In exact math, you can treat:

$$
l = \frac{L}{\sigma} \implies P = 2^{2l} = 2^{2L / \sigma}
$$

In implementation, computing $2^{L / \sigma}$ exactly may not be possible, so we use approximations efficiently.

**Figure 3: Fixed-Point Encoding Pipeline**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                     FIXED-POINT ENCODING / DECODING                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ENCODING (Continuous → Integer):                                           │
│                                                                             │
│  ┌─────────┐    log₂    ┌─────────┐    ×σ     ┌─────────┐   round   ┌─────┐│
│  │ Real S  │ ─────────► │ Real l  │ ────────► │ Real L' │ ────────► │  L  ││
│  │ (√price)│            │ (cont.) │           │ (scaled)│           │(int)││
│  └─────────┘            └─────────┘           └─────────┘           └─────┘│
│       │                      │                     │                   │   │
│       │                      │                     │                   │   │
│       ▼                      ▼                     ▼                   ▼   │
│   S = √P              l = log₂(S)           L' = σ·l            L = ⌊L'⌉  │
│                                                                             │
│                              ERROR INJECTION POINT: ≤ 1/(2σ)               │
│                                                    └─────────────┘          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DECODING (Integer → Continuous):                                           │
│                                                                             │
│  ┌─────┐    ÷σ     ┌─────────┐   exp2fp   ┌─────────┐   ÷2^Qs   ┌─────────┐│
│  │  L  │ ────────► │ Approx  │ ─────────► │  S_fp   │ ────────► │ Approx  ││
│  │(int)│           │   l̂     │            │ (fixed) │           │   Ŝ     ││
│  └─────┘           └─────────┘            └─────────┘           └─────────┘│
│       │                 │                      │                     │     │
│       │                 │                      │                     │     │
│       ▼                 ▼                      ▼                     ▼     │
│   L ∈ ℤ            l̂ = L/σ              S_fp = exp2fp(L)      Ŝ = S_fp/2^Qs│
│                                                                             │
│              ERROR: ≤1/(2σ)        ERROR: ≤ε_exp2·S              TOTAL     │
│              └──────────┘          └────────────────┘          ERROR ≤ ε   │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ROUND-TRIP ERROR BOUND:                                                    │
│                                                                             │
│    |Ŝ - S|     1                                                           │
│    ─────── ≤ ───── · ln(2) + ε_exp2  →  0  as σ → ∞                        │
│      S        2σ                                                            │
│                                                                             │
│  With σ = 2^64:  relative error < 10^-19 (comparable to 128-bit float)     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

_Figure 3: The fixed-point encoding pipeline transforms continuous sqrt-price S into an integer L via logarithm, scaling by σ = 2^F, and rounding. Decoding reverses the process using a fixed-point exp₂ approximation. Error is introduced at the rounding step (bounded by 1/2σ) and the exponential approximation (bounded by ε_exp2). Total error vanishes as precision increases._

<!-- LaTeX/TikZ version for PDF:
\begin{figure}[h]
\centering
\begin{tikzpicture}[node distance=2cm, auto]
  \node (S) [draw, rectangle] {Real $S$};
  \node (l) [draw, rectangle, right of=S] {Real $l$};
  \node (L) [draw, rectangle, right of=l] {Integer $L$};
  \draw[->] (S) -- node {$\log_2$} (l);
  \draw[->] (l) -- node {$\times\sigma$, round} (L);
  % Add error annotations
\end{tikzpicture}
\caption{Fixed-Point Encoding Pipeline}
\label{fig:fixed-point-encoding}
\end{figure}
-->

## 5. Three-Layer Representation Architecture

We define a three-layer price representation architecture:

1. **Layer 0 (Continuous):** A real-valued log-sqrt-price axis $l \in \mathbb{R}$ which is the sole "source of truth" for prices in the economic model.

2. **Layer 1 (Global Slots):** A fixed discrete lattice in $l$-space used for on-chain storage and indexing of positions and orders.

3. **Layer 2 (Local Mesh):** A trade-specific discretization of a bounded interval of $l$-space used during execution. This mesh can be dynamically adapted per trade to optimize accuracy and performance.

All three layers refer to the same underlying price coordinate $l$; layers differ only in how they discretize and organize that axis.

**Figure 4: Three-Layer Representation Architecture**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    THREE-LAYER PRICE REPRESENTATION                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  LAYER 0: CONTINUOUS (Mathematical Model)                    [PERSISTENT]   │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│     l ∈ ℝ (log-sqrt-price axis)                                            │
│     ──────────────────────────────────────────────────────────────────►    │
│                                                                             │
│     λ(l): CLMM liquidity density (piecewise constant)                      │
│     ┌───┐     ┌───────────┐           ┌─────┐                              │
│     │   │     │           │           │     │                              │
│     │ L₁│     │    L₂     │           │ L₃  │          λ(l)                │
│     │   │     │           │           │     │                              │
│     ┴───┴─────┴───────────┴───────────┴─────┴──────────────────►  l        │
│                                                                             │
│     μ(dl): CLOB orders as Dirac measures                                   │
│              ↑         ↑              ↑                                    │
│              │q₁       │q₂            │q₃     (atomic at l₀,ⱼ)             │
│     ─────────┴─────────┴──────────────┴────────────────────────►  l        │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  LAYER 1: GLOBAL SLOTS (On-Chain Storage)                    [PERSISTENT]   │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│     Discrete lattice: lₛ = l_ref + s · Δl,  s ∈ ℤ                          │
│                                                                             │
│         s=-2    s=-1    s=0     s=1     s=2     s=3     s=4                │
│     ─────┼───────┼───────┼───────┼───────┼───────┼───────┼─────►  l        │
│          │       │       │       │       │       │       │                 │
│          └───────┴───────┴───────┴───────┴───────┴───────┘                 │
│                         Δl (slot spacing)                                   │
│                                                                             │
│     • Each slot stores: LP bands overlapping it, orders nearest to it      │
│     • Enables O(1) lookup of relevant liquidity near current price         │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  LAYER 2: LOCAL MESH (Per-Trade Execution)                   [TRANSIENT]    │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│     Adaptive mesh for price path [l*, l_target]:                           │
│                                                                             │
│     l*        l₁    l₂       l₃  l₄         l₅           l_target          │
│     ●─────────▲─────○────────▲───○──────────▲────────────●                 │
│     │         │     │        │   │          │            │                 │
│     │ L_AMM,0 │     │L_AMM,2 │   │          │  L_AMM,5   │                 │
│     │         │     │        │   │          │            │                 │
│     └─────────┴─────┴────────┴───┴──────────┴────────────┘                 │
│                                                                             │
│     Legend:  ● = endpoints   ▲ = LP boundaries   ○ = CLOB order prices     │
│              L_AMM,k = segment liquidity (constant per segment)             │
│                                                                             │
│     • Built fresh for each trade, discarded after execution                │
│     • Contains all structural points needed for exact execution            │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  INFORMATION FLOW:                                                          │
│                                                                             │
│    Layer 0  ──────►  Layer 1  ──────►  Layer 2                             │
│    (defines         (indexes for      (executes trade                      │
│     economics)       fast lookup)      with precision)                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

_Figure 4: The three-layer architecture separates concerns: Layer 0 defines the continuous economic model with CLMM liquidity density λ(l) and CLOB atomic measures μ(dl). Layer 1 provides persistent on-chain indexing via a global slot lattice for efficient state lookup. Layer 2 constructs a transient, trade-specific mesh that captures all structural points (LP boundaries, order prices, endpoints) needed for exact execution. Layers 0 and 1 persist between trades; Layer 2 is rebuilt for each execution._

<!-- LaTeX/TikZ version for PDF:
\begin{figure}[h]
\centering
\begin{tikzpicture}
  % Three horizontal bands representing layers
  % Layer 0: continuous axis with piecewise constant function
  % Layer 1: discrete tick marks at regular intervals
  % Layer 2: adaptive mesh with different node types
  % Arrows showing information flow between layers
\end{tikzpicture}
\caption{Three-Layer Representation Architecture}
\label{fig:three-layer-architecture}
\end{figure}
-->

### 5.1 Layer 0: Continuous Log-Sqrt-Price Axis

#### 5.1.1 Price coordinate

Let:

- $P > 0$ be the spot price of token $B$ in terms of token $A$.
- $S = \sqrt P > 0$ be the square-root price.
- $l = \log_2 S = \frac{1}{2} \log_2 P \in \mathbb{R}$ be the continuous log-sqrt-price coordinate.

Then:

$$
S = 2^l, \quad P = S^2 = 2^{2l}
$$

We treat $l \in \mathbb{R}$ as the **fundamental price coordinate**. All other quantities (spot price, sqrt price) are derived functions of $l$:

$$
S(l) = 2^l, \quad P(l) = 2^{2l}
$$

#### 5.1.2 CLMM liquidity as a piecewise constant density

Consider a finite or countable family of CLMM positions indexed by $i \in I$, each specified by:

- Liquidity amount $L_i > 0$,
- Lower and upper log-sqrt bounds $l_{lower,i}, l_{upper,i} \in \mathbb{R}$ with $l_{lower,i} < l_{upper,i}$.

Position $i$ contributes constant liquidity $L_i$ on the interval $[l_{lower,i}, l_{upper,i}]$ and zero outside.

Define the **liquidity density function**:

$$
\lambda : \mathbb{R} \to [0, \infty), \quad \lambda(l) = \sum_{i \in I} L_i \cdot 1_{[l_{lower,i}, l_{upper,i}]}(l)
$$

**Intuitively**, $\lambda(l)$ tells us "how much liquidity is available at log-price $l$." Each LP contributes their liquidity $L_i$ only within their chosen range; outside that range, they contribute nothing. The total liquidity at any price is simply the sum of all LPs who have "turned on" their liquidity at that price.

Here $1_I$ is the indicator function of interval $I$:

$$
1_I(l) = \begin{cases}1, & \ l \in I \\ 0, & \ l \notin I \end{cases}
$$

Assumption: we implicitly assume the family ${(l_{lower,i}, l_{upper,i})}_{i \in I}$ is such that for every compact interval $K \subset \mathbb{R}$, the sum $\sum_{i \in I} L_i \cdot 1_{[l_{lower,i}, l_{upper,i}]}(l)$ is finite for all $l \in K$ (i.e. locally finite liquidity). This ensures $\lambda(l)$ is well-defined and finite on any bounded region of interest.

Property:

- $\lambda$ is a **piecewise constant** with jumps only at the endpoints of the intervals $l_{lower,i}, l_{upper,i}$.

At any $l$, the total **active CLMM liquidity** in Layer 0 (Section 5.1) is $\lambda(l)$.

**Figure 5: CLMM Liquidity Density Function**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CLMM LIQUIDITY AS PIECEWISE CONSTANT DENSITY             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Individual LP Positions (each contributes Lᵢ on [l_lower,i, l_upper,i]):  │
│                                                                             │
│  LP₁:           ┌───────────┐                                              │
│  (L₁=100)       │           │                                              │
│                 └───────────┘                                              │
│                                                                             │
│  LP₂:      ┌─────────────────────────┐                                     │
│  (L₂=150)  │                         │                                     │
│            └─────────────────────────┘                                     │
│                                                                             │
│  LP₃:                    ┌─────────────────┐                               │
│  (L₃=80)                 │                 │                               │
│                          └─────────────────┘                               │
│                                                                             │
│  ─────────┴─────┴────────┴─────┴───────────┴─────────────────────────► l   │
│         l₁,low  l₂,low  l₁,up l₃,low     l₂,up  l₃,up                      │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Aggregate Liquidity λ(l) = Σᵢ Lᵢ · 𝟙[l_lower,i, l_upper,i](l):            │
│                                                                             │
│  λ(l)                                                                       │
│   ▲                                                                         │
│   │                                                                         │
│ 330├─────────────────┬─────┐                                               │
│   │                 │     │  L₁+L₂+L₃ = 330                                │
│ 250├──────────┬─────┘     │                                                │
│   │          │           │  L₁+L₂ = 250                                    │
│ 150├──┬──────┘           └─────┐                                           │
│   │  │                        │  L₂ = 150                                  │
│  80│  │                        └────────┐                                  │
│   │  │                                  │  L₃ = 80                         │
│   0└──┴──────────────────────────────────┴──────────────────────────► l    │
│      │   │        │    │            │        │                              │
│    l₂,low l₁,low  l₁,up l₃,low     l₂,up   l₃,up                           │
│                                                                             │
│  KEY: λ(l) jumps only at LP band boundaries (l_lower,i or l_upper,i)       │
│       λ(l) is constant between consecutive boundaries                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

_Figure 5: CLMM liquidity is modeled as a piecewise constant density function λ(l). Each LP position i contributes constant liquidity Lᵢ within its range [l_lower,i, l_upper,i]. The aggregate liquidity at any point l is the sum of all LP contributions active at that point. Jumps in λ(l) occur only at LP band boundaries, which become mandatory mesh nodes in Layer 2._

<!-- LaTeX/TikZ version for PDF:
\begin{figure}[h]
\centering
\begin{tikzpicture}
  \draw[->] (0,0) -- (10,0) node[right] {$l$};
  \draw[->] (0,0) -- (0,4) node[above] {$\lambda(l)$};
  % Draw overlapping rectangles for LP positions
  % Draw resulting step function
\end{tikzpicture}
\caption{CLMM Liquidity Density Function}
\label{fig:clmm-liquidity-density}
\end{figure}
-->

#### 5.1.3 CLOB orders as an atomic measure on $l$

A single limit order $j$ is given by:

- Side $side_j \in \{buy, sell\}$
- Target price $P_{0,j} > 0$, equivalently target sqrt price $S_{0,j} = \sqrt{P_{0,j}}$
- Corresponding log-sqrt-price:

$$
l_{0,j} = \log_2 S_{0,j} = \frac{1}{2} \log_2 P_{0,j}
$$

- Quantity $q_j > 0$ (measured in some canonical unit, e.g. quantity of $B$ for buy orders and quantity of $A$ or $B$ for sell orders; the precise convention can be defined later).

We model CLOB orders as an **atomic measure** (a sum of point masses, where each limit order contributes a Dirac delta at its price level) on $\mathbb{R}$ (the $l$-axis):

Define the signed or vector-valued measure (depending on side conventions) by:

$$
\mu(dl) = \sum_j q_j \cdot \delta_{l_{0,j}}(dl - l_{0,j})
$$

where $\delta(\cdot - l_0)$ denotes the Dirac measure at point $l_0$.

For a Borel set $A \subset \mathbb{R}$:

$$
\mu(A) = \sum_{j: l_{0,j} \in A} q_j
$$

At a specific $l \in \mathbb{R}$, the "point depth" of the order book is:

$$
\mu(\{l\}) = \sum_{j: l_{0,j} = l} q_j
$$

Note:

- In practice, exact equality $l_{0,j} = l$ is rare; for execution we will work with local neighborhoods and local mesh (Section 5.3, Layer 2). The formal model here treats them as exact points for mathematical clarity.

#### 5.1.4 Combined liquidity and depth

At Layer 0, the complete economic state along the $l$-axis is characterized by:

- The **CLMM liquidity density** $\lambda(l)$,
- The **CLOB order measure** $\mu$.

These can be combined (conceptually) into a generalized "liquidity measure" consisting of:

- A **continuous (piecewise constant) part** given by $\lambda(l)$,
- A **purely atomic part** given by $\mu$.

Perpetuals and risk logic reference the same axis $l$ for:

- mark price $P(l)$,
- margin requirements,
- liquidation thresholds, etc.

Layer 0 is purely continuous and independent of discretization.

### 5.2 Layer 1: Global Log-Price Slots (Persistent Grid)

Layer 1 introduces a **global regular grid** on $l$-space to organize on-chain state. This grid is **implementation specific** and does not change the economics from Layer 0 (Section 5.1).

#### 5.2.1 Slot Lattice and Anchors

Fix two real constants:

- A base origin $l_{ref} \in \mathbb{R}$,
- A positive step $\Delta l > 0$ (slot spacing in log-sqrt space).

Define **slot indices**:

$$
s \in \mathbb{Z}
$$

and associate to each slot $s$ an anchor:

$$
l_s = l_{ref} + s \cdot \Delta l
$$

Thus:

$$
\{ l_s : s \in \mathbb{Z} \} = l_{ref} + \Delta l \cdot \mathbb{Z}
$$

is a regular lattice in $\mathbb{R}$.

If desired, we may also define an **integer encoding** of $l_s$ using the fixed-point scaling factor $\sigma$ from Section 4.1:

- Recall: log-sqrt-price encode/decode:

$$
L = round(\sigma \cdot l) \in \mathbb{Z}, \quad \hat l = \frac{L}{\sigma}
$$

Then for slot $s$:

$$
L_s = round(\sigma \cdot l_s) = round(\sigma \cdot (l_{ref} + s \cdot \Delta l)) \in \mathbb{Z}
$$

This gives an integer representation of the slot anchors suitable for on-chain storage and arithmetic. This is an implementation detail; mathematically, Layer 1 is defined in terms of real anchors $l_s$.

#### 5.2.2 Slot index of an arbitrary log price

Given any $l \in \mathbb{R}$, define its **slot index** by a chosen rounding rule.

Typical choices:

- **Floor slot**:

$$
s_{\lfloor \cdot \rfloor}(l) = \left\lfloor \frac{l - l_{ref}}{\Delta l} \right\rfloor
$$

so that:

$$
l_{s_{\lfloor \cdot \rfloor}(l)} \leq l < l_{s_{\lfloor \cdot \rfloor}(l) + 1}
$$

- **Nearest slot**:

$$
s_{\text{round}}(l) = \text{round}\left( \frac{l - l_{ref}}{\Delta l} \right)
$$

so that:

$$
\left| l - l_{s_{\text{round}}(l)} \right| \leq \frac{\Delta l}{2}
$$

We will use:

- Floor for **interval coverage** (LP bands),
- Nearest for **point-based indexing** (limit orders).

#### 5.2.3 Slot indexing for LP bands

Consider a position $i$ with band $[l_{lower,i}, l_{upper,i}]$ in Layer 0.

Define its **slot coverage indices**:

$$
s_{lower,i} = \left\lfloor \frac{l_{lower,i} - l_{ref}}{\Delta l} \right\rfloor , \quad s_{upper,i} = \left\lceil \frac{l_{upper,i} - l_{ref}}{\Delta l} \right\rceil
$$

Then we have:

$$
l_{s_{lower,i}} \leq l_{lower,i}, \quad l_{upper,i} \leq l_{s_{upper,i}}
$$

and the band $[l_{lower,i}, l_{upper,i}]$ is contained in the **slot interval**:

$$
[l_{s_{lower,i}}, l_{s_{upper,i}}] = [l_{ref} + s_{lower,i} \Delta l, \ l_{ref} + s_{upper,i} \Delta l]
$$

We say that position $i$ **spans slots**:

$$
[s_{lower,i}, s_{upper,i}] \cap \mathbb{Z}
$$

On-chain, we can maintain, for each slot index $s$, a list of positions $i$ such that:

$$
s \in [s_{lower,i}, s_{upper,i}]
$$

i.e. positions whose bands intersect the slot's anchor region. This allows us to discover candidate LP positions near a given price by searching slots in a small neighborhood of the current slot.

Formally, define the per-slot LP set:

$$
\mathcal{I}_s = \{ i \in I : s_{lower,i} \leq s \leq s_{upper,i} \}
$$

Then for any $l$ such that $l \in [l_s, l_{s+1})$, we know any band covering $l$ is included in some $\mathcal{I}_{s'}$ with $s' \in \{s-1, s, s+1\}$ (depending on how you define coverage). Implementation can refine the exact window.

#### 5.2.4 Slot indexing of limit orders

Let order $j$ have log-sqrt-price $l_{0,j}$.

Define its **slot index** via nearest-slot rounding:

$$
s_j = \text{round}\left( \frac{l_{0,j} - l_{ref}}{\Delta l} \right)
$$

By construction:

$$
\left| l_{0,j} - l_{s_j} \right| \leq \frac{\Delta l}{2}
$$

On-chain, we store order IDs in per-slot order lists $\mathcal{J}_s$, where:

$$
\mathcal{J}_s = \{ j : s_j = s \}
$$

Crucially:

- We store the **exact** $l_{0,j}$ (or exact encoded price) inside the order's record.
- The slot index $s_j$ is a **pure indexing hint** used to quickly discover orders whose prices are near the current price.

Approximation property:

- If at current log-price $l_*$ we want to find all orders with $\left| l_{0,j} - l_* \right| \leq \mathbb{R}$, we can search slots:

$$
s \in \left[ s_{\text{round}}(l_* - \mathbb{R}), \ s_{\text{round}}(l_* + \mathbb{R}) \right]
$$

and then filter by the exact $l_{0,j}$ in each slot's list.

#### 5.2.5 Layer 1 as an implementation layer

Summary of Layer 1 (Section 5.2):

- The pair $(l_{ref}, \Delta l)$ defines a **global grid resolution** in log-sqrt space.
- Each LP band and each CLOB order is mapped to **integer slot indices** for storage:
  - LP positions: interval of slots $[s_{lower,i}, s_{upper,i}]$,
  - CLOB orders: single slot $s_j$.
- This mapping is **many-to-one**: many LPs and orders can live in the same slot; a single band may span multiple slots.
- $\Delta l$ is an implementation parameter; it does not constrain Layer-0 prices, which remain fully continuous in $l$.

### 5.3 Layer 2: Local Execution Mesh (Adaptive Discretization)

Layer 2 defines a **trade-specific discretization** over a bounded interval of $l$-space. Conceptually:

- Layer 2 takes the continuous data ($\lambda$, $\mu$) from Layer 0 (Section 5.1),
- Restricts it to a price path interval,
- and approximates it by a finite number of **nodes** and **segments**.

This local mesh is built transiently for each execution and discarded afterward.

#### 5.3.1 Price path interval

Let:

- $l_* \in \mathbb{R}$ be the current log-sqrt-price,
- $l_{target} \in \mathbb{R}$ be a predicted or upper bound on the final log-sqrt-price the trade may reach. The exact $l_{target}$ may be derived from trade size estimates, user price limits, or a bounding interval for execution.

Define the **price path interval** $I_{path}$ as:

$$
I_{path} = \begin{cases}
[l_*, l_{target}], & \text{if } l_{target} \ge l_* \\
[l_{target}, l_*], & \text{if } l_{target} < l_*
\end{cases}
$$

By definition, $I_{path}$ is a compact interval in $\mathbb{R}$.

Define:

- $LP\_set(I_{path}) = \{ i : [l_{lower,i}, l_{upper,i}] \cap I_{path} \neq \emptyset \}$: LPs that intersect the path,
- $ORD\_set(I_{path}) = \{ j : l_{0,j} \in I_{path} \}$: orders whose prices lie in the path.

These sets are **finite** under reasonable assumptions (finite state per pair, bounded book near current price).

#### 5.3.2 Structural Mesh Points

We define a preliminary set of **structural points** in $I_{path}$:

1. **LP boundaries inside the path.**

   For each $i \in LP\_ set(I_{path})$, include:

   - $l_{lower,i}$ if $l_{lower,i}$ $\in I_{path}$,
   - $l_{upper,i}$ if $l_{upper,i}$ $\in I_{path}$.

   Denote this set:

   $$
   \mathcal{B}_{LP} = \{ l_{lower,i} : i \in LP\_ set(I_{path}), l_{lower,i} \in I_{path} \} \cup \{ l_{upper,i} : i \in LP\_ set(I_{path}), l_{upper,i} \in I_{path} \}
   $$

2. **Limit order prices inside the path.**

   For each $j \in ORD\_ set(I_{path})$, include $l_{0,j}$.

   Denote this set:

   $$
   \mathcal{B}_{ORD} = \{ l_{0,j} : j \in ORD\_ set(I_{path}) \}
   $$

3. **Endpoints of the path.**

Always include $l_*$ and $l_{target}$:

$$
\mathcal{B}_{END} = \{ l_*, l_{target} \}
$$

Define the set of **mandatory mesh nodes**:

$$
\mathcal{B} = \mathcal{B}_{LP} \cup \mathcal{B}_{ORD} \cup \mathcal{B}_{END}
$$

By construction:

- $\mathcal{B}$ is a finite (assuming finite LP/ORD in a bounded region),
- $\mathcal{B} \subset I_{path}$,
- $\mathcal{B}$ contains all LP band boundaries and order prices within the path.

#### 5.3.3 Optional refinement points

To control numerical error and microstructure granularity, the protocol may add **refinement points** inside sub-intervals.

**Figure 6: Local Mesh Construction for Trade Execution**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                      LAYER 2: LOCAL MESH CONSTRUCTION                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  INPUTS:                                                                    │
│  • Current price l* and target l_target define I_path = [l*, l_target]     │
│  • LP bands intersecting I_path: {(l_lower,i, l_upper,i, Lᵢ)}              │
│  • CLOB orders in I_path: {(l₀,ⱼ, qⱼ)}                                      │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STEP 1: Collect Mandatory Structural Points                               │
│                                                                             │
│      l*                                                 l_target            │
│      │                                                      │               │
│      ▼                                                      ▼               │
│  ────■──────────────────────────────────────────────────────■────► l       │
│      │         │         │    │     │         │             │               │
│      │    l_lower,1  l_upper,1 │  l_lower,2  l_upper,2      │               │
│      │         ▼         ▼    │     ▼         ▼             │               │
│  ────■─────────▲─────────▲────●─────▲─────────▲─────────────■────► l       │
│                                    │                                        │
│                              l₀,ⱼ (order)                                   │
│                                                                             │
│  Legend:  ■ = endpoints (B_END)                                            │
│           ▲ = LP boundaries (B_LP)                                         │
│           ● = CLOB order prices (B_ORD)                                    │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STEP 2: (Optional) Add Refinement Points                                  │
│                                                                             │
│      ■─────────▲────·────▲────●────·▲─────────▲─────────────■               │
│                     ·              ·                                        │
│                   refinement points (for numerical precision)               │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STEP 3: Sort and Deduplicate → Final Mesh M                               │
│                                                                             │
│      l₀      l₁      l₂     l₃    l₄     l₅      l₆      l₇                │
│      ■───────▲───────·──────▲─────●──────▲───────▲───────■                  │
│      │       │       │      │     │      │       │       │                  │
│      └───┬───┴───┬───┴──┬───┴──┬──┴──┬───┴───┬───┴───┬───┘                  │
│          │       │      │      │     │       │       │                      │
│       L_AMM,0  L_AMM,1 L_AMM,2 L_AMM,3 L_AMM,4 L_AMM,5 L_AMM,6              │
│                                                                             │
│      Each segment [lₖ, lₖ₊₁] has constant liquidity L_AMM,k                │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PROPERTIES:                                                                │
│  ✓ M = {l₀ < l₁ < ... < lₙ} is finite and ordered                         │
│  ✓ All LP boundaries in I_path appear as mesh nodes                        │
│  ✓ All CLOB order prices in I_path appear as mesh nodes                    │
│  ✓ λ(l) is constant on each open segment (lₖ, lₖ₊₁)                        │
│  ✓ Segment formulas apply exactly with L = L_AMM,k                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

_Figure 6: Layer 2 constructs a local mesh M by collecting all structural points within the trade's price path interval: LP band boundaries, CLOB order prices, and path endpoints. Optional refinement points may be added for numerical precision. The resulting sorted mesh partitions I_path into segments, each with constant CLMM liquidity, enabling exact application of the segment execution formulas._

<!-- LaTeX/TikZ version for PDF:
\begin{figure}[h]
\centering
\begin{tikzpicture}
  % Number line showing I_path
  % Different symbols for B_END, B_LP, B_ORD
  % Segments labeled with L_AMM,k
\end{tikzpicture}
\caption{Local Mesh Construction for Trade Execution}
\label{fig:local-mesh-construction}
\end{figure}
-->

To keep the mesh size tractable while improving accuracy, we may add refinement points according to the following strategies:

Let $\mathcal{R} \subset I_{path}$ be a finite set of additional points determined by some rule, e.g.:

- Subdivision by a fixed log-step $\delta l_{refine} > 0$,
- Adaptive subdivision where $\left | \lambda(l) \right|$ is large,
- Points chosen to satisfy specific error bounds on CLMM integrals.

We require:

- $\mathcal{R}$ is finite,
- $\mathcal{R}$ does not duplicate points in $\mathcal{B}$ (or we can deduplicate later).

The final **mesh node set** is:

$$
\mathcal{M} = \mathcal{B} \cup \mathcal{R} \subset I_{path}
$$

Define $\mathcal{M}$ sorted in strictly increasing order:

$$
\mathcal{M} = \{ l_0 < l_1 < \ldots < l_{N} \}, \quad {l_0 < l_1 < \ldots < l_{N}}
$$

By construction:

- $l_0 = \min(I_{path})$, $l_{N} = \max(I_{path})$,
- Every LP boundary and order price in $I_{path}$ appears as some $l_k$.

The mesh partitions the path:

$$
I_{path} = \bigcup_{k=0}^{N-1} [l_k, l_{k+1}]
$$

with pairwise disjoint interiors.

#### 5.3.4 Local discretization of AMM liquidity

On each segment $[l_k, l_{k+1}]$ we approximate the continuous liquidity density $\lambda(l)$ from Layer 0 by a **segment-wise constant** effective liquidity $L_{AMM,k}$.

Possible choices:

1. **Midpoint sampling:**

   Let

   $$
   \bar{l}_k = \frac{l_k + l_{k+1}}{2}
   $$

   Define:

   $$
   L_{AMM,k} = \lambda(\bar{l}_k)
   $$

2. **Exact average**:

   If we want an exact match to the integral of $\lambda$ over the segment:

   $$
   L_{AMM,k} = \frac{1}{l_{k+1} - l_k} \int_{l_k}^{l_{k+1}} \lambda(l) \ dl
   $$

   Since $\lambda$ is piecewise constant in $l$, this integral is trivial to compute exactly from the bands.

In either case, define the **segment liquidity approximation**:

$$
\lambda_{k}(l) = \begin{cases} L_{AMM,k}, & l \in [l_k, l_{k+1}] \\ 0, & \text{otherwise} \end{cases}
$$

and the **mesh-based liquidity density**:

$$
\lambda^{(\mathcal{M})} (l) = \sum_{k=0}^{N-1} L_{AMM,k} \cdot 1_{[l_k, l_{k+1}]}(l)
$$

Error remark:

- As the **mesh diameter** $max_{k} (l_{k+1} - l_k) \to 0$ and sampling/averaging is chosen appropriately, $\lambda^{(\mathcal{M})}(l) \to \lambda(l)$ almost everywhere.
- For piecewise constant $\lambda$, if mesh boundaries include all band boundaries, the exact choice $L_{AMM,k}= \lambda(\bar{l}_k)$ yields **zero error** almost everywhere; the only differences can occur at endpoints, which are measure-zero for integrals.

#### 5.3.5 Local discretization of CLOB depth

On the mesh nodes {$l_k$}, we attach the CLOB atomic measure:

For each node $l_k$, define the set of orders at that node:

$$
\mathcal{J}_k = \{ j : l_{0,j} = l_k \}
$$

For each $k$, the total quantity at $l_k$ is:

$$
Q_{CLOB,k} = \sum_{j \in \mathcal{J}_k} q_j
$$

In the ideal continuous model, many orders may have prices not exactly equal to the chosen $l_k$. In practice, the mesh construction would:

- either include $l_{0,j}$ as a mesh node if $j \in ORD\_ set(I_{path})$ (which we already do in $\mathcal{B}_{ORD}$),
- or approximate price levels if you choose to snap them.

In our formal construction above (Section 5.3.2), we **included every** $l_{0,j}$ in $\mathcal{M}$, so:

- For any $j \in ORD\_ set(I_{path})$, there exists a unique $k$ such that $l_k = l_{0,j}$,
- Therefore, every order in the path is represented exactly at some node.

Layer-2 "CLOB structure" on the mesh is then:

- A finite set of nodes with attached order queues (buys/sells) and total depth $Q_{CLOB,k}$.

#### 5.3.6 Execution along the local mesh

Given a trade (swap, market order, liquidation, etc.), we proceed as follows (at a high level).

Let:

- $l_{curr} = l_*$ be the starting node (there is some index $k_*$ such that $l_{k_*} = l_{curr}$; if not, we can treat $l_*$ as lying within a segment and start from that segment),
- A trade direction (e.g. buy $B$ with $A$ or vice versa) determines the direction in $l$-space:

  - e.g. buying $B$ with $A$ might push price up, so we move from low $l$ to high $l$,
  - selling $B$ might push price down, so we move from high $l$ to low $l$.

The execution algorithm:

1. **Initialization.**

   - Determine direction of price movement and thus whether to traverse nodes $k_*, k_*+1, k_*+2, \ldots$ or $k_*, k_*-1, k_*-2, \ldots$
   - Initialize remaining trade size (quantity to buy/sell).

2. **CLOB matching at current node.**

   At node $l_{k}$:

   - Identify opposing orders in $\mathcal{J}_k$ (depending on trade side).
   - Match the trade against these orders up to:
     - the node's total quantity $Q_{CLOB,k}$,
     - or until the remaining trade quantity is exhausted.
     - or until a user-defined or protocol-defined price/size limit is reached.
   - Update order book state and trader balances accordingly.

3. **AMM execution on the next segment.**

   If there is remaining trade quantity after CLOB matching at $l_k$, we consider the adjacent segment in the direction of price movement:

   - If moving up: segment $[l_k, l_{k+1}]$ with liquidity $L_{AMM,k}$,
   - If moving down: segment $[l_{k-1}, l_k]$ with liquidity $L_{AMM,k-1}$.

   We then apply the CLMM invariant over that segment, using the approximated constant liquidity $L_{AMM,k}$ and the mapping $P(l) = 2^{2l}$ to compute:

   - The **maximum tradable amount** on that segment before reaching the boundary $l_{k+1}$ (or $l_{k-1}$),
   - the resulting change in $l$ if we execute only a partial portion of the segment capacity.

   Let:

   - $\Delta qty_k^{max}$ = maximum quantity the AMM can offer on that segment,
   - $\Delta qty_{needed}$ = remaining quantity for the trade.

   If $\Delta qty_{needed} \leq \Delta qty_k^{max}$, we:

   - Compute the final log-price $l_{final} \in [l_k, l_{k+1}]$, consistent with the invariant and the consumed amount,
   - Update balances and terminate the trade.

   If $\Delta qty_{needed} > \Delta qty_k^{max}$, we:

   - Consume the entire segment capacity, moving price to the boundary $l_{k+1}$ (or $l_{k-1}$),
   - Update balances and remaining trade amount,
   - Move to the next node and repeat from step 2.

4. **Termination conditions.**

   Execution terminates when:

   - The trade is fully filled (remaining quantity = 0),
   - The path endpoint $l_{target}$ is reached,
   - Liquidity is exhausted (no more CLMM liquidity or CLOB orders in the direction),
   - or some risk/user constraint (max slippage, oracle bounds, etc.) is hit.

From a mathematical perspective, this is equivalent to integrating the continuous CLMM liquidity and CLOB atoms along the path with a **piecewise constant** approximation $\lambda^{(\mathcal{M})}$ and exact CLOB points at nodes.

#### 5.3.7 Convergence to the continuous model

Let $\mathcal{M}_n$ be a sequence of meshes on $I_{path}$ such that:

- All band boundaries and order prices are always included in $\mathcal{M}_n$,
- The mesh **diameter** $\delta_n = \max_{k} (l_{k+1}^{(n)} - l_k^{(n)}) \to 0$ as $n \to \infty$.
- The segment liquidity approximations $L_{AMM,k}^{(n)}$ is chosen via midpoint or average sampling.

Then:

- The mesh-based liquidity density $\lambda^{(\mathcal{M}_n)}$ converges to $\lambda$ almost everywhere,
- The integrated quantities (e.g. total amount of token traded along the path) computed by the mesh-based execution converge to those of the ideal continuous model, assuming the CLMM differential equations in $l$ are well-posed and Lipschitz in the relevant region.

Intuitively: as segments get finer, the discrete stepwise execution becomes indistinguishable (within numerical precision) from integrating the continuous dynamics of the underlying AMM + CLOB along $l$.

### 5.4 CLMM Dynamics in Log-Sqrt-Price Coordinates

We keep the token notation from Section 1:

- Tokens: $A$ and $B$,
- Price of $B$ in terms of $A$: $P > 0$,
- Square-root price: $S = \sqrt P > 0$,
- Log-sqrt-price: $l = \log_2 S = \frac{1}{2} \log_2 P$, so $S = 2^l$, $P = 2^{2l}$.

We consider the CLMM part of the system with **instantaneous aggregate liquidity** $\lambda(l)$ as defined in Layer 0 (Section 5.1.2):

$$
\lambda(l) = \sum_{i \in I} L_i \cdot 1_{[l_{lower,i}, l_{upper,i}]}(l)
$$

On any open interval where $\lambda$ is constant, the CLMM behaves like a classical concentrated liquidity AMM with **constant liquidity** over that price region.

#### 5.4.1 CLMM Differential in Sqrt-price (S)

On an interval where the **aggregate liquidity** is constant and equal to some value $L > 0$, the standard Uniswap v3-style CLMM formulas [@adams2021uniswap] for moving the square-root price from $S$ to $S + dS$ are:

- Pool's infinitesimal change in token $A$ reserves:

$$
\frac{dA}{dS} = - \frac{L}{S^2}
$$

- Pool's infinitesimal change in token $B$ reserves:

$$
\frac{dB}{dS} = L
$$

These differential relations integrate to the familiar finite formulas for constant liquidity $L$ over a price move from $S_a$ to $S_b$:

$$
\Delta A = L \left( \frac{1}{S_a} - \frac{1}{S_b} \right), \quad \Delta B = L (S_b - S_a)
$$

where $\Delta A, \Delta B$ are the pool's reserve changes (the trader's changes are $-\Delta A, -\Delta B$).

The signs are determined by the direction of $S$:

- If $S_b > S_a$, then $\Delta B > 0$ and $\Delta A < 0$,
- If $S_b < S_a$, then $\Delta B < 0$ and $\Delta A > 0$.

We will keep this orientation and simply regard $\Delta A, \Delta B$ as **pool deltas** as a function of the ordered pair $(S_a, S_b)$.

#### 5.4.2 Change of Variables to Log-Sqrt-Price ($l$)

We now express everything in terms of $l$.

Recall:

$$
S = 2^l, \quad \frac{dS}{dl} = (\ln 2) \cdot 2^l = (\ln 2) \cdot S
$$

so:

$$
dS = (\ln 2) \cdot S \ dl
$$

On an interval where the aggregate liquidity equals some constant $L$, we have:

$$
\frac{dA}{dl} = \frac{dA}{dS} \cdot \frac{dS}{dl} = -\frac{L}{S^2} \cdot (\ln 2) \cdot S = - L (\ln 2) \cdot \frac{1}{S}
$$

Similarly:

$$
\frac{dB}{dl} = \frac{dB}{dS} \cdot \frac{dS}{dl} = L \cdot (\ln 2) \cdot S
$$

Using $S = 2^l$ and $\frac{1}{S} = 2^{-l}$, we can write:

$$
\frac{dA}{dl} = - L (\ln 2) \cdot 2^{-l}, \quad \frac{dB}{dl} = L (\ln 2) \cdot 2^{l}
$$

Thus, in **log-sqrt-price coordinates** $l$, the CLMM differential system for a constant-liquidity region is:

$$
\boxed{
dA(l) = - L (\ln 2) \cdot 2^{-l} \ dl
, \quad
dB(l) = L (\ln 2) \cdot 2^{l} \ dl
}
$$

**Intuitively**, as the price moves up ($dl > 0$), the pool gains token $B$ and loses token $A$—this is because buyers are purchasing $B$ with $A$. The factors $2^{-l}$ and $2^l$ capture how the same percentage price move requires different absolute token amounts at different price levels: at higher prices, each tick costs more $A$ but fewer $B$.

Again, these are **pool-level differentials**; the trader's infinitesimal changes are the negatives.

#### 5.4.3 Integrated Form in $l$: Finite Moves

Suppose we move log-sqrt-price from $l_a$ to $l_b$ with $l_b \neq l_a$, in a region where liquidity is constant and equal to $L$.

Define the pool deltas:

$$
\Delta A (l_a \to l_b; L) := A(l_b) - A(l_a), \quad \Delta B (l_a \to l_b; L) = B(l_b) - B(l_a)
$$

Integrate the differentials:

$$
\Delta A (l_a \to l_b; L) = \int_{l_a}^{l_b} dA(l) = -L (\ln 2) \int_{l_a}^{l_b} 2^{-l} \ dl
$$

But:

$$
\int 2^{-l} \ dl = \frac{2^{-l}}{-\ln 2} + C
$$

so:

$$
\Delta A (l_a \to l_b; L) = -L (\ln 2) \left[ \frac{2^{-l}}{-\ln 2} \right]_{l_a}^{l_b} = L \left[ 2^{-l} \right]_{l_a}^{l_b} = L (2^{-l_b} - 2^{-l_a})
$$

Similarly:

$$
\Delta B (l_a \to l_b; L) = \int_{l_a}^{l_b} dB(l) = L (\ln 2) \int_{l_a}^{l_b} 2^{l} \ dl
$$

And:

$$
\int 2^{l} \ dl = \frac{2^{l}}{\ln 2} + C
$$

so:

$$
\Delta B (l_a \to l_b; L) = L (\ln 2) \left[ \frac{2^{l}}{\ln 2} \right]_{l_a}^{l_b} = L \left[ 2^{l} \right]_{l_a}^{l_b} = L (2^{l_b} - 2^{l_a})
$$

Hence the exact **finite-move formulas** in terms of $l$ are:

$$
\boxed{
\Delta A (l_a \to l_b; L) = L (2^{-l_b} - 2^{-l_a}), \quad \Delta B (l_a \to l_b; L) = L (2^{l_b} - 2^{l_a})
}
$$

**Intuitively**, these formulas give the exact token amounts exchanged when price moves from $l_a$ to $l_b$. If price goes up ($l_b > l_a$), then $\Delta B > 0$ (pool receives $B$) and $\Delta A < 0$ (pool pays out $A$). The liquidity parameter $L$ acts as a multiplier: more liquidity means larger trades are needed to move the price by the same amount.

These are equivalent to the classical CLMM formulas expressed in terms of $S$:

- Since $S_a = 2^{l_a}$ and $S_b = 2^{l_b}$, we have:

$$
2^{-l_a} = \frac{1}{S_a}, \quad 2^{-l_b} = \frac{1}{S_b}
$$

Thus:

$$
\Delta A (l_a \to l_b; L) = L \left( \frac{1}{S_b} - \frac{1}{S_a} \right), \quad \Delta B (l_a \to l_b; L) = L (S_b - S_a)
$$

which matches the known CLMM expressions.

#### 5.4.4 Extension to General Liquidity Density $\lambda(l)$

In the continuous Layer-0 model (Section 5.1), aggregate liquidity can vary with $l$:

$$
\lambda(l) = \sum_{i \in I} L_i \cdot 1_{[l_{lower,i}, l_{upper,i}]}(l)
$$

On any open interval where $\lambda$ is constant (no band endpoints inside), we can set $L = \lambda(l)$ and apply the previous formulas. At **band boundaries** where $\lambda$ jumps, the differential equations still hold away from the point, and finite moves are obtained by piecewise integration across the intervals.

In differential form, for arbitrary $\lambda(l)$, we can write:

$$
\boxed{
dA(l) = - \lambda(l) (\ln 2) \cdot 2^{-l} \ dl
, \quad
dB(l) = \lambda(l) (\ln 2) \cdot 2^{l} \ dl
}
$$

Integrating from $l_a$ to $l_b$:

$$
\Delta A (l_a \to l_b) = - (\ln 2) \int_{l_a}^{l_b} \lambda(l) \cdot 2^{-l} \ dl, \quad \Delta B (l_a \to l_b) = (\ln 2) \int_{l_a}^{l_b} \lambda(l) \cdot 2^{l} \ dl
$$

In our setting, $\lambda(l)$ is piecewise constant and we will deliberately choose the local mesh to have nodes at all jump points (band boundaries). That allows us to treat $\lambda$ as constant on each segment ($l_k, l_{k+1}$), which leads directly into the segment-wise execution formulas (Section 5.5).

**Remark (Custom Liquidity Densities):** The formalism above applies to **arbitrary** $\lambda(l)$, not only piecewise constant bands. This allows future extensions such as: (1) Gaussian or smooth density profiles concentrated around active prices, (2) volatility-responsive liquidity adjustments based on market conditions, and (3) risk-optimized LP strategies that minimize impermanent loss [9, 10]. Such distributions can be represented exactly in the Layer-0 continuous model (Section 5.1) and approximated via adaptive mesh refinement in Layer 2 (Section 5.3), enabling capital-efficient strategies beyond traditional uniform CLMM bands.

#### 5.4.5 Alternative Invariants

Throughout this subsection we have used the standard constant-product CLMM as a canonical example, leading to the familiar $x y = k$-style behavior when expressed in suitable coordinates. However, the Layer-0 / Layer-2 machinery developed in Sections 4.1–4.3 does not depend on this particular invariant. The only structural requirements are that the AMM admit:

- a well-defined price process that can be parameterized as a monotone function of the log-sqrt coordinate $l$, and
- well-defined infinitesimal token-flow differentials $(dA(l), dB(l))$ along that price process.

Any CFMM whose invariant can be written in this form can be represented as a dynamics on $l$ and then discretized on the local mesh exactly as in Section 5.5, with liquidity summarized in an aggregate density $\lambda(l)$. In particular, "log-based" invariants defined directly in the log-price or log-sqrt-price domain — rather than via an underlying $(x,y)$ polynomial — are a natural direction for further research and can be incorporated into the same framework by specifying their associated price map and differentials.

### 5.5 Segment-wise CLMM Execution on the Local Mesh

We now plug the **segment liquidity** $L_{AMM,k}$ from Layer 2 (Section 5.3) into the above CLMM differential (Section 5.4) to obtain explicit **per-segment execution math**.

Recall from the local mesh construction:

- We have an ordered mesh:

$$
\mathcal{M} = \{ l_0 < l_1 < \ldots < l_{N} \}
$$

- The price path interval is partitioned as:

$$
I_{path} = \bigcup_{k=0}^{N-1} [l_k, l_{k+1}]
$$

- On each segment $[l_k, l_{k+1}]$, we have defined an effective liquidity $L_{AMM,k} > 0$, intended to approximate (and in our piecewise-constant case, equal) the continuous $\lambda(l)$ on that segment.

Under the usual construction (all band boundaries included in $\mathcal{M}$), $\lambda(l)$ is constant on each open segment $(l_k, l_{k+1})$, so we may choose:

$$
L_{AMM,k} = \lambda(\bar{l}_k), \quad \bar{l}_k \in (l_k, l_{k+1})
$$

and this gives **exact** equality $\lambda(l) = L_{AMM,k}$ for all $l \in (l_k, l_{k+1})$. Thus, per segment we can use the constant-liquidity formulas with $L = L_{AMM,k}$.

#### 5.5.1 Per-segment CLMM Deltas in $l$-space

Fix a segment index $k \in \{0, \ldots, N-1\}$. On the segment $[l_k, l_{k+1}]$, define:

- $L_k := L_{AMM,k}$ (for brevity),
- $l_{min} = l_k$,
- $l_{max} = l_{k+1}$.

For any two log-sqrt-prices $l_a, l_b \in [l_{min}, l_{max}]$, we define the **segment CLMM deltas**:

$$
\Delta A_k (l_a \to l_b) = L_k (2^{-l_b} - 2^{-l_a}), \quad \Delta B_k (l_a \to l_b) = L_k (2^{l_b} - 2^{l_a})
$$

These are exactly the finite formulas from before, applied with liquidity $L = L_k$.

Remarks:

- If $l_b > l_a$, then:
  - $2^{l_b} > 2^{l_a}$, so $\Delta B_k > 0$
  - $2^{-l_b} < 2^{-l_a}$, so $\Delta A_k < 0$.
- If $l_b < l_a$, the signs are reversed.

These signs encode the direction of reserves flow for the AMM; the trader's deltas are $-\Delta A_k$ and $-\Delta B_k$.

If we prefer to express these in terms of sqrt-prices, let:

$$
S_a = 2^{l_a}, \quad S_b = 2^{l_b}
$$

Then:

$$
2^{-l_a} = \frac{1}{S_a}, \quad 2^{-l_b} = \frac{1}{S_b}
$$

and we recover:

$$
\Delta A_k (l_a \to l_b) = L_k \left( \frac{1}{S_b} - \frac{1}{S_a} \right), \quad \Delta B_k (l_a \to l_b) = L_k (S_b - S_a)
$$

with the understanding that $S_a, S_b \in [2^{l_k}, 2^{l_{k+1}}]$.

#### 5.5.2 Per-segment Capacity

For a given segment $k$, we can define the **maximum CLMM capacity** of that segment in terms of token flows if the price traverses the **full segment**.

Assume we move from the lower node $l_k$ to the upper node $l_{k+1}$. Then:

$$
\text{Full-segment deltas:} \Delta A_k^{max, \uparrow} = L_k (2^{-l_{k+1}} - 2^{-l_k}), \quad \Delta B_k^{max, \uparrow} = L_k (2^{l_{k+1}} - 2^{l_k})
$$

Similarly, moving from the upper node down to the lower:

$$
\Delta A_k^{max, \downarrow} = L_k (2^{-l_k} - 2^{-l_{k+1}}) = -\Delta A_k^{max, \uparrow}, \quad \Delta B_k^{max, \downarrow} = L_k (2^{l_k} - 2^{l_{k+1}}) = -\Delta B_k^{max, \uparrow}
$$

In absolute value, the segment provides:

$$
\left| \Delta A_k^{max} \right| = L_k \left| 2^{-l_{k+1}} - 2^{-l_k} \right|, \quad \left| \Delta B_k^{max} \right| = L_k \left| 2^{l_{k+1}} - 2^{l_k} \right|
$$

These quantities are exactly what you need for:

- deciding whether the trade can traverse the entire segment,
- or whether it will **terminate inside** the segment.

#### 5.5.3 Partial Usage of a Segment: Solving for $l_{end}$

In practice, while executing a swap, you often know:

- current log-price $l_{curr}$,
- segment liquidity $L_k$,
- remaining amount of one token to trade (e.g. an amount of token $A$ in or an amount of token $B$ out),

and you want to determine:

- the **new log-price** $l_{next}$ **within the same segment** that corresponds to consuming exactly that amount, provided it lies inside the segment bounds.

We now write the **inverse formulas** for $l_{next}$ given $\Delta A$ or $\Delta B$.

Let $l_{curr} \in [l_k, l_{k+1}]$ be the starting point in segment $k$, and let $\Delta A, \Delta B$ denote the desired **pool-level** deltas on that segment (sign included).

##### 5.5.3.1 Solving from $\Delta B$

From the per-segment formula:

$$
\Delta B = \Delta B_k (l_{curr} \to l_{next}) = L_k (2^{l_{next}} - 2^{l_{curr}})
$$

Rearrange to solve for $2^{l_{next}}$:

$$
2^{l_{next}} = 2^{l_{curr}} + \frac{\Delta B}{L_k}
$$

Assuming the right-hand side is positive (which is necessary since $2^l >0$), we get:

$$
l_{next} = \log_2 \left( 2^{l_{curr}} + \frac{\Delta B}{L_k} \right)
$$

To remain within the segment, we require:

$$
l_{min} \leq l_{next} \leq l_{max} \quad (\text{i.e. } l_k \leq l_{next} \leq l_{k+1})
$$

If the computed $l_{next}$ lies outside $[l_k, l_{k+1}]$, it means the requested $\Delta B$ exceeds the segment's capacity in that direction; in that case, the actual move will stop at the boundary $l_k$ or $l_{k+1}$, and the segment is fully consumed.

##### 5.5.3.2 Solving from $\Delta A$

From the per-segment formula:

$$
\Delta A = \Delta A_k (l_{curr} \to l_{next}) = L_k (2^{-l_{next}} - 2^{-l_{curr}})
$$

Rearrange to solve for $2^{-l_{next}}$:

$$
2^{-l_{next}} = 2^{-l_{curr}} + \frac{\Delta A}{L_k}
$$

We require the right-hand side to be positive (since $2^{-l} > 0$):

$$
2^{-l_{curr}} + \frac{\Delta A}{L_k} > 0
$$

Then:

$$
l_{next} = -\log_2 \left( 2^{-l_{curr}} + \frac{\Delta A}{L_k} \right)
$$

Again, to remain within the segment:

$$
l_{k} \leq l_{next} \leq l_{k+1}
$$

If the computed $l_{next}$ lies outside this interval, the requested $\Delta A$ cannot be fully realized in this segment; the execution moves to the boundary and the segment is fully consumed.

#### 5.5.4 Composition Across Segments

For a complete trade along the price path:

1. Start at initial node $l_{ast} = l_{k_{ast}}$ (or possibly an interior point in some segment)

2. For each step:

   - Compute full-segment capacities $\Delta A_k^{max}, \Delta B_k^{max}$ on the next segment in the direction of price movement.
   - Compare remaining trade amount with this capacity.
   - Either:
     - Consume the entire segment and move to the next node, or
     - Use the inverse formulas above to find $l_{final}$ inside the segment and stop.

3. At each node $l_k$, interleave CLMM segment execution with CLOB matching at that node, as described in the previous sections.

Mathematically, the total CLMM deltas along the path are the **sum of per-segment contributions**:

If the actual price path visits nodes $l_{k_0} = l_*, l_{k_1}, \ldots, l_{k_m} = l_{final}$ in order, then:

$$
\Delta A_{total} = \sum_{r=0}^{m-1} \Delta A_{k_r} (l_{k_r} \to l_{k_{r+1}}), \quad \Delta B_{total} = \sum_{r=0}^{m-1} \Delta B_{k_r} (l_{k_r} \to l_{k_{r+1}})
$$

with the last segment possibly ending at an interior point of $[l_{k_{m}}, l_{k_m+1}]$. This is exactly the piecewise integral of the overall differential system:

$$
dA(l) = - \lambda(l) (\ln 2) \cdot 2^{-l} \ dl, \quad dB(l) = \lambda(l) (\ln 2) \cdot 2^{l} \ dl
$$

approximated (and, with our choice of mesh, **exactly realized**) by segment-wise constant liquidity $L_{AMM,k}$.

**Figure 7: Segment-wise CLMM Execution Flowchart**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SEGMENT-WISE EXECUTION ALGORITHM                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                        ┌─────────────────────┐                              │
│                        │   START at node lₖ  │                              │
│                        │   Remaining: Δq_rem │                              │
│                        └──────────┬──────────┘                              │
│                                   │                                         │
│                                   ▼                                         │
│                    ┌──────────────────────────────┐                         │
│                    │  CLOB orders at node lₖ?     │                         │
│                    │  (opposing side to trade)    │                         │
│                    └──────────────┬───────────────┘                         │
│                                   │                                         │
│                      ┌────────────┴────────────┐                            │
│                      │ YES                     │ NO                         │
│                      ▼                         ▼                            │
│         ┌────────────────────────┐   ┌─────────────────────┐               │
│         │ Match CLOB orders at   │   │                     │               │
│         │ price P(lₖ)            │   │                     │               │
│         │ Update: Δq_rem -= Δq   │   │                     │               │
│         └───────────┬────────────┘   │                     │               │
│                     │                │                     │               │
│                     └────────────────┴─────────┐           │               │
│                                                │           │               │
│                                                ▼           │               │
│                              ┌───────────────────────────┐ │               │
│                              │   Δq_rem = 0 ?            │ │               │
│                              └─────────────┬─────────────┘ │               │
│                                            │               │               │
│                               ┌────────────┴───────────┐   │               │
│                               │ YES                    │ NO│               │
│                               ▼                        ▼   │               │
│              ┌─────────────────────┐    ┌──────────────────┴──────────┐    │
│              │       DONE          │    │  Enter CLMM segment         │    │
│              │  Trade complete at  │    │  [lₖ, lₖ₊₁] with L_AMM,k   │    │
│              │  final price P(l)   │    └──────────────┬──────────────┘    │
│              └─────────────────────┘                   │                   │
│                                                        ▼                   │
│                              ┌───────────────────────────────────────┐     │
│                              │  Compute segment capacity:            │     │
│                              │  ΔBₖᵐᵃˣ = L_AMM,k · (2^lₖ₊₁ - 2^lₖ)   │     │
│                              └───────────────────┬───────────────────┘     │
│                                                  │                         │
│                                                  ▼                         │
│                              ┌───────────────────────────────────────┐     │
│                              │  |Δq_rem| ≤ |ΔBₖᵐᵃˣ| ?               │     │
│                              └───────────────────┬───────────────────┘     │
│                                                  │                         │
│                                ┌─────────────────┴──────────────┐          │
│                                │ YES                            │ NO       │
│                                ▼                                ▼          │
│              ┌─────────────────────────────┐  ┌────────────────────────┐   │
│              │  Solve for l_final:         │  │  Consume full segment  │   │
│              │  2^l_final = 2^lₖ +Δq_rem/L │  │  Δq_rem -= ΔBₖᵐᵃˣ      │   │
│              │  Δq_rem = 0                 │  │  Move to node lₖ₊₁     │   │
│              │                             │  │  k = k + 1             │   │
│              └─────────────┬───────────────┘  └───────────┬────────────┘   │
│                            │                              │                │
│                            ▼                              │                │
│              ┌─────────────────────────┐                  │                │
│              │         DONE            │                  │                │
│              │  Final price: P(l_final)│                  │                │
│              └─────────────────────────┘                  │                │
│                                                           │                │
│                              ┌────────────────────────────┘                │
│                              │                                             │
│                              └─────────────► (loop back to CLOB check)     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

_Figure 7: The segment-wise execution algorithm processes trades by alternating between CLOB matching at mesh nodes and CLMM traversal through segments. At each node lₖ, opposing CLOB orders are matched first at price P(lₖ). If trade remains, the algorithm enters the adjacent CLMM segment, either consuming it fully (moving to the next node) or partially (solving for the final interior price). This continues until the trade is complete._

<!-- LaTeX/TikZ version for PDF:
\begin{figure}[h]
\centering
\begin{tikzpicture}[node distance=1.5cm]
  % Flowchart nodes with decision diamonds and process rectangles
  % Arrows showing flow with YES/NO labels
  % Loop back arrow for segment iteration
\end{tikzpicture}
\caption{Segment-wise CLMM Execution Flowchart}
\label{fig:segment-execution-flowchart}
\end{figure}
-->

### 5.6 Relationship Between Layers

We summarize the roles and relationships of the three layers formally.

#### 5.6.1 Layer 0 $\to$ Layer 1 (Projection to Slots)

Given:

- A set of CLMM bands $\{(l_{lower,i}, l_{upper,i}, L_i)\}_{i \in I}$,
- A set of orders $\{(l_{0,j}, q_j)\}_{j \in J}$,
- Slot parameters $(l_{ref}, \Delta l)$,

Layer 1 defines the maps:

- **LP band slot projection**:

$$
\Pi_{LP}: I \to \mathbb{Z} \times \mathbb{Z}, \quad \Pi_{LP}(i) = (s_{lower,i}, s_{upper,i})
$$

- **Order slot projection**:

$$
\Pi_{ORD}: J \to \mathbb{Z}, \quad \Pi_{ORD}(j) = s_j
$$

These maps are **many-to-one** and lossless with respect to the economic parameters (we will store the original $l$-values for each object; the slots are auxiliary indices).

Given a current log-price $l_*$, we can find a "relevant" set of LPs and orders by:

- Computing the current slot $s_* = s_{\text{round}}(l_*)$,
- Searching in a window $[s_* - R, s_* + R]$ for some small radius $R$, and then filtering by the exact $l$-conditions.

#### 5.6.2 Layer 0 + Layer 1 $\to$ Layer 2 (Local Mesh Construction)

Given:

- The continuous data $(\lambda, \mu)$ from Layer 0 (Section 5.1),
- The current log-price $l_*$ and target $l_{target}$,
- The sets of LP bands and orders intersecting $I_{path}$, which we can efficiently discover via Layer 1 slots (Section 5.2),

Layer 2 (Section 5.3) constructs a finite mesh $\mathcal{M}$ by:

1. Gathering all LP boundaries and order prices in $I_{path}$,
2. Adding path endpoints,
3. Adding optional refinement points based on desired numerical criteria,
4. Sorting and deduplicating to obtain {$l_0 < l_1 < \ldots < l_N$}.

From this mesh and the Layer-0 liquidity/order data, we define:

- Segment-wise AMM liquidity $L_{AMM,k}$,
- Node-based depth $Q_{CLOB,k}$.

which fully specify the **local execution environment** for the trade.

#### 5.6.3 Persistence vs Transience

- **Layer 0 (Section 5.1)**:

  - Logical state: band bounds $(l_{lower,i}, l_{upper,i})$, order prices $l_{0,j}$, and liquidity/quantities $L_i, q_j$.
  - Conceptually persistent (though on-chain it is encoded in fixed-point, Section 4).

- **Layer 1 (Section 5.2)**:

  - Persistent on-chain indexing:
    - slot index ranges for LPs,
    - slot index for orders.
  - Used to answer queries like "which bands and orders are near the current price?"

- **Layer 2 (Section 5.3)**:

  - Constructed per trade / per execution.
  - Depends on:
    - current price and target,
    - discovered LPs and orders from nearby slots,
    - optional refinement rules.
  - Destroyed after the trade completes; no state is persisted except updated balances, positions, orders back in Layer 0 + Layer 1 structures.

### 5.7 Conclusion

This three-layer architecture yields:

- A **continuous, unified economic model** over $l \in \mathbb{R}$ (Layer 0, Section 5.1),
- **Efficient indexing and storage** via global slots (Layer 1, Section 5.2),
- **Adaptive, high-precision execution** via local meshes (Layer 2, Section 5.3).

while keeping prices fundamentally log-based (no global-ticks, Section 3) and allowing CLMM, CLOB, and perps to all coexist on the same axis with consistent semantics.

## 6. Formal Fixed-Point Specification

This section specifies the precise relationship between the **continuous** economic model in log-sqrt-price coordinates (Section 5) and the **discrete on-chain representation** in fixed-point integer arithmetic (Section 4).
At a high level, we show that there is a single global constant $C$ (depending only on liquidity and price bounds, not on implementation details) such that, for any trade along a fixed price path, the total numerical error in the token deltas is bounded by

$$
  C \Big( \tfrac{1}{\sigma} + \varepsilon_{\exp_2} + \varepsilon_{\log_2} + \varepsilon_{\mathrm{arith}} \Big),
$$

where $\tfrac{1}{\sigma}$ is the log-encoding granularity, $\varepsilon_{\exp_2}$ and $\varepsilon_{\log_2}$ control the accuracy of the $exp_2$ and $log_2$ routines, and $\varepsilon_{\mathrm{arith}}$ captures cumulative fixed-point rounding error. In particular, by increasing the number of fractional bits and tightening the exp/log approximations, the on-chain execution converges uniformly to the continuous model, independently of Layer-1 slot choices or Layer-2 mesh heuristics (as long as band boundaries and order prices are included as mesh nodes).

More concretely, we introduce:

- a fixed-point encoding of the log-sqrt-price $l$,
- fixed-point formats for sqrt-price $S$ and price $P$, and liquidity $Q$,
- abstract specifications of the numerical $exp_2$ and $log_2$ routines,
- and error bounds showing that on-chain execution converges to the continuous model.

Throughout this section, we assume all arithmetic is carried out in a fixed-width integer domain (e.g. 256-bit signed/unsigned integers). We model overflow at the specification level by imposing explicit bounds on representable values and treating out-of-range states as invalid, following standard numerical analysis methodology [@higham2002accuracy].

### 6.1 Fixed-Point Encoding of Log-Sqrt-Price

#### 6.1.1 Integer domain and Scaling

Let:

- $F \in \mathbb{N}$ be the number of fractional log bits,
- $\sigma = 2^F > 0$ be the log scaling factor,
- $w \in \mathbb{N}$ be the bit-width of the signed integer type used for log-coordinates (e.g. (w = 256)).

Define the integer domain:

$$
\mathbb{Z}_w = \{ z \in \mathbb{Z} :  -2^{w-1} \leq z \leq 2^{w-1} - 1 \}
$$

The log-sqrt-price coordinate $l \in \mathbb{R}$ is encoded as:

$$
L \in \mathbb{Z}_w, \quad l \approx \frac{L}{\sigma}
$$

We interpret:

- $L$ as the **on-chain representation**
- $l$ as the **ideal continuous log-sqrt-price**.

#### 6.1.2 Rounding and Encoding Map

We adopt **round-to-nearest with ties to even** as the canonical rounding rule. Formally, define:

$$
\mathrm{rnd} : \mathbb{R} \to \mathbb{Z}, \quad \mathrm{rnd}(x) = \begin{cases}
\lfloor x \rfloor & \text{if } x - \lfloor x \rfloor < \frac{1}{2} \\
\lceil x \rceil & \text{if } x - \lfloor x \rfloor > \frac{1}{2} \\
\text {nearest even integer} & \text{if } x - \lfloor x \rfloor = \frac{1}{2}
\end{cases}
$$

This satisfies the usual property:

$$
|\mathrm{rnd}(x) - x| \leq \frac{1}{2}, \quad \forall x \in \mathbb{R}
$$

We define the **log-sqrt-price encoding** and **decoding** maps as:

- **Encode (continuous $\to$ integer)**:

$$
\mathrm{enc}_l : \mathbb{R} \to \mathbb{Z}_w, \quad \mathrm{enc}_l(l) = \begin{cases}
\mathrm{clip_{[L_{min}, L_{max}]}} \big (\mathrm{rnd}(l \cdot \sigma) \big), & \text{if not overflow} \\
\text{undefined (revert)}, & \text{otherwise}
\end{cases}
$$

where $\mathrm{clip_{[a,b]}}(x) = \min(\max(x,a),b)$ and $L_{min}, L_{max} \in \mathbb{Z}_w$ are protocol-chosen bounds corresponding to the minimum and maximum supported prices.

- **Decode (integer $\to$ approximate log value)**:

$$
\mathrm{dec}_l : \mathbb{Z}_w \to \mathbb{R}, \quad \mathrm{dec}_l(L) = \frac{L}{\sigma}
$$

In the economic model, we treat any state that would require clipping as **invalid** (i.e. trades that attempt to move the price outside the representable range are rejected). Thus, when we write $L = \mathrm{enc}_l(l)$, we _implicitly assume_ $L_{min} \le \mathrm{rnd}(l \cdot \sigma) \le L_{max}$.

#### 6.1.3 Encoding error and induced price error

Let $l^* \in \mathbb{R}$ be a "true" log-sqrt-price, and suppose the encoding does not overflow:

$$
L^* = \mathrm{enc}_l(l^*), \quad \hat{l} = \mathrm{dec}_l(L) = \frac{L}{\sigma}
$$

Set the encoding error:

$$
\delta l := \hat{l} - l^*
$$

Then by the rounding property:

$$
|\delta l| = \left| \frac{\mathrm{rnd}(l^* \cdot \sigma)}{\sigma} - l^* \right| = \frac{|\mathrm{rnd}(l^* \cdot \sigma) - l^* \cdot \sigma|}{\sigma} \leq \frac{1}{2\sigma}
$$

The corresponding true and decoded prices are:

$$
S^* = 2^{l^*}, \quad \hat{S} = 2^{\hat{l}} = 2^{l^* + \delta l} = S^* \cdot 2^{\delta l}
$$

$$
P^* = (S^*)^2 = 2^{2l^*}, \quad \hat{P} = 2^{2\hat{l}} = 2^{2l^* + 2\delta l} = P^* \cdot 2^{2\delta l}
$$

Thus the **relative errors** satisfy:

$$
\frac{\hat{S}}{S^*} = 2^{\delta l} \in \Big[ 2^{-\frac{1}{2\sigma}}, 2^{\frac{1}{2\sigma}} \Big]
$$

$$
\frac{\hat{P}}{P^*} = 2^{2\delta l} \in \Big[ 2^{-\frac{1}{\sigma}}, 2^{\frac{1}{\sigma}} \Big]
$$

For large $\sigma$ (many fractional bits), we can use the bound $|x| \leq \frac{1}{\sigma}$ and the inequality $|2^x - 1| \leq (\ln 2) |x| 2^{|x|}$ to obtain:

**Lemma 6.1 (Encoding-induced Price Error).**

Assume $\sigma \geq 1$ and no overflow. Then:

$$
\left| \frac{\hat{P}}{P^*} - 1 \right| = \left| 2^{2\delta l} - 1 \right| \leq C \cdot \frac{\ln 2}{\sigma}
$$

for some absolute constant $C \in [1, 2]$. In particular, for $\sigma \geq 32$, we have:

$$
\left| \frac{\hat{P}}{P^*} - 1 \right| \lesssim \frac{\ln 2}{\sigma}
$$

**_Proof._**

Using $|\delta l| \leq \frac{1}{2\sigma}$, we have $|2\delta l| \leq \frac{1}{\sigma}$. By the mean value theorem applied to $f(x) = 2^x$ on [$\frac{-1}{\sigma}, \frac{1}{\sigma}$],

$$
|2^{2\delta l} - 1| \leq \max_{x \in [-\frac{1}{\sigma}, \frac{1}{\sigma}]} |f'(x)| \cdot |2\delta l| = \max_{x \in [-\frac{1}{\sigma}, \frac{1}{\sigma}]} (\ln 2) 2^x \cdot |2\delta l| \leq (\ln 2) 2^{\frac{1}{\sigma}} \cdot \frac{1}{\sigma}
$$

Taking $C = 2^{\frac{1}{\sigma}} \in [1,2]$ for all $\sigma \geq 1$ yields the bound. $\blacksquare$

This shows that increasing $\sigma$ (the number of fractional bits in $L$) reduces the **pure encoding error** of prices at rate $O\left(\frac{1}{\sigma}\right)$.

### 6.2 Fixed-Point Formats for Sqrt-Price, Price, and Liquidity

We now fix the explicit integer formats for the main quantities used in CLMM execution.

#### 6.2.1 Q-Formats

A Q-format $Q_{I.F}$ [@yates2013fixed] is a fixed-point representation of real numbers of the form:

$$
x = \frac{X}{2^F}, \quad X \in \mathbb{Z}
$$

where:

- $F \ge 0$ is the number of **fractional bits**,
- $I$ is the number of **integer bits** so that $|X| \leq 2^{I+F} - 1$ in a signed representation.

We denote the raw integer by $X$ and write:

$$
x \equiv \frac{X}{2^F} \quad \text{Q}_{I.F}
$$

We assume the implementation uses sufficiently large $I$ (e.g. $I+F \leq 256$) so that overflow is impossible within the supported economic range. As in Section 6.1.2, we treat any operation that would overflow as invalid in the economic model.

#### 6.2.2 Fixed-point Domains

We choose the following canonical formats:

- Log-sqrt-price:

$$
L \in \mathbb{Z}_w, \quad l \approx \frac{L}{\sigma}, \quad \sigma = 2^F
$$

- Sqrt-price:

$$
S_{\mathrm{fp}} \in \mathbb{Z}, \quad S \approx \frac{S_{\mathrm{fp}}}{2^{Q_S}}, \quad \text{Q}_{I_S.Q_S}
$$

- Price:

$$
P_{\mathrm{fp}} \in \mathbb{Z}, \quad P \approx \frac{P_{\mathrm{fp}}}{2^{Q_P}}, \quad \text{Q}_{I_P.Q_P}
$$

Typically, $P$ will be derived as $P \approx S^2$, so we require

$$
Q_P \geq 2 Q_S, \quad I_P \ge 2 I_S + 1
$$

to accommodate squaring and scaling with bounded rounding error.

- Liquidity:

$$
Q_{\mathrm{fp}} \in \mathbb{Z}, \quad Q \approx \frac{Q_{\mathrm{fp}}}{2^{Q_Q}}, \quad \text{Q}_{I_Q.Q_Q}
$$

All CLMM formulas are implemented by integer operations on these fixed-point domains, followed by appropriate shifts and rounding.

### 6.3 Fixed-Point Exponential: $\operatorname{exp_2fp}$

We now formalize the on-chain computation of the square-root price from the log-sqrt integer $L$.

#### 6.3.1 Abstract Specification

Define:

$$
\operatorname{exp_2fp} : \mathbb{Z}_w \to \mathbb{Z}
$$

to be a function that, for each admissible $L$, returns a fixed-point approximation $S_{\mathrm{fp}} = \operatorname{exp_2fp}(L)$ with the interpretation:

$$
\hat{S}(L) := \frac{S_{\mathrm{fp}}}{2^{Q_S}}
$$

as an approximation to the true value:

$$
S(L) = 2^{\frac{L}{\sigma}}
$$

We require two properties:

1. **Monotonicity:** For all $L_1 < L_2$ in the domain,

   $$
   \operatorname{exp_2fp}(L_1) < \operatorname{exp_2fp}(L_2)
   $$

2. **Relative Error Bound:** There exists a protocol-chosen constant $\varepsilon_{\exp_2} > 0$ such that for all $L$ in the domain,

$$
\left| \frac{\hat{S}(L)}{S(L)} - 1 \right| \leq \varepsilon_{\exp_2}
$$

Equivalently, in absolute terms:

$$
|\hat{S}(L) - S(L)| \leq \varepsilon_{\exp_2} \cdot S(L) \quad \forall L
$$

The specific implementation (polynomial, table-based, etc.) is not fixed in the economic model; only these properties are required.

#### 6.3.2 Example construction (range reduction)

One canonical construction (not normative but illustrative) follows the standard range reduction approach for exponential functions [@muller2016elementary; @tang1989exponential]:

1. Decompose:

   $$
   \frac{L}{\sigma} = q + f, \quad q \in \mathbb{Z}, \quad f \in [0,1)
   $$

   by setting:

   $$
   q = \left\lfloor \frac{L}{\sigma} \right\rfloor, \quad r = L - q\sigma \in {0, 1, \ldots, \sigma - 1}, \quad f = \frac{r}{\sigma}
   $$

2. Note:

   $$
   S(L) = 2^{\frac{L}{\sigma}} = 2^{q + f} = 2^q \cdot 2^f
   $$

3. Implement:

- $2^q$ by exponent shifting in fixed-point: if $S_0 = 2^{Q_S}$ represents 1.0, then $2^q$ is represented by $S_{q,\mathrm{fp}} = S_0 \ll q$ (left shift) for $q \geq 0$ , or $\gg |q|$ if $q < 0$, subject to range checks.
- $2^f$ by a polynomial or table approximation on the interval $f \in [0,1)$.

That is, define an approximation $\widetilde{E}(f)$ to $2^f$ such that:

$$
\left| \frac{\widetilde{E}(f)}{2^f} - 1 \right| \leq \varepsilon_{\mathrm{frac}}
$$

and then set:

$$
S_{\mathrm{fp}} = \left \lfloor S_{q,\mathrm{fp}} \cdot \frac{\widetilde{E}(f)}{2^{Q_S}} \right \rceil
$$

where $\lfloor \cdot \rceil$ denotes rounding to nearest integer.

Provided the rounding error is controlled, the composite error $\varepsilon_{\exp_2}$ can be bounded in terms of $\varepsilon_{\mathrm{frac}}$ and the fixed-point rounding error. We do not fix $\widetilde{E}$ at the specification level, but require that the resulting $\operatorname{exp_2fp}$ satisfies the monotonicity and relative error properties above.

#### 6.3.3 Error propagation to Price

Given $\hat{S}(L)$ with relative error $\varepsilon_{\exp_2}$, the approximate price is:

$$
\hat{P}(L) = \hat{S}(L)^2, \quad P(L) = S(L)^2 = 2^{\frac{2L}{\sigma}}
$$

Then:

$$
\frac{\hat{P}(L)}{P(L)} = \left( \frac{\hat{S}(L)}{S(L)} \right)^2
$$

Let:

$$
\delta_S (L) := \frac{\hat{S}(L)}{S(L)} - 1
$$

so that:

$$
\left| \frac{\hat{P}(L)}{P(L)} - 1 \right| = \left| \left( \frac{\hat{S}(L)}{S(L)} \right)^2 - 1 \right| = \left| \left( 1 + \delta_S \right)^2 - 1 \right| = |2\delta_S (L) + \delta_S (L)^2|
$$

where $|\delta_S (L)| \leq \varepsilon_{\exp_2}$. Thus:

**Lemma 6.2 ($exp_2$-induced price error).**

If $\operatorname{exp_2fp}$ satisfies the relative error bound $|\delta_S (L)| \leq \varepsilon_{\exp_2}$, then for all admissible $L$,

$$
\left| \frac{\hat{P}(L)}{P(L)} - 1 \right| \leq 2 \varepsilon_{\exp_2} + \varepsilon_{\exp_2}^2
$$

In particular, for $\varepsilon_{\exp_2} \leq 10^{-4}$, the price error satisfies:

$$
\left| \frac{\hat{P}(L)}{P(L)} - 1 \right| \leq 2.0001 \varepsilon_{\exp_2}
$$

**_Proof._** Immediate from the previous derivation. $\blacksquare$

### 6.4 Fixed-Point Logarithm: $\operatorname{log_2fp}$

We also require a numerical $log_2$ routine to reconstruct log-sqrt-price from a fixed-point representation of $S$ (e.g. for oracle integration, position management, or diagnostics).

#### 6.4.1 Abstract Specification

Let $S_{\mathrm{fp}} \in \mathbb{Z}$ be a fixed-point sqrt-price with interpretation:

$$
S = \frac{S_{\mathrm{fp}}}{2^{Q_S}} > 0
$$

We define:

$$
\operatorname{log_2fp} : \mathbb{Z}_{>0} \to \mathbb{Z}_w
$$

such that the decoded log-sqrt-price:

$$
\hat{L} := \frac{\operatorname{log_2fp}(S_{\mathrm{fp}})}{\sigma}
$$

approximates $\log_2 S$.

We require:

1. **Domain and monotonicity:** For all $S_{\mathrm{fp},1} < S_{\mathrm{fp},2}$,

   $$
   \operatorname{log_2fp}(S_{\mathrm{fp},1}) < \operatorname{log_2fp}(S_{\mathrm{fp},2})
   $$

   and the function is defined only for $S_{\mathrm{fp}} > 0$.

2. **Absolute log error bound:** There exists $\varepsilon_{\log_2} > 0$ such that for all admissible $S_{\mathrm{fp}} > 0$,

$$
\left| \hat{L} - \log_2 S \right| \leq \varepsilon_{\log_2}
$$

Equivalently:

$$
\left| \frac{\operatorname{log_2fp}(S_{\mathrm{fp}})}{\sigma} - \log_2 \left( \frac{S_{\mathrm{fp}}}{2^{Q_S}} \right) \right| \leq \varepsilon_{\log_2}
$$

Again, we leave the implementation (e.g. range reduction + polynomial or table) unspecified, requiring only these properties.

#### 6.4.2 Example construction (normalization)

A canonical normalized implementation follows the standard approach for logarithm computation [@muller2016elementary; @tang1990logarithm]:

1. Find the integer exponent $e$ such that:

   $$
   S_{\mathrm{fp}} = m \cdot 2^e, \quad m \in [2^{Q_S}, 2^{Q_S + 1})
   $$

   i.e. normalize $S$ into $[1, 2)$.

2. Write:

   $$
   S = \frac{S_{\mathrm{fp}}}{2^{Q_S}} = 2^{e-Q_S} \cdot \frac{m}{2^{Q_S}} = 2^{e - Q_S} \cdot (1 + \delta)
   $$

   where $\delta \in [0, 1)$.

3. Then:

   $$
   \log_2 S = (e - Q_S) + \log_2 (1 + \delta)
   $$

4. Approximate $\log_2 (1 + \delta)$ on $\delta \in [0,1)$ by a polynomial or LUT $\widetilde{L}(\delta)$ with bounded error, and define:

$$
\hat{l} = (e - Q_S) + \widetilde{L}(\delta)
$$

Implementation details determine $\varepsilon_{\log_2}$; at the economic model level we only require that $\varepsilon_{\log_2}$ be sufficiently small and known.

### 6.5 Monotonicity and Safety

The monotonicity of $\operatorname{exp_2fp}$ and $\operatorname{log_2fp}$ is crucial to avoid non-physical artifacts such as "backwards" price moves or non-monotone reconstruction of log-prices.

**Proposition 6.3 (Monotone price map).**

Suppose $\operatorname{exp_2fp}$ is strictly increasing in $L$. Then the decoded sqrt-price $\hat{S}(L)$ and price $\hat{P}(L)$ are strictly increasing functions of $L$.

**_Proof._**

If $L_1 < L_2$, then $\operatorname{exp_2fp}(L_1) < \operatorname{exp_2fp}(L_2)$. Since the decoding $S = \frac{S_{\mathrm{fp}}}{2^{Q_S}}$ is strictly increasing in $S_{\mathrm{fp}}$, it follows that $\hat{S}(L_1) < \hat{S}(L_2)$. Squaring preserves strict monotonicity for positive arguments, so $\hat{P}(L_1) < \hat{P}(L_2)$. $\square$

**Proposition 6.4 (Monotone log reconstruction).**

Suppose $\operatorname{log_2fp}$ is strictly increasing in $S_{\mathrm{fp}}$. Then the reconstructed log-sqrt-price $\hat{L}(S_{\mathrm{fp}})$ is strictly increasing in $S_{\mathrm{fp}}$.

**_Proof._**

If $S_{\mathrm{fp},1} < S_{\mathrm{fp},2}$, then $\operatorname{log_2fp}(S_{\mathrm{fp},1}) < \operatorname{log_2fp}(S_{\mathrm{fp},2})$. Dividing by the positive constant $\sigma$ preserves strict inequality, so $\hat{L}(S_{\mathrm{fp},1}) < \hat{L}(S_{\mathrm{fp},2})$. $\square$

These properties guarantee that:

- as the on-chain $L$ increases (due to trades pushing the price up), the decoded price $\hat{P}$ also increases,
- reconstruction $L$ from $S_{\mathrm{fp}}$ cannot produce reversed ordering.

Thus the numerical layer preserves the qualitative ordering structure of the continuous model.

### 6.6 Aggregate Error Bounds and Convergence

We now sketch how the various numerical errors (encoding, $exp_2$, $log_2$, fixed-point arithmetic, and mesh discretization) combine, and show that they can be made arbitrarily small by tuning parameters. The bounds in this subsection are **global** in the sense that they hold uniformly over all trades along a fixed price path, and they are **architecture-stable** in the sense that they do not depend on the particular choice of Layer-1 slot spacing or Layer-2 mesh refinement strategy, provided all band boundaries and order prices are present as mesh nodes.

#### 6.6.1 Error Sources

We distinguish the following sources:

1. **Encoding error** from continuous $l^*$ to integer $L$ (Section 6.1.3).
2. **Exponential approximation error** from $\operatorname{exp_2fp}$ (Section 6.3.3).
3. **Logarithm approximation error** from $\operatorname{log_2fp}$ (Section 6.4.1).
4. **Fixed-point arithmetic error** in addition, multiplication, division, and rounding in Q-formats.
5. **Mesh approximation error** in Layer 2 (discretizing $\lambda(l)$).

The first three are per-point errors; the fourth accumulates across operations; the fifth is geometric (discretization in $l$-space).

**Figure 8: Error Propagation Chain in Fixed-Point Execution**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ERROR PROPAGATION THROUGH THE SYSTEM                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CONTINUOUS MODEL (exact, infinite precision)                               │
│  ════════════════════════════════════════════                               │
│                                                                             │
│      l* ────────► S* = 2^l* ────────► ΔB* = L·(S*_b - S*_a)                │
│      │            │                   │                                     │
│      │ true       │ true              │ true result                         │
│      │ log-price  │ sqrt-price        │                                     │
│      │            │                   │                                     │
│      ▼            ▼                   ▼                                     │
│  ════════════════════════════════════════════════════════════════           │
│  FIXED-POINT IMPLEMENTATION (approximate, finite precision)                 │
│  ════════════════════════════════════════════════════════════════           │
│      │            │                   │                                     │
│      ▼            ▼                   ▼                                     │
│  ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐            │
│  │ENCODING│──►│  EXP2  │──►│  MULT  │──►│  SUB   │──►│ RESULT │            │
│  │  l→L   │   │  L→S   │   │  L·S   │   │ ΔB_k   │   │   ΔB   │            │
│  └───┬────┘   └───┬────┘   └───┬────┘   └───┬────┘   └───┬────┘            │
│      │            │            │            │            │                  │
│      ▼            ▼            ▼            ▼            ▼                  │
│   ε_enc       ε_exp2       ε_mult       ε_sub        ε_total               │
│   ≤1/(2σ)    ≤ε_exp2·S    ≤δ_op        ≤δ_op                               │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ERROR BOUND COMPOSITION:                                                   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │   ε_total  ≤  C · ( 1/(2σ) + ε_exp2 + ε_log2 + ε_arith )           │   │
│  │                  \_______/   \____________________/                 │   │
│  │                  encoding     function approximation                │   │
│  │                                                                     │   │
│  │   where C depends on liquidity L and price range, but NOT on σ     │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CONVERGENCE: As precision increases (σ → ∞, ε_exp2 → 0, ε_log2 → 0):     │
│                                                                             │
│               ε_total → 0                                                   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Precision │ σ = 2^F  │  Encoding Error  │  Approx Total Error     │   │
│  │────────────┼──────────┼──────────────────┼─────────────────────────│   │
│  │  F = 32    │  ~4×10⁹  │  ~10⁻¹⁰          │  ~10⁻⁸ (float-like)    │   │
│  │  F = 48    │  ~3×10¹⁴ │  ~10⁻¹⁵          │  ~10⁻¹³                 │   │
│  │  F = 64    │  ~2×10¹⁹ │  ~10⁻²⁰          │  ~10⁻¹⁸ (double-like)  │   │
│  │  F = 96    │  ~8×10²⁸ │  ~10⁻²⁹          │  ~10⁻²⁷ (quad-like)    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

_Figure 8: Error propagates through the fixed-point computation pipeline: encoding (continuous l to integer L), exponential approximation (L to S), and arithmetic operations (multiplication, subtraction). Each stage contributes a bounded error. The total error is the sum of these contributions, weighted by a constant C that depends on the economic quantities but not on precision parameters. As precision increases (larger σ, better exp2/log2 approximations), total error vanishes, ensuring convergence to the continuous model._

<!-- LaTeX/TikZ version for PDF:
\begin{figure}[h]
\centering
\begin{tikzpicture}
  % Two parallel tracks: continuous (top) and fixed-point (bottom)
  % Arrows showing error injection at each stage
  % Error bound formulas at bottom
\end{tikzpicture}
\caption{Error Propagation Chain in Fixed-Point Execution}
\label{fig:error-propagation}
\end{figure}
-->

For the arithmetic layer, it is convenient to bundle all per-operation rounding errors into a single worst-case aggregate quantity.

**Lemma 6.5 (Fixed-point arithmetic stability).**

Consider any fixed trade along a price path interval $[l_{\mathrm{start}}, l_{\mathrm{end}}]$ and a corresponding execution on a mesh with segments $k = k_0, \ldots, k_m$. Suppose each primitive fixed-point operation (addition, subtraction, multiplication, division, or rounding) incurs an absolute rounding error of at most $\delta_{\mathrm{op}}$ in the relevant Q-format, and that the total number of such operations in the CLMM evaluation is bounded by $N_{\mathrm{ops}}$ (depending only on protocol configuration, not on numerical precisions). Then the cumulative arithmetic error in any token delta can be bounded as

$$
  |E_{\mathrm{arith}}| \le N_{\mathrm{ops}} \cdot \delta_{\mathrm{op}} =: \varepsilon_{\mathrm{arith}},
$$

where $\varepsilon_{\mathrm{arith}}$ can be made arbitrarily small by increasing the fractional bit-lengths $Q_S, Q_P, Q_Q$.

_Proof Sketch._ Each primitive operation perturbs its exact real counterpart by at most $\delta_{\mathrm{op}}$, and the full computation is a finite composition of such operations. Bounding the number of operations by $N_{\mathrm{ops}}$ and summing their contributions yields the stated worst-case bound. $\blacksquare$

#### 6.6.2 Segment-wise CLMM error: $\Delta B$

Consider a signle mesh segment $k$ with true log-endpoints $l_a, l_b$ and constant liquidity $L_k$. The continuous model yields:

$$
\Delta B_k = L_k (2^{l_b} - 2^{l_a})
$$

The implementation instead computes:

1. Encoded log values:

   $$
   L_a = \mathrm{enc}_l(l_a), \quad L_b = \mathrm{enc}_l(l_b)
   $$

   with decoded logs $\hat{l}_a = \frac{L_a}{\sigma}, \hat{l}_b = \frac{L_b}{\sigma}$, satisfying $|\hat{l}_a - l_a|, |\hat{l}_b - l_b| \leq \frac{1}{2\sigma}$.

2. Sqrt-prices via $\operatorname{exp_2fp}$:

   $$
   \hat{S}_a = \frac{\operatorname{exp_2fp}(L_a)}{2^{Q_S}}, \quad \hat{S}_b = \frac{\operatorname{exp_2fp}(L_b)}{2^{Q_S}}
   $$

   with relative error ($|\hat{S}_x/S(\hat{l}_x) - 1| \leq \varepsilon_{\exp_2}$) for $x \in \{a,b\}$, where $S(\hat{l}_x) = 2^{\hat{l}_x}$.

3. Segment delta:

$$
\hat{\Delta B}_k = L_k \cdot (\hat{S}_b - \hat{S}_a)
$$

computed in fixed-point with a final rounding step.

Write the **total error** as:

$$
E_B := \hat{\Delta B}_k - \Delta B_k
$$

We decompose:

$$
\Delta B_k
= L_k \big( 2^{l_b} - 2^{l_a} \big)
= L_k \big( S(\hat{l}_b) 2^{l_b - \hat{l}_b} - S(\hat{l}_a)2^{l_a - \hat{l}_a} \big).
$$

We can separate:

- log-encoding error: $l_x - \hat{l}_x$, bounded by $\frac{1}{2\sigma}$,
- exp2 approximation error: $\hat{S}_x - S(\hat{l}_x)$, bounded relatively by $\varepsilon_{\exp_2}$,
- fixed-point arithmetic error in forming $\hat{\Delta B}_k$

A simple but useful bound is:

**Proposition 6.6 (Segment $\Delta B$ error bound).**

Assume:

- encoding error |$\hat{l}_x - l_x$| $\leq \frac{1}{2\sigma}$
- exp approximation error |$\hat{S}_x - S(\hat{l}_x)| \leq \varepsilon_{\exp_2} S(\hat{l}_x)$
- the multiplication and subtraction to compute $\hat{\Delta B}_k$ incur a total rounding error of at most $\varepsilon_{\mathrm{arith}} L_k \max(\hat{S}_a, \hat{S}_b)$.

Then:

$$
|E_B| \le C_1 L_k \max(S(l_a), S(l_b)) \left( \frac{1}{\sigma} + \varepsilon_{\exp_2} + \varepsilon_{\mathrm{arith}} \right)
$$

for some constant $C_1$ depending only on $\ln 2$ and the maximum log error $\frac{1}{2\sigma}$.

**_Proof Sketch._**

Use triangle inequality:

$$
|\hat{S}_b - \hat{S}_a - (2^{l_b} - 2^{l_a})|
\le |\hat{S}_b - S(\hat{l}_b)| + |\hat{S}_a - S(\hat{l}_a)| +
|S(\hat{l}_b) - 2^{l_b}| + |S(\hat{l}_a) - 2^{l_a}| +
\text{(arith rounding)}.
$$

The first two terms are bounded by $\varepsilon_{\exp_2} S(\hat{l}_x)$; the next two arise from encoding error and can be bounded via the Lipschitz constant of $2^l$ on the relevant interval; the last term is by assumption. Multiplying by $L_k$ yields the stated bound. $\blacksquare$

An analogous bound holds for $\Delta A_k$.

#### 6.6.3 Aggregation across segments

For a full trade, we traverse segments $k = k_0, k_1, \ldots, k_m$ and sum segment deltas:

$$
\Delta B_{\mathrm{true}} = \sum_{r=0}^{m} \Delta B_{k_r}, \quad
\hat{\Delta B}_{\mathrm{num}} = \sum_{r=0}^{m} \hat{\Delta B}_{k_r}
$$

The total error is:

$$
E_{B,\mathrm{tot}} := \hat{\Delta B}_{\mathrm{num}} - \Delta B_{\mathrm{true}}
= \sum_{r=0}^{m} E_{B,k_r}
$$

By Proposition 6.6:

$$
|E_{B,\mathrm{tot}}|
\le \left(\max_{r} C_1 L_{k_r} \max_x S(l_x)\right) \cdot m \cdot \left( \frac{1}{\sigma} + \varepsilon_{\exp_2} + \varepsilon_{\mathrm{arith}} \right)
$$

Similarly for $\Delta A$.

Thus, for any _fixed_ maximum number of segments per trade (which is bounded in practice by a protocol-configured mesh resolution), we obtain a linear bound:

$$
|E_{B,\mathrm{tot}}| \leq C_{tot} \left( \frac{1}{\sigma} + \varepsilon_{\exp_2} + \varepsilon_{\mathrm{arith}} \right)
$$

with $C_{tot}$ depending on liquidity magnitudes, price range, and the maximum segment count.

#### 6.6.4 Mesh approximation error

Recall from the main text that, in the continuous Layer-0 model, aggregate liquidity is given by:

$$
\lambda(l) = \sum_{i \in I} L_i \cdot 1_{[l_{lower,i}, l_{upper,i})}(l)
$$

and the CLMM differentials are:

$$
dA(l) = - \lambda(l) (\ln 2) \cdot 2^{-l} \ dl
, \quad
dB(l) = \lambda(l) (\ln 2) \cdot 2^{l} \ dl
$$

In Layer 2, we construct a mesh $\mathcal{M} = \{ l_0 < l_1 < \ldots < l_N \}$ and define a segment-wise constant approximation:

$$
\lambda^{(\mathcal{M})}(l) = \sum_{k=0}^{N-1} L_{\mathrm{AMM},k} \cdot 1_{[l_k,l_{k+1})}(l)
$$

with choices such as:

- midpoint sampling: $L_{\mathrm{AMM},k} = \lambda(\bar{l}_k)$,
- or exact average over the segment.

Because $\lambda$ is piecewise constant with jumps only at band boundaries, and we include all band boundaries in $\mathcal{M}$, we have:

$$
\lambda^{(\mathcal{M})}(l) = \lambda(l) \quad \text{for all } l \notin {l_0, \ldots, l_N}.
$$

The set of discrepancies is finite and hence of measure zero, so the integrals

$$
\int_{l_a}^{l_b} \lambda^{(\mathcal{M})}(l)2^{\pm l} dl
$$

coincide with those of $\lambda(l)$ for all intervals whose endpoints are nodes in $\mathcal{M}$. Thus:

**Proposition 6.7 (Exact mesh integrals for band-based $\lambda$).**

If all CLMM band boundaries are included in $\mathcal{M}$ and $L_{\mathrm{AMM},k}$ is chosen as any representative value of $\lambda(l)$ on $[l_k, l_{k+1})$, then for any mesh nodes ($l_a, l_b$):

$$
\int_{l_a}^{l_b} \lambda^{(\mathcal{M})}(l) 2^{\pm l} dl
= \int_{l_a}^{l_b} \lambda(l) 2^{\pm l} dl.
$$

Hence the only errors in ($\Delta A, \Delta B$) arise from numerical approximation, not from mesh discretization of $\lambda$.

When additional refinement points are inserted inside segments (e.g. to control per-segment error or for numerical stability), the mesh is merely partitioned further; ($\lambda^{(\mathcal{M})}$) remains equal to ($\lambda(l)$) almost everywhere, and the exact integrals are preserved.

In particular, the bounds in this subsection, and the convergence result below, are **independent** of:

- the choice of Layer-1 global slot spacing (Section 5.2), and
- any admissible Layer-2 refinement heuristic (Section 5.3),

as long as the mesh contains all band boundaries and order prices as nodes.

#### 6.6.5 Convergence to the continuous model

We now summarize the convergence behavior.

Let:

- $\sigma \to \infty$: increasing log-scaling precision,
- $\varepsilon_{\exp_2} \to 0$, $\varepsilon_{\log_2} \to 0$: improving exp/log approximations,
- fixed-point fractional lengths $Q_S, Q_P, Q_Q \to \infty$: reducing arithmetic rounding error,
- mesh diameter $\delta = \max_k (l_{k+1} - l_k) \to 0$: refining the execution mesh (while still including all band boundaries and order prices).

Under these limits, we obtain:

**Theorem 6.8 (Numerical convergence; uniform global precision).**

Fix any price path interval $[l_{\mathrm{start}}, l_{\mathrm{end}}]$ and any finite CLMM + CLOB state satisfying the local finiteness assumptions of the paper. Let ($\Delta A_{\mathrm{true}}, \Delta B_{\mathrm{true}}$) be the ideal continuous CLMM deltas along this path, and let ($\hat{\Delta A}, \hat{\Delta B}$) be the on-chain computed deltas using:

- log encoding with scaling $\sigma$,
- ($\operatorname{exp_2fp}$ and $\operatorname{log_2fp}$ with error bounds $\varepsilon_{\exp_2}, \varepsilon_{\log_2}$),
- Q-format arithmetic with rounding errors bounded per operation,
- a mesh $\mathcal{M}$ including all band boundaries and order prices, with maximum step $\delta$.

Then there exists a constant $C > 0$ (depending on liquidity and price bounds but not on the numerical parameters) such that:

$$
\big| \hat{\Delta A} - \Delta A_{\mathrm{true}} \big| + \big| \hat{\Delta B} - \Delta B_{\mathrm{true}} \big|
  \le C \left( \frac{1}{\sigma} + \varepsilon_{\exp_2} + \varepsilon_{\log_2} + \varepsilon_{\mathrm{arith}} \right)
$$

where $\varepsilon_{\mathrm{arith}}$ captures worst-case cumulative rounding error over all segments.

In particular, by jointly sending $\sigma \to \infty$, $\varepsilon_{\exp_2} \to 0$, $\varepsilon_{\log_2} \to 0$, and increasing fractional bits in Q-formats, the numerical execution converges to the continuous model:

$$
\hat{\Delta A} \to \Delta A_{\mathrm{true}}, \quad
\hat{\Delta B} \to \Delta B_{\mathrm{true}}
$$

**_Proof Sketch._**

Combine:

- the encoding error bound of Lemma 6.1,
- the $exp_2$-induced price error bound of Lemma 6.2,
- the arithmetic stability bound of Lemma 6.5,
- the segment-wise error bound of Proposition 6.6,
- linear aggregation across a finite number of segments,
- and the exactness of mesh-based integrals in Proposition 6.7.

All terms are bounded by constants multiple of ($\frac{1}{\sigma} + \varepsilon_{\exp_2} + \varepsilon_{\log_2} + \varepsilon_{\mathrm{arith}}$) for fixed liquidity and price ranges. $\blacksquare$

> **Intuition:** Theorem 6.8 is the "master convergence result" for the entire numerical system. It says: all the approximations we make—rounding log-prices to integers, approximating $2^x$ with polynomials, doing fixed-point arithmetic—add up to a total error that is proportional to our precision parameters. If you want more accuracy, just use more bits. There's no mysterious accumulation of errors or instability; the whole system behaves linearly in the precision you give it. This is the payoff of working in log-space: multiplicative price dynamics become additive, and errors compose predictably.

### 6.7 Implementation Considerations and Canonical Parameter Choices

The preceding subsections are stated in terms of abstract fixed-point formats and generic error parameters $(\sigma, \varepsilon_{\exp_2}, \varepsilon_{\log_2}, \varepsilon_{\mathrm{arith}})$. For practical deployment, it is useful to record one concrete parameter choice and its numerical implications. This subsection summarizes such a choice in a platform-agnostic way; a more detailed, chain-specific engineering specification can live in a separate whitepaper or protocol documentation.

#### 6.7.1 Log-Scaling Precision and Global Price Resolution

Recall that the log-sqrt-price is encoded as $L \in \mathbb{Z}_w$ with scaling $\sigma = 2^F$ (Section 6.1), so that $l \approx L/2^F$. Increasing $F$ improves the global resolution of prices in a uniform way across the entire state space.

Two representative configurations are:

- $F = 32$ (log step $2^{-32} \approx 2.3 \times 10^{-10}$),
- $F = 48$ (log step $2^{-48} \approx 3.6 \times 10^{-15}$).

From Lemma 6.1, the encoding-induced relative price error satisfies

$$
\left| \frac{\hat{P}}{P^*} - 1 \right| \lesssim C_P \cdot 2^{-F}
$$

for some constant $C_P \in [1,2]$. Thus:

- For $F = 32$, the global encoding error is $O(10^{-10})$.
- For $F = 48$, the global encoding error is $O(10^{-14})$, comparable to IEEE 754 double-precision relative error ($\sim 10^{-15}$) [@higham2002accuracy].

In both cases, the implied price range $P \sim 2^{2l}$ is far larger than any economically relevant region; the binding constraints in practice come from the chosen Q-formats for $S, P,$ and $Q$, not from the representable range of $L$.

#### 6.7.2 Canonical Fixed-Point Layout

A natural canonical layout, compatible with many modern execution environments, is:

- Log-sqrt-price: $L \in \mathbb{Z}_w$ (e.g. signed 64-bit or wider) with $F = 48$ fractional bits.
- Sqrt-price: $S_{\mathrm{fp}}$ in a Q$64.64$ format, so that $S \approx S_{\mathrm{fp}} / 2^{64}$.
- Price: derived as $P \approx S^2$ in a compatible Q-format (e.g. Q$64.64$ or Q$96.32$) and not necessarily stored persistently.
- Liquidity: $Q_{\mathrm{fp}}$ in a Q$64.64$ (or similar) format.

With these choices, individual arithmetic operations have rounding error of order $2^{-64}$, and — for realistic state bounds — intermediate products fit safely into 128-bit or wider accumulators. Together with $F = 48$, this yields global relative price errors of order $10^{-14}$ once the exp/log approximations are chosen appropriately, in line with the convergence bounds of Theorem 6.8.

#### 6.7.3 Integer–Fractional Decomposition for Exponentiation

On-chain computation of $S(L) = 2^{L/\sigma}$ can be implemented by splitting the exponent into integer and fractional parts. Writing

$$
L = q \cdot 2^F + r, \quad q \in \mathbb{Z}, \quad r \in \{0,1,\ldots,2^F - 1\},
$$

we have

$$
2^{L/2^F} = 2^{q + r/2^F} = 2^q \cdot 2^{r/2^F}.
$$

The integer factor $2^q$ is implemented by shifts in the chosen Q-format, while the fractional factor $2^{r/2^F}$ is approximated on the compact domain $[0,1)$, for example by a polynomial or table-based routine as sketched in Section 6.3.2. By choosing a monotone approximation on this domain and enforcing appropriate range checks, one obtains an $\operatorname{exp\_2fp}$ that is both strictly increasing and has relative error bounded by a design parameter $\varepsilon_{\exp_2}$.

This decomposition ensures that:

- the “large-scale” dependence on price is handled by cheap integer shifts, and
- the transcendental work is confined to a small, well-controlled fractional interval.

#### 6.7.4 Summary

The abstract error parameters in Section 6 admit concrete instantiations where:

- $F = 48$ (log scaling $\sigma = 2^{48}$),
- Q$64.64$ (or similar) formats are used for $S, P,$ and $Q$,
- $\operatorname{exp\_2fp}$ is implemented via integer–fractional decomposition with a monotone polynomial (or table) on $[0,1)$,

yielding global uniform relative price errors on the order of $10^{-14}$. The full engineering details — including specific polynomial coefficients, overflow policies, and platform-specific integer widths — are best treated in a separate implementation specification, but the existence of such a configuration shows that the continuous log-domain model is numerically realizable at numerical precision comparable to centralized exchange systems.

---

This section completes the bridge between the **ideal continuous log-domain model** and the **finite-precision on-chain realization**, showing that:

- the log-sqrt-price encoding is well-behaved and monotone,
- the numerical exp/log routines can be abstractly specified via monotonicity and error bounds,
- the CLMM dynamics implemented in fixed-point arithmetic converge to the continuous dynamics as precision is increased.

## 7. Unified Execution Semantics: CLMM + CLOB + Perpetuals

We now extend the Layer-2 execution model (Section 5.3 and 5.5) to specify **unified execution semantics** for:

- CLMM liquidity,
- resting CLOB orders, and
- perpetual futures positions (perps),

all evolving on the same log-sqrt-price axis $l$.

Throughout this section, we fix a single trading pair ($A, B$), a local mesh $\mathcal{M} = \{ l_0 < l_1 < \ldots < l_N \}$ segment-wise CLMM liquidity ${L_{\mathrm{AMM},k}}_{k=0}^{N-1}$ as in Section 5.5, and CLOB order sets ${\mathcal{J}_k}_{k=0}^N$ at each mesh node $l_k$ as in Section 5.3.5.

We introduce:

- a **deterministic node execution ordering** at each $l_k$,
- a precise rule for **crossing CLOB orders** when the price path hits their level,
- and a **perpetuals layer** whose mark prices, funding, and liquidations are all defined in the same coordinate $l$, and whose liquidations are implemented as additional flows through the CLMM + CLOB execution engine.

### 7.1 Local State and Price Paths

We first fix notation for the state on the mesh and notion of a price path.

#### 7.1.1 Node state

At a mesh node $l_k$, we define the **_local node state_** at time $t$ as:

$$
\mathcal{S}_k(t) = \big( A_{\mathrm{pool}}(t), B_{\mathrm{pool}}(t), {\mathcal{J}_k^{\mathrm{buy}}(t), \mathcal{J}_k^{\mathrm{sell}}(t)}, \mathcal{P}(t) \big)
$$

where:

- $A_{\mathrm{pool}}(t), B_{\mathrm{pool}}(t)$ are the CLMM reserves at time $t$,
- $\mathcal{J}_k^{\mathrm{buy}}(t)$ (resp. $\mathcal{J}_k^{\mathrm{sell}}(t)$) is the finite set of outstanding **buy** (resp. **sell**) limit orders with log-price equal to $l_k$,
- $\mathcal{P}(t)$ collects any additional per-account data such as perp positions and margin (to be introduced in Section 7.4).

The **spot log-price** at time $t$ is denoted by $l(t) \in \mathcal{M} \cup \bigcup_k (l_k, l_{k+1})$; the corresponding spot price is

$$
P(t) = P(l(t)) = 2^{2 l(t)}
$$

#### 7.1.2 Trade direction and price path

Consider a single incoming trade, abstractly represented as a **signed quantity** $\Delta q$ in token $B$ from the trader's perspective:

- $\Delta q > 0$: trader is _buying_ $B$ (using $A$ ), creating **upward** price pressure,
- $\Delta q < 0$: trader is _selling_ $B$ (for $A$ ), creating **downward** price pressure.

Given an initial log-price $l_* = l(0)$ and a target bound $l_{\mathrm{target}}$ (from user limits, risk constraints, or path planning), the **price path interval** is:

$$
I_{\mathrm{path}} =
\begin{cases}
[l_*, l_{\mathrm{target}}], & \text{if } l_{\mathrm{target}} \ge l_* \\
[l_{\mathrm{target}}, l_*], & \text{if } l_{\mathrm{target}} < l_*
\end{cases}
$$

as in Section 5.3.1. A **feasible execution path** is a continuous function

$$
l : [0, T] \to I_{\mathrm{path}}
$$

that is piecewise monotone and respects the segment-wise CLMM dynamics (Section 5.5) and node-wise CLOB matching rules that we now formalize.

Throughout, we assume the trade is **fully aggressive**: it consumes any immediately marketable liquidity (CLOB or CLMM) available along its direction until either the trade size is exhausted, the endpoint of $I_{\mathrm{path}}$ is reached, or no further liquidity is available.

### 7.2 Node Execution Ordering

We now specify a deterministic rule for the interaction between CLOB and CLMM at a mesh node. Intuitively, **all executable CLOB volume at the current node price is consumed before the price is allowed to move into the adjacent CLMM segment**.

We first formalize "direction" at a node.

#### 7.2.1 Direction at a node

Let $l(t) = l_k$ at some time $t$, and suppose the remaining trade quantity in token $B$ is $\Delta q_{\mathrm{rem}}(t)$.

- if $\Delta q_{\mathrm{rem}}(t) > 0$, we say the **direction** at $l_k$ is **up** (buy pressure, tending to move towards $l_{k+1}$, provided $k < N$),
- if $\Delta q_{\mathrm{rem}}(t) < 0$, we say the **direction** at $l_k$ is **down** (sell pressure, tending to move towards $l_{k-1}$, provided $k > 0$).

At a node $l_k$, **opposing-side** orders are those that can be matched with the current trade direction:

- If direction is up (buy pressure), opposing orders are **sell** limit orders at $l_k$: $\mathcal{J}_k^{\mathrm{sell}}$.
- If direction is down (sell pressure), opposing orders are **buy** limit orders at $l_k$: $\mathcal{J}_k^{\mathrm{buy}}$.

These are the orders that are **marketable** at the node's current price $P(l_k)$.

#### 7.2.2 Deterministic node execution rule

We adopt the following **node execution rule**:

**Rule 7.1 (Node execution ordering).**

Let the current log-price be $l(t) = l_k$ and remaining trade quantity $\Delta q_{\mathrm{rem}}(t) \neq 0$.

1. **CLOB phase at node (l_k).**

   - If $\Delta q_{\mathrm{rem}}(t) > 0$ (buy pressure), match against the opposing **sell** queue $\mathcal{J}_k^{\mathrm{sell}}$ at $P(l_k)$, following an internal deterministic order (e.g., price-time priority [@openbook2022; @daian2020flash] or pro-rata) until either:

     - $\mathcal{J}_k^{\mathrm{sell}}$ is empty, or
     - $\Delta q_{\mathrm{rem}}(t)$ is exhausted.

   - If $\Delta q_{\mathrm{rem}}(t) < 0$ (sell pressure), match against the opposing **buy** queue $\mathcal{J}_k^{\mathrm{buy}}$ at $P(l_k)$, analogously.

   No CLMM movement is allowed (i.e., $l(t)$) is held fixed at $l_k$ while there remains any opposing-side volume _and_ remaining trade quantity.

2. **CLMM phase from node (l_k).**

   If after step 1, we still have $\Delta q_{\mathrm{rem}}(t) \neq 0$, then:

   - If direction is up and $k < N$, we enter the CLMM segment $[l_k, l_{k+1})$ with liquidity $L_{\mathrm{AMM},k}$ and evolve $l(t)$ within that segment according to the CLMM dynamics (Section 5.5), until:

     - either the trade is fully executed (remaining quantity hits zero) at some interior point $l \in (l_k, l_{k+1})$, or
     - the segment boundary $l_{k+1}$ is reached with remaining quantity.

   - If direction is down and $k > 0$, we similarly enter segment $[l_{k-1}, l_k)$ with liquidity $L_{\mathrm{AMM},k-1}$, moving towards $l_{k-1}$.

3. **Iteration across nodes.**

- If a CLMM phase step ends at node $l_{k\pm 1}$ with remaining quantity $\Delta q_{\mathrm{rem}}(t) \neq 0$, we repeat the procedure from step 1 at the new node.

This rule imposes an unambiguous ordering between:

- resting CLOB liquidity at the node, and
- CLMM segment-wise liquidity adjacent to the node.

In particular, at any node with opposing-side depth, **CLOB orders at that level trade first** at the node's price $P(l_k)$.

**Figure 9: Node Execution Ordering (Rule 7.1)**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    NODE EXECUTION ORDERING: CLOB BEFORE CLMM                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  At each mesh node l_k, execution follows a strict two-phase protocol:     │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │   ┌───────────────────────────────────────────────────────────┐    │   │
│  │   │          PHASE 1: CLOB MATCHING (price frozen)            │    │   │
│  │   └───────────────────────────────────────────────────────────┘    │   │
│  │                                                                     │   │
│  │         Incoming              At Node l_k                          │   │
│  │         Buy Order   ───────►  ┌─────────────────────┐              │   │
│  │         (Δq > 0)              │  SELL orders queue  │              │   │
│  │                               │  at price P(l_k)    │              │   │
│  │                               │  ┌───┬───┬───┬───┐  │              │   │
│  │                               │  │ S₁│ S₂│ S₃│...│  │ ◄── Match   │   │
│  │                               │  └───┴───┴───┴───┘  │     these   │   │
│  │                               └─────────────────────┘     first    │   │
│  │                                                                     │   │
│  │         • Price l(t) stays FIXED at l_k during CLOB matching       │   │
│  │         • All fills occur at exactly P(l_k)                        │   │
│  │         • Continue until: orders exhausted OR trade filled         │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                               │                                             │
│                               │ If Δq_rem ≠ 0 (trade not complete)         │
│                               ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │   ┌───────────────────────────────────────────────────────────┐    │   │
│  │   │          PHASE 2: CLMM SEGMENT TRAVERSAL                  │    │   │
│  │   └───────────────────────────────────────────────────────────┘    │   │
│  │                                                                     │   │
│  │         l_k                               l_{k+1}                  │   │
│  │          │                                   │                      │   │
│  │          ▼                                   ▼                      │   │
│  │          ●━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━●                      │   │
│  │          │◄────── CLMM segment ──────────►│                        │   │
│  │          │      liquidity = L_AMM,k        │                        │   │
│  │          │                                 │                        │   │
│  │          │    l(t) moves continuously     │                        │   │
│  │          │    within segment as trade     │                        │   │
│  │          │    executes against AMM        │                        │   │
│  │                                                                     │   │
│  │         • Price moves: P(l_k) → P(l) as l increases                │   │
│  │         • Uses segment formulas: ΔB = L·(2^l - 2^l_k)              │   │
│  │         • Stop when: segment boundary reached OR trade filled      │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                               │                                             │
│                               │ If boundary reached with Δq_rem ≠ 0        │
│                               ▼                                             │
│                    ┌─────────────────────┐                                  │
│                    │  Move to node l_{k+1}│                                 │
│                    │  Repeat from Phase 1 │                                 │
│                    └─────────────────────┘                                  │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  KEY GUARANTEE: No arbitrage between CLOB and CLMM at same price level    │
│                 (CLOB always executes first while available)               │
└─────────────────────────────────────────────────────────────────────────────┘
```

_Figure 9: The node execution ordering rule ensures deterministic, fair execution at each mesh node. Phase 1 matches all available opposing CLOB orders at the fixed price P(l_k) before any price movement. Phase 2 enters the adjacent CLMM segment only after CLOB depth is exhausted, allowing price to move continuously. This two-phase protocol prevents arbitrage between CLOB and CLMM at the same price level and provides price-time priority for limit orders._

<!-- LaTeX/TikZ version for PDF:
\begin{figure}[h]
\centering
\begin{tikzpicture}
  % Two-phase diagram with CLOB matching box and CLMM segment arrow
  % Show price frozen during Phase 1, moving during Phase 2
\end{tikzpicture}
\caption{Node Execution Ordering (Rule 7.1)}
\label{fig:node-execution-ordering}
\end{figure}
-->

#### 7.2.3 No-arbitrage between CLOB and CLMM at a node

We now formalize that, under Rule 7.1, there is **no opportunity to trade at different prices** at the same node between the CLMM and the CLOB.

**Proposition 7.2 (No CLOB-CLMM arbitrage at node).**

Fix a node $l_k$ and suppose the spot log-price $l(t)$ equals $l_k$ at time $t$. Suppose there exists opposing-side CLOB liquidity at $l_k$ (i.e., $\mathcal{J}_k^{\mathrm{sell}} \neq \emptyset$ for buy pressure, or $\mathcal{J}_k^{\mathrm{buy}} \neq \emptyset$ for sell pressure). Then, under Rule 7.1:

1. All trades executed at time $t$ occur at the unique price $P(l_k)$, regardless of whether they consume CLOB or CLMM liquidity.
2. No trader can execute at a strictly better price via the CLMM while opposing-side CLOB liquidity remains at $l_k$.

**_Proof._**

Under Rule 7.1, as long as opposing-side CLOB depth is present at $l_k$ and the trade has remaining quantity, the CLMM phase is not entered and the log-price $l(t)$ is frozen at $l_k$. Therefore:

- Any instantaneous CLMM differential (dA, dB) at that moment (if we hypothetically allowed a CLMM trade of infinitesimal size) would be priced at the marginal price associated with $l_k$, namely $P(l_k)$.
- All actual executions during the CLOB phase occur at price $P(l_k)$ by definition of the limit order price.

Since the CLMM phase is only entered after all opposing CLOB depth at $l_k$ is exhausted, there is no state in which:

- opposing CLOB at price $P(l_k)$ remains, and
- the CLMM trades at a different price.

Thus, during the CLOB phase, all trades execute at the same unique price $P(l_k)$, and there is no opportunity to buy cheaper or sell higher via the CLMM as long as CLOB liquidity remains at that node. $\square$

> **Intuition:** Proposition 7.2 ensures fair execution: when both a limit order book and an AMM coexist at the same price level, you can't "game" the system by picking whichever gives you a better deal. The rule is simple—limit orders always get filled first at their posted price. Only after they're exhausted does the AMM kick in. Since both execute at the same price, there's no arbitrage between them. This is crucial for a hybrid system: it means limit-order makers get proper priority, and takers can't exploit price discrepancies.

#### 7.2.4 Price continuity

Note that in Rule 7.1:

- during a CLOB phase at node $l_k$, the price path $l(t)$ is **constant** $l(t) = l_k$,
- during a CLMM phase, $l(t)$ evolves **continuously** within a segment $[l_k, l_{k+1})$ under the CLMM dynamics (Section 5.5),
- at the transition between segments $[l_k, l_{k+1})$ and $[l_{k+1}, l_{k+2})$, the endpoint $l_{k+1}$ is shared, so the concatenated path is continuous.

Thus:

**Lemma 7.3 (Price path continuity).**

Under Rule 7.1, for any aggressive trade and any finite set of CLOB orders on the mesh, the resulting price path $l(t)$ is continuous on $[0,T]$. CLOB executions produce _jumps in reserves and order book state_ but not jumps in $l(t)$.

**_Proof._**

Immediate from the construction: all CLOB matching occurs at fixed $l_k$; all CLMM movement follows continuous evolution within segments; segment endpoints match. $\square$

### 7.3 Crossing a CLOB order at a Mesh Node

The local mesh construction (Section 5.3.2-5.3.5) ensures that each CLOB order price $l_{0,j}$ appears as some mesh node $l_k$. We now formalize how an **incoming price path that crosses such a node** must interact with the order.

Let $j$ be a CLOB order with log-price $l_{0,j}$ and quantity $q_j > 0$, and suppose $l_{0,j} = l_k$ for some $k$. Assume the trade direction is such that the order is _marketable_ once the path reaches $l_k$ (i.e., a buy-side aggressive trade crossing a resting sell, or vice versa).

We consider the limit behavior as the price approaches $l_k$ from below or above.

#### 7.3.1 Crossing in log-space

Consider a monotone segment of the price path $l(t)$ with direction up or down, and suppose there exists a time $t_0$ such that:

$$
\lim_{\epsilon \to 0^+} l(t_0 - \epsilon) = l^- \neq l_k, \quad \lim_{\epsilon \to 0^+} l(t_0 + \epsilon) = l^+
$$

with $l^- < l_k \leq l^+$ in the case of upward motion, or $l^- > l_k \geq l^+$ in the case of downward motion.

We say that the path **crosses the node** $l_k$ at time $t_0$. By construction of the execution algorithm, the following holds.

**Proposition 7.4 (Instantaneous consumption at order price).**

Let $j$ be an order at node $l_k$ that is marketable given the direction of motion, and suppose a trade's price path crosses $l_k$ at time $t_0$ as above. Then under Rule 7.1:

1. Immediately upon reaching $l_k$ (i.e. when $l(t) = l_k$), the execution enters a CLOB phase at node $k$ and consumes order $j$ (possibly partially, depending on remaining trade quantity).
2. The token reserves $A_{\mathrm{pool}}, B_{\mathrm{pool}}$ are **discontinuous** at time $t_0$ due to the discrete transfer from the resting order, but the log-price $l(t)$ is continuous at $t_0$.
3. If the remaining trade quantity after matching at $l_k$ is nonzero, the execution then continues into the adjacent CLMM segment in the direction of motion.

**_Proof Sketch._**

As soon as the path reaches $l_k$, Rule 7.1 mandates a CLOB phase at node $k$ before any further CLMM movement. Since order $j$ is marketable (opposing-side at the node's price), it is included in the set of orders considered in step 1 of Rule 7.1 and is either fully or partially consumed at price $P(l_k)$. This induces a jump in the trader's and counterparty's balances and, if the counterparty is external to the pool, no immediate change in pool reserves; if the CLOB is internalized by the pool or a hybrid design, it can be represented as a discrete jump in the pool or LP balances. In either case, the state of the book changes discretely, while $l(t)$ remains pinned at $l_k$ during the matching, hence continuous at $t_0$.

Once no further opposing-side liquidity at $l_k$ remains (or the trade is fully filled), Rule 7.1 allows the execution to proceed into the adjacent CLMM segment in the direction of the residual trade quantity, yielding the claimed behavior. $\square$

The limit notation

$$
\lim_{\epsilon \to 0^+} l(t_0 \pm \epsilon) = l_{0,j}
$$

is thus compatible with the execution semantics: the price path is continuous through $l_{0,j}$, while the **state of reserves and order queue** may have discrete jumps at that time.

### 7.4 Perpetuals on the Log-Price Axis

We now extend the framework to include perpetual futures (perps) [@bitmex2016perpetual; @dydx2023v4] whose mark price, funding, and liquidation logic are all expressed on the same log-price axis $l$. This subsection describes a **generic perpetual module** that can be layered on the log-axis, including the standard funding rate mechanism used by most existing perpetual protocols. In Section 8, we specialize to a **spot-anchored design** where funding is identically zero by construction; the present subsection establishes the general execution semantics that both designs share.

The key design goal is:

- perps **do not introduce a separate price notion**, but instead derive all price-dependent logic from $P(l)$, and
- perps **inject additional order flow** (liquidations, forced reductions) into the same CLMM + CLOB execution engine defined above.

#### 7.4.1 Perpetual positions and mark price

We fix a **settlement token**, which without loss of generality we take to be $A$. A perp position for account $m$ is specified by:

- a signed position size $q_m \in \mathbb{R}$ in units of $B$ (positive for long, negative for short),
- a margin balance $M_m \geq 0$ in token $A$,
- possibly additional accrued funding / fees states $F_m$.

The **mark log-price** at time $t$ is simply the pool log-price:

$$
l_{\mathrm{mark}}(t) := l(t)
$$

and the mark price is:

$$
P_{\mathrm{mark}}(t) := P(l_{\mathrm{mark}}(t)) = 2^{2 l(t)}.
$$

For each account $m$, we define the **mark-to-market equity** (in units of token $A$) as;

$$
E_m(t) = M_m + q_m \cdot P_{\mathrm{mark}}(t) + F_m
$$

where $F_m$ absorbs any accumulated funding payments, realized PnL, and fees.

This ties all perp valuation to the **same spot price process** $P(l(t))$ generated by CLMM + CLOB execution.

#### 7.4.2 Funding rate

We do not commit to a particular funding scheme but assume a generic process where a scalar **funding rate** $f(t)$ [@bitmex2016perpetual; @dydx2023v4] is derived from the discrepancy between:

- the perp's implicit "fair" price $P_{\mathrm{perp}}(t)$ (e.g., oracle or basis-adjusted), and
- the spot price $P_{\mathrm{mark}}(t)$.

One natural choice is:

$$
P_{\mathrm{perp}}(t) = P\big(l_{\mathrm{oracle}}(t)\big)
$$

where $l_{\mathrm{oracle}}(t)$ is an external log-price process (e.g., a time-weighted oracle). The **basis** in log-space is then:

$$
\Delta l_{\mathrm{basis}}(t) = l_{\mathrm{perp}}(t) - l_{\mathrm{oracle}}(t)
$$

and a funding rate function $f : \mathbb{R} \to \mathbb{R}$ (e.g. linear) can be applied:

$$
f(t) = f\big( \Delta l_{\mathrm{basis}}(t) \big).
$$

Funding over a small time interval [$t, t + dt$] updates each account's ($F_m$) according to:

$$
dF_m(t) = - q_m \cdot f(t) \ dt
$$

with $\sum_m dF_m(t) = 0$ (funding is a zero-sum transfer between longs and shorts). The precise choice of $f$ and oracle process $l_{\mathrm{oracle}}(t)$ is orthogonal to the unified execution semantics; all that matters is that they are functions of the same log-price axis.

#### 7.4.3 Liquidation thresholds in $l$-space

Following standard perpetual protocol design [@bitmex2016perpetual; @dydx2023v4], each account $m$ has:

- an **initial margin requirement** $\mathrm{IM}_m(q_m)$,
- a **maintenance margin requirement** $\mathrm{MM}_m(q_m)$,

both specified as nonnegative functions of position size $q_m$ (and possibly volatility, portfolio composition, etc.).

An account is **eligible for liquidation** at time $t$ when:

$$
E_m(t) \leq \mathrm{MM}_m(q_m).
$$

This inequality can be rewritten in terms of log-price $l(t)$ via:

$$
E_m(t) = M_m + q_m \cdot 2^{2 l(t)} + F_m.
$$

For any fixed account $m$ and fixed margin state ($M_m, F_m, q_m$), this condition defines a **liquidation region in log-price space**. In particular, if $q_m > 0$ (long), then there exists a critical log-price $l_{\mathrm{liq},m}$ solving:

$$
M_m + q_m \cdot 2^{2 l_{\mathrm{liq},m}} + F_m = \mathrm{MM}_m(q_m)
$$

such that $E_m(t) \leq \mathrm{MM}_m(q_m)$ for all $l(t) \leq l_{\mathrm{liq},m}$. Analogous statements hold for shorts.

Thus, **liquidation boundaries are themselves level sets in the log-price axis**.

#### 7.4.4 Liquidation as forced order flow through the engine

When $E_m(t) \leq \mathrm{MM}_m(q_m)$, the protocol may initiate a **liquidation action** for account $m$, which we model as an **aggressive trade** along the same CLMM + CLOB execution machinery.

For concreteness, consider a simple policy: fully close the position $q_m$ upon liquidation. The liquidator (or protocol) generates a trade:

- For a long position ($q_m > 0$), a **sell** order of size $q_{\mathrm{liq}} = q_m$ in token $B$,
- For a short position ($q_m < 0$), a **buy** order of size $|q_m|$.

We treat this as an aggressive trade with remaining quantity $\Delta q_{\mathrm{rem}}(0) = - q_m$ (for closing a long) or $\Delta q_{\mathrm{rem}}(0) = |q_m|$ (for closing a short, consistent with the sign convention), starting at the current log-price $l(0) = l_{\mathrm{mark}}(t)$. This trade is then executed by:

- first matching against opposing-side CLOB depth at the node,
- then traversing CLMM segments if necessary,

**exactly** as in Rule 7.1.

Formally, let ($A_{\mathrm{pool}}^{\mathrm{pre}}, B_{\mathrm{pool}}^{\mathrm{pre}}, \mathcal{P}^{\mathrm{pre}}$) be the state just before liquidation; let ($A_{\mathrm{pool}}^{\mathrm{post}}, B_{\mathrm{pool}}^{\mathrm{post}}, \mathcal{P}^{\mathrm{post}}$) be the state immediately after the forced trade is fully executed. Then:

- The perp position for account $m$ is updated to zero: $q_m^{\mathrm{post}} = 0$,
- The margin account $M_m$ is debited or credited by the realized PnL of the closing trade (and fees),
- The CLMM + CLOB state has been updated according to our usual dynamics,
- Other accounts are unaffected except via price impact and funding transfers.

This yield the following structural property.

**Proposition 7.5 (Unified execution under liquidation).**

Assume liquidations are implemented as aggressive trades through the unified CLMM + CLOB engine as above. Then:

1. Liquidations do not introduce any new price notion beyond $P(l)$; the closing trades are executed at the same prices and along the same path as any other aggressive trade.
2. The total change in pool reserves $(A_{\mathrm{pool}}, B_{\mathrm{pool}})$ and the price path $l(t)$ during a liquidation event are determined solely by:

   - the existing CLMM liquidity density $\lambda(l)$,
   - the CLOB order book $\mu$,
   - and the liquidation trade size,
     with no special cases required

3. If we consider a counterfactual scenario where an external trader submitted an identical aggressive order at the same time, the resulting price path and pool reserve changes would be identical.

**_Proof Sketch._**

In our construction, a liquidation is represented as nothing more than a specific aggressive trade with a deterministic size and direction, entering the same execution machinery as any other trade. The execution semantics (Rule 7.1 and the CLMM segment dynamics of Section 5.5) are independent of the identity of the flow origin (liquidation vs. voluntary trade). Therefore:

1. The price path $l(t)$ is governed by the same local mesh and liquidity density, and hence the same mapping $P(l)$.
2. The pool reserve deltas and order book updates follow exactly the same rules.
3. Substituting the liquidation with an equal-size aggressive order from an arbitrary external trader at the same log-price yields an identical execution; the only difference is how the resulting PnL is attributed to margin accounts.

$\square$

This shows that perpetuals are **logically layered on top of**, rather than "beside", the spot CLMM + CLOB engine. All complexity of perps (funding, margin, liquidation) ultimately reduces to:

- accounting changes in $\mathcal{P}$, and
- additional aggressive trades along the unified log-price axis.

### 7.5 Summary of Unified Semantics

The execution semantics developed in this section can be summarized as follows:

- The **log-price process** $l(t)$ is continuous, driven by a combination of:

  - discrete CLOB executions at mesh nodes (which do not move $l$), and
  - continuous CLMM evolution along segments (which move $l$).

- At any node $l_k$,

  - all **marketable CLOB volume at that node** is consumed first at price $P(l_k)$,
  - only then is the adjacent CLMM segment entered, ensuring **no CLOB-CLMM arbitrage at the node**.

- **CLOB orders** at price $l_{0,j}$ are consumed at the instant the price path reaches $l_{0,j}$, causing jumps in reserves and order book state but not in the log-price itself.

- **Perpetuals**:

  - derive all pricing, margins, and liquidations from the same log-price axis via $P(l)$,
  - use mark-to-market equity functions $E_m(t)$ expressed in $l$,
  - and implement liquidations as ordinary aggressive trades through the CLMM + CLOB engine.

Thus, CLMM liquidity, CLOB depth, and perpetuals all share a **single, coherent, execution semantics** on the log-price axis. The low-level numerical realizations (fixed-point encoding, exp/log routines, mesh refinement) covered in Section 6 do not alter these semantics; they only approximate them with provably bounded error.

## 8. Spot-Anchored Derivative Layer on the Log-Price Axis

In this section, we introduce a derivative layer (perpetual swaps, expiring futures, and options) that is _entirely anchored_ to the spot log-price process generated by the unified CLMM + CLOB engine. This represents a **specialization** of the generic perpetual semantics introduced in Section 7.4: we retain the same execution machinery (mark price derived from $l(t)$, liquidations as forced order flow, unified CLMM + CLOB engine), but eliminate the funding rate mechanism entirely by anchoring derivative prices directly to spot and hedging atomically at trade time.

The key design principles are:

1. There is a **single price process** $l(t)$, hence a single spot price process $P(l(t)) = 2^{2 l(t)}$.
2. Derivatives **do not define a separate price curve**; their payoffs and mark-to-market values are functionals of $P(l(t))$ only.
3. The protocol maintains **delta-neutrality** with respect to the underlying by hedging net derivative exposure via the same CLMM + CLOB spot engine.
4. LPs in the spot layer **do not warehouse derivative directional risk**; they see derivative hedging flow as additional spot order flow and earn corresponding fees.

This yields a derivative architecture with:

- no separate "perp price",
- no basis between spot and perps,
- and no need for traditional funding-rate mechanics.

Throughout, we work with a fixed trading pair ($A, B$) and the log-sqrt-price coordinate $l(t)$ as in Sections 5-7.

### 8.1 State and Account Model

We first formalize the global state and the account structure required for derivatives.

#### 8.1.1 Global spot state

We retain the unified spot state from Section 7. At time $t$, the spot engine is characterized by:

- log-sqrt price $l(t) \in \mathbb{R}$,
- spot price $P(t) := P(l(t)) = 2^{2 l(t)}$,
- CLMM reserves $(A_{\mathrm{pool}}(t), B_{\mathrm{pool}}(t))$,
- CLOB order sets ${\mathcal{J}_k^{\mathrm{buy}}(t), \mathcal{J}_k^{\mathrm{sell}}(t)}$ on the mesh nodes $l_k$,
- liquidity density $\lambda(l)$ as in Section 5.1.2.

The evolution of $l(t)$ and pool reserves is governed by the unified CLMM + CLOB execution semantics of Section 7.

#### 8.1.2 Derivative accounts

Let $\mathcal{M}$ be the set of derivative accounts (traders) indexed by $m \in \mathcal{M}$. Each account $m$ has:

- a **margin balance** $M_m(t) \geq 0$ in token $A$,
- a set of **perpetual positions** $\{ q_{m, \alpha}(t) \}_{\alpha \in \mathcal{I}_{\mathrm{perp}}}$, where:

  - $\alpha$ indexes derivative instruments (e.g. product ID, market),
  - $q_{m, \alpha}(t) \in \mathbb{R}$ is the signed position size in units of token $B$ (positive = long, negative = short),

- (optionally) a set of **futures** or **options** positions $\{ \pi_{m, \beta}(t) \}_{\beta \in \mathcal{I}_{\mathrm{opt}}}$,
- an accumulated **PnL and fee state** $F_m(t) \in \mathbb{R}$ in units of token $A$.

The exact indexing of instruments is not essential; we focus first on a single perpetual instrument and later generalize.

### 8.2 Spot-Anchored Perpetual Swaps

We begin with a single perpetual swap on $(A, B)$ and define it as a **delta-one claim** on the spot price process $P(l(t))$, without its own price curve.

#### 8.2.1 Perpetual positions

Fix a single perpetual instrument and omit the $\alpha$ index for brevity. A perpetual position for account $m$ at time $t$ is specified by:

- size $q_m(t) \in \mathbb{R}$ in units of token $B$,
- an **effective entry price** $P_{\mathrm{entry},m}(t) \in \mathbb{R}_{>0}$,
- a margin balance $M_m(t)$,
- and accumulated PnL + fees $F_m(t)$.

We assume that position updates (opens, partial closes, reversals) are normalized so that at any time $t$ there exists a well-defined effective entry price $P_{\mathrm{entry},m}(t)$, for instance via volume-weighted average on notional.

#### 8.2.2 Mark price and mark-to-market equity

Let the **mark log-price** at time $t$ be the current spot log-price:

$$
l_{\mathrm{mark}}(t) := l(t)
$$

and the **mark price** be the spot price:

$$
P_{\mathrm{mark}}(t) := P(l_{\mathrm{mark}}(t)) = 2^{2 l(t)}.
$$

The mark-to-market equity of account $m$ in units of token $A$ is:

$$
E_m(t) = M_m(t) + q_m(t)\big( P_{\mathrm{mark}}(t) - P_{\mathrm{entry},m}(t) \big) + F_m(t)
$$

There is no separate perp mid-price; the unique mark price is $P_{\mathrm{mark}}(t)$.

**Definition 8.1 (Spot-anchored perpetual swap).**

A spot-anchored perpetual swap on $(A, B)$ is a contract whose PnL for account $m$ is given by:

$$
\mathrm{PnL}_m(t) = q_m(t) \big( P(l(t)) - P_{\mathrm{entry},m}(t) \big) + F_m(t)
$$

with **mark price** equal to the spot price $P(l(t))$ at all times.

In particular, there is no funding term; all valuation is strictly tied to the spot process.

#### 8.2.3 Basis and absence of funding

In traditional perpetual swap designs [@bitmex2016perpetual; @dydx2023v4; @gmx2022technical; @perp2021v2], one distinguishes:

- a perp mark price $P_{\mathrm{perp}}(t)$,
- a spot price $P_{\mathrm{spot}}(t)$,

and defines a **basis**:

$$
\mathrm{basis}(t) = P_{\mathrm{perp}}(t) - P_{\mathrm{spot}}(t)
$$

with funding designed to penalize non-zero basis.

In our design, we identify:

$$
P_{\mathrm{perp}}(t) := P_{\mathrm{mark}}(t) = P(l(t)), \quad P_{\mathrm{spot}}(t) := P(l(t))
$$

hence:

**Proposition 8.2 (Zero basis by construction).**

For a spot-anchored perpetual swap as in Definition 8.1, the basis is identically zero:

$$
\mathrm{basis}(t) \equiv 0, \quad \forall t \ge 0
$$

and hence any traditional funding term proportional to $\mathrm{basis}(t)$ vanishes identically.

**_Proof._**

Immediate from the identifications $P_{\mathrm{perp}}(t) = P_{\mathrm{spot}}(t) = P(l(t))$ for all $t$. $\square$

In particular, the derivative layer introduces no independent price curve; all economics are inherited from the spot log-price process.

**Figure 10: Spot-Anchored vs. Traditional Perpetual Design**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PERPETUAL SWAP DESIGN COMPARISON                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  TRADITIONAL PERP DESIGN (e.g., most CEX and DEX perps [@bitmex2016perpetual; @dydx2023v4; @gmx2022technical; @perp2021v2])   │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│     P_perp(t)  ──────────────────────┐                                     │
│     (perp mark price,               │                                      │
│      from order book                │                                      │
│      or TWAP oracle)                │  basis(t) = P_perp - P_spot         │
│                                     │             (can be ≠ 0)             │
│     P_spot(t)  ─────────────────────┤                                      │
│     (spot price,                    │                                      │
│      from external                  │                                      │
│      oracle or AMM)                 │                                      │
│                                                                             │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │  FUNDING RATE = f(basis) · position_size                        │    │
│     │                                                                 │    │
│     │  • Longs pay shorts when basis > 0 (perp overpriced)           │    │
│     │  • Shorts pay longs when basis < 0 (perp underpriced)          │    │
│     │  • Applied periodically (e.g., every 8 hours)                  │    │
│     │  • Creates carry costs and complexity                          │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  SPOT-ANCHORED PERP DESIGN (this paper)                                    │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│     P_perp(t) = P_spot(t) = P(l(t)) = 2^{2l(t)}                           │
│                     │                                                       │
│                     │  ◄── SINGLE PRICE SOURCE                             │
│                     │      (the unified spot engine)                        │
│                     │                                                       │
│                     ▼                                                       │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │  basis(t) ≡ 0    (by construction, for all t)                  │    │
│     │                                                                 │    │
│     │  FUNDING RATE ≡ 0  (no funding mechanism needed)               │    │
│     │                                                                 │    │
│     │  • No carry costs for holding positions                        │    │
│     │  • No oracle risk for mark price                               │    │
│     │  • PnL purely from spot price movement                         │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│  HOW IT WORKS:                                                             │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                             │
│     User opens long perp (+q)  ───────┐                                    │
│                                       │                                     │
│                                       ▼                                     │
│                         ┌─────────────────────────┐                        │
│                         │   Protocol Hedge        │                        │
│                         │   Sells q tokens spot   │                        │
│                         │   via CLMM + CLOB       │                        │
│                         └───────────┬─────────────┘                        │
│                                     │                                       │
│                                     ▼                                       │
│              Net protocol exposure: +q (perp) - q (spot) = 0              │
│                                                                             │
│     Result: Protocol is always delta-neutral; no funding needed to        │
│             incentivize balance between longs and shorts                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

_Figure 10: Traditional perpetual designs maintain separate perp and spot prices, creating a basis that must be managed via funding rate payments. Our spot-anchored design uses the unified spot price P(l(t)) as the single source of truth for both spot and perp valuation. By construction, basis ≡ 0, eliminating funding entirely. The protocol maintains delta-neutrality by hedging perp exposure directly in the spot engine, not by relying on funding to balance longs and shorts._

<!-- LaTeX/TikZ version for PDF:
\begin{figure}[h]
\centering
\begin{tikzpicture}
  % Left side: traditional design with two price lines and basis gap
  % Right side: spot-anchored with single price line
  % Arrows showing funding flow vs. hedge flow
\end{tikzpicture}
\caption{Spot-Anchored vs. Traditional Perpetual Design}
\label{fig:spot-anchored-vs-traditional}
\end{figure}
-->

### 8.3 Protocol-Level Delta Hedging on the Log Axis

To avoid warehousing net directional risk, the protocol maintains a hedge in the underlying spot asset via the CLMM + CLOB engine. This approach adapts classical delta-hedging techniques from options theory [@black1973pricing; @merton1973theory] to the perpetual swap context.

#### 8.3.1 Net perp exposure and hedge position

Let

$$
Q_{\mathrm{net}}(t) = \sum_{m \in \mathcal{M}} q_m(t)
$$

be the aggregate perp exposure in units of token $B$. The protocol maintains a **spot hedge position** $H_{\mathrm{spot}}(t) \in \mathbb{R}$ in token $B$ via trades in the underlying spot engine.

We define the **delta-neutrality constraint**:

$$
H_{\mathrm{spot}}(t) = - Q_{\mathrm{net}}(t), \quad \forall t
$$

Thus, whenever a perp position changes (e.g. a trade opens or closes), the protocol adjusts its spot hedge by:

$$
\Delta H_{\mathrm{spot}} = - \Delta Q_{\mathrm{net}}
$$

by executing a corresponding spot trade through the CLMM + CLOB engine.

#### 8.3.2 Atomic hedging as a composite operation

At the level of the economic model, we view each perp trade as an **atomic composite operation** consisting of:

1. Updating the perp position $q_m$ and their entry prices $P_{\mathrm{entry},m}$,
2. Adjusting the spot hedge $H_{\mathrm{spot}}$ so that

   $$
   H_{\mathrm{spot}}^{\mathrm{post}} = H_{\mathrm{spot}}^{\mathrm{pre}} - \Delta Q_{\mathrm{net}}
   $$

3. Executing the hedge adjustment $\Delta H_{\mathrm{spot}}$ as an aggressive spot trade via the unified CLMM + CLOB execution semantics (Section 7).

We assume these steps occur at a common logical time $t_0$ and at the **current log-price** $l(t_0)$, so that:

- the perp's entry/exit price equals $P(l(t_0))$,
- the hedge trade is priced according to the same spot engine and may move $l(t)$ depending on size and available liquidity.

This leads to the following.

**Definition 8.3 (Atomic hedged perp trade).**

An atomic hedged perp trade at time $t_0$ is a composite state update $\mathcal{S}(t_0^-) \to \mathcal{S}(t_0^+)$ satisfying:

1. For some account $m$, $q_m$ changes by $\Delta q_m \neq 0$ with entry or exit price equal to $P(l(t_0^-))$,
2. The net exposure changes by $\Delta Q_{\mathrm{net}} = \Delta q_m$,
3. The protocol's spot hedge position is updated by $\Delta H_{\mathrm{spot}} = - \Delta Q_{\mathrm{net}}$,
4. The change $\Delta H_{\mathrm{spot}}$ is realized as an aggressive trade through the CLMM + CLOB engine starting from log-price $l(t_0^-)$.

At the modeling level, we treat this composite as instantaneous, with resulting spot price $l(t_0^+)$ determined by the unified execution semantics.

**Figure 11: Atomic Hedged Perp Trade Mechanics**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ATOMIC HEDGED PERP TRADE (Definition 8.3)                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  BEFORE TRADE (time t₀⁻):                                                   │
│  ══════════════════════════════════════════════════════════════════════    │
│                                                                             │
│  ┌─────────────────────────┐         ┌─────────────────────────┐           │
│  │     PERP POSITIONS      │         │     PROTOCOL HEDGE      │           │
│  ├─────────────────────────┤         ├─────────────────────────┤           │
│  │  User A: +10 (long)     │         │                         │           │
│  │  User B: -5  (short)    │         │  H_spot = -Q_net = -5   │           │
│  │  User C: 0              │         │  (short 5 tokens B)     │           │
│  ├─────────────────────────┤         │                         │           │
│  │  Q_net = +5             │◄───────►│  Net exposure = 0       │           │
│  └─────────────────────────┘         └─────────────────────────┘           │
│                                                                             │
│  USER C OPENS LONG POSITION: Δq_m = +20                                    │
│  ══════════════════════════════════════════════════════════════════════    │
│                                                                             │
│       ┌─────────────────────────────────────────────────────────────┐      │
│       │                   ATOMIC OPERATION                          │      │
│       │         (all steps at logical time t₀)                      │      │
│       └─────────────────────────────────────────────────────────────┘      │
│                               │                                             │
│                               │                                             │
│           ┌───────────────────┼───────────────────┐                        │
│           │                   │                   │                        │
│           ▼                   ▼                   ▼                        │
│   ┌───────────────┐   ┌───────────────┐   ┌───────────────────────┐       │
│   │  STEP 1:      │   │  STEP 2:      │   │  STEP 3:              │       │
│   │  Update perp  │   │  Compute      │   │  Execute spot hedge   │       │
│   │  position     │   │  hedge delta  │   │  via CLMM + CLOB      │       │
│   │               │   │               │   │                       │       │
│   │  q_C: 0 → +20 │   │  ΔH_spot =    │   │  Sell 20 tokens B     │       │
│   │  Entry price: │   │  -ΔQ_net =    │   │  into the spot        │       │
│   │  P(l(t₀⁻))    │   │  -20          │   │  engine               │       │
│   └───────────────┘   └───────────────┘   └───────────────────────┘       │
│                                                   │                        │
│                                                   ▼                        │
│                                           Price may move:                  │
│                                           l(t₀⁻) → l(t₀⁺)                  │
│                                                                             │
│  AFTER TRADE (time t₀⁺):                                                   │
│  ══════════════════════════════════════════════════════════════════════    │
│                                                                             │
│  ┌─────────────────────────┐         ┌─────────────────────────┐           │
│  │     PERP POSITIONS      │         │     PROTOCOL HEDGE      │           │
│  ├─────────────────────────┤         ├─────────────────────────┤           │
│  │  User A: +10 (long)     │         │                         │           │
│  │  User B: -5  (short)    │         │  H_spot = -Q_net = -25  │           │
│  │  User C: +20 (long) NEW │         │  (short 25 tokens B)    │           │
│  ├─────────────────────────┤         │                         │           │
│  │  Q_net = +25            │◄───────►│  Net exposure = 0       │           │
│  └─────────────────────────┘         └─────────────────────────┘           │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  INVARIANT: H_spot(t) = -Q_net(t)  always maintained                │   │
│  │  RESULT: Protocol has zero directional exposure at all times        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

_Figure 11: An atomic hedged perp trade updates the user's perp position and the protocol's spot hedge in a single logical operation. When User C opens a +20 long position, the protocol simultaneously sells 20 tokens B into the spot engine, maintaining the invariant H_spot = -Q_net. The hedge trade executes through the same CLMM + CLOB engine as any other spot trade, potentially moving the price. The protocol remains delta-neutral at all times._

<!-- LaTeX/TikZ version for PDF:
\begin{figure}[h]
\centering
\begin{tikzpicture}
  % Before/after state boxes showing positions and hedge
  % Arrows showing the atomic update steps
  % Highlight invariant maintenance
\end{tikzpicture}
\caption{Atomic Hedged Perp Trade Mechanics}
\label{fig:atomic-hedged-perp-trade}
\end{figure}
-->

#### 8.3.3 Combined PnL of perps and hedge

Let the **perp layer PnL** be:

$$
\mathrm{PnL}^{\mathrm{perp}}(t) = \sum_{m\in \mathcal{M}} \big( q_m(t)\big( P(l(t)) - P_{\mathrm{entry},m}(t)\big) + F_m(t) \big)
$$

and the **hedge PnL** (in units of token $A$) from the protocol's spot hedge be denoted $\mathrm{PnL}^{\mathrm{hedge}}(t)$, computed from the sequence of spot trades $\{ \Delta H_{\mathrm{spot}}(\tau) \}$ executed along the spot price path $P(l(\tau))$.

Under continuous-time idealization and ignoring fees for the moment, if the hedge position is maintained continuously as

$$
H_{\mathrm{spot}}(t) = - Q_{\mathrm{net}}(t)
$$

then the infinitesimal combined PnL change obeys:

**Lemma 8.4 (Infinitesimal delta-neutrality).**

Assume:

- the protocol adjusts $H_{\mathrm{spot}}(t)$ continuously so that $H_{\mathrm{spot}}(t) = - Q_{\mathrm{net}}(t)$ for all $t$,
- all trades occur at the spot price $P(l(t))$ with no slippage or fees.

then, for any infinitesimal change in log-price $dl$ (equivalently in spot price $dP$), the combined PnL of perps + hedge satisfies:

$$
d\big( \mathrm{PnL}^{\mathrm{perp}}(t) + \mathrm{PnL}^{\mathrm{hedge}}(t) \big) = 0
$$

**_Proof Sketch._**

Under these assumptions, at time $t$, the perp layer's infinitesimal PnL change from a price move $dP$ is:

$$
d\mathrm{PnL}^{\mathrm{perp}}(t) = Q_{\mathrm{net}}(t) \, dP
$$

The hedge PnL change is:

$$
d\mathrm{PnL}^{\mathrm{hedge}}(t) = H_{\mathrm{spot}}(t) \, dP
$$

Under the delta-neutrality constraint $H_{\mathrm{spot}}(t) = - Q_{\mathrm{net}}(t)$, we have:

$$
d\mathrm{PnL}^{\mathrm{perp}}(t) + d\mathrm{PnL}^{\mathrm{hedge}}(t) = Q_{\mathrm{net}}(t)\, dP - Q_{\mathrm{net}}(t)\, dP = 0
$$

$\square$

> **Intuition:** Lemma 8.4 is the heart of the hedging strategy. It says: if you hold a portfolio that's long $X$ units of perp and simultaneously short $X$ units of spot, price movements don't affect your total value—gains on one side exactly cancel losses on the other. This is classic "delta hedging" from options theory [@black1973pricing; @merton1973theory], but applied to perpetuals. The protocol uses this to offload directional risk: every time someone opens a perp, the protocol immediately takes the opposite spot position, keeping itself market-neutral.

In discrete time with atomic hedged perp trades (Definition 8.3), the same property holds between hedge adjustments, and any residual risk arises only from re-hedging discretization and fees.

### 8.4 LP Exposure and Fee Sharing

We now formalize the statement that LPs in the spot engine do **not** take on additional directional risk from the derivative layer; they only interact with hedging and liquidation flow as ordinary spot flow.

#### 8.4.1 Order flow decomposition

Consider the spot engine viewed as a pure CLMM + CLOB market as in Sections 5-7. Let us conceptually decompose all spot trades over some time interval $[0, T]$ into:

- **exogenous spot flow**: trades originating from non-derivative users (ordinary swaps, spot limit orders),
- **derivative hedge flow**: trades originating from the protocol's spot hedge adjustments for perps, futures, or options (including liquidations).

Formally, we can write the **total net traded amount in token $B$** seen by the spot engine to time $T$ as:

$$
\Delta B_{\mathrm{total}}(0\to T) = \Delta B_{\mathrm{exo}}(0\to T) + \Delta B_{\mathrm{hedge}}(0\to T)
$$

where $\Delta B_{\mathrm{hedge}}$ is the signed sum of all hedge trades $\Delta H_{\mathrm{spot}}$ and liquidation-induced trades.

From the spot engine's perspective, there is no intrinsic distinction between these two sources; both are just sequences of market and limit orders executed via the unified CLMM + CLOB semantics.

#### 8.4.2 LP wealth as a functional of spot flow

Let $\mathcal{W}_{\mathrm{LP}}(T)$ denote the (vector of) LP wealth at time $T$, i.e. the values of all LP positions in tokens $A$ and $B$ plus accumulated fees, expressed in some numeraire. Under the CLMM + CLOB model, for a fixed path $l(t)$ and fixed sequences of spot trades hitting the pool, $\mathcal{W}_{\mathrm{LP}}(T)$ is completely determined by:

- the **initial LP positions** (band ranges, amounts, order placements),
- the **path of spot price** $l(t)$,
- the **sequence of spot trades** (sizes and directions) routed through the engine.

Importantly, the derivative layer does not introduce any new state variable into this dependence; it only changes the composition of spot order flow.

We can thus write:

$$
\mathcal{W}_{\mathrm{LP}}(T) = \Phi\big( l(\cdot), { \Delta B_{\mathrm{total}}(\tau)}\_{\tau\in [0,T]} \big)
$$

for some functional $\Phi$ induced by the CLMM + CLOB dynamics.

#### 8.4.3 No additional directional derivative risk for LPs

We now formalize that LPs do not "carry the other side" of perp positions.

Consider two worlds:

- **World A (With derivatives):**
  The system includes the spot-anchored perp layer with protocol-level hedging as in Section 8.3. The spot engine sees total order flow $\Delta B_{\mathrm{total}} = \Delta B_{\mathrm{exo}} + \Delta B_{\mathrm{hedge}}$.

- **World B (Without derivatives):**
  There is no perp layer; instead, the spot engine receives an exogenous sequence of spot trades whose net token $B$ flow matches $\Delta B_{\mathrm{total}}$ exactly.

**Proposition 8.5 (LP risk equivalence).**

Fix an initial CLMM + CLOB state and a spot price path $l(t)$. Suppose that in World A and World B the spot engine sees the **same total spot flow** $\Delta B_{\mathrm{total}}$ over $[0,T]$. Then, the LP wealth at time $T$ is identical in both worlds:

$$
\mathcal{W}_{\mathrm{LP}}^{A}(T) = \mathcal{W}_{\mathrm{LP}}^{B}(T)
$$

**_Proof Sketch._**

In both worlds, the CLMM + CLOB engine starts from the same initial state, sees the same sequence of spot trades (sizes and directions) and the same price path $l(t)$, governed by the unified execution rules. The engine does not distinguish whether a particular trade originated from a hedging operation, a liquidation, or an ordinary user. The update rules for LP positions and fees depend only on the sequence of trades and price path, which are identical by assumption. Therefore the resulting LP wealth functional $\Phi$ has the same arguments in both worlds, yielding $\mathcal{W}_{\mathrm{LP}}^{A}(T) = \mathcal{W}_{\mathrm{LP}}^{B}(T)$. $\square$

This shows that, relative to any given spot price path and aggregate order flow, LPs do not incur **additional directional risk** simply because a derivative layer exists; they are not systematically on the other side of perp bets. They only see more (or differently structured) spot flow and corresponding fee opportunities.

> **Intuition:** Proposition 8.5 answers a critical question: "Do LPs get burned by the derivatives layer?" The answer is no. From an LP's perspective, a trade is a trade—they don't know or care whether it came from someone swapping tokens, or from the protocol hedging a perpetual position. What matters is the total volume and direction. Since the protocol's hedges are executed as ordinary spot trades through the same AMM, LPs earn the same fees and face the same impermanent loss as they would from any other order flow of equal size. The derivatives layer gives LPs _more_ trading volume (and thus more fees) without changing their fundamental risk profile.

In particular, the protocol may allocate to LPs a **share of derivative-layer fees** (e.g. a fraction of perp open/close fees) as compensation for providing the underlying price discovery and liquidity, without implying that LPs bear the derivative layer's net exposure.

### 8.5 Margin, Liquidations, and Forced Flow

We briefly specialize the generic liquidation logic from Section 7.4 to the spot-anchored perp layer.

#### 8.5.1 Margin and liquidation condition

For each account $m$, given position $q_m(t)$ and equity $E_m(t)$ as in Section 8.2, we assume:

- an **initial margin** requirement $\mathrm{IM}_m(q_m)$,
- a **maintenance margin** requirement $\mathrm{MM}_m(q_m)$,

both nonnegative functions of position size and, possibly, other risk factors.

Account $m$ is eligible for liquidation at time $t$ when:

$$
E_m(t) \leq \mathrm{MM}_m(q_m)
$$

Using $P(l(t)) = 2^{2 l(t)}$, this defines for each $m$ a **liquidation region** in log-price space:

$$
\mathcal{L}_m = { l \in \mathbb{R} : M_m + q_m(2^{2l} - P_{\mathrm{entry},m}) + F_m \le \mathrm{MM}_m(q_m) }
$$

Under monotonicity of $P(l)$, this is typically an interval (e.g. for a long position, liquidation when $l$ falls below a critical level).

```text
           Equity E_m(l)
               │
               │         ╱
   IM ─────────┼────────╱────────────────────  Initial Margin
               │       ╱
               │      ╱
   MM ─────────┼─────╱───────────────────────  Maintenance Margin
               │    ╱
               │   ╱
    0 ─────────┼──╱──────────────────────────
               │ ╱
         ██████│╱                              Liquidation Zone
         ██████│                               (E_m ≤ MM)
         ██████│
        ───────┼──────────────────────────► log-price l
               │
          l_liq│   l_entry
               │
               ▼

    For a LONG position (q_m > 0):
    ┌─────────────────────────────────────────────────────┐
    │  E_m(l) = M_m + q_m·(P(l) - P_entry) + F_m          │
    │                                                     │
    │  • Equity increases as l increases (price rises)    │
    │  • Liquidation region: l < l_liq (shaded zone)      │
    │  • At l_liq: E_m = MM_m (liquidation threshold)     │
    │  • Position opened at l_entry with margin M_m       │
    └─────────────────────────────────────────────────────┘

    For a SHORT position (q_m < 0):
    ┌─────────────────────────────────────────────────────┐
    │  Equity DECREASES as l increases (price rises)      │
    │  Liquidation region: l > l_liq (high prices)        │
    └─────────────────────────────────────────────────────┘
```

_Figure 12:_ Margin thresholds and liquidation region for a long perpetual position. Equity $E_m(l)$ varies linearly with log-price $l$ (via $P(l) = 2^{2l}$). When equity falls below the maintenance margin (MM), the position enters the liquidation zone (shaded). Initial margin (IM) is required to open a position; MM is the threshold for forced closure.\_

<!--
% LaTeX/TikZ version for PDF rendering:
% \begin{figure}[h]
% \centering
% \begin{tikzpicture}[scale=0.9]
%   % Axes
%   \draw[->] (-1,0) -- (6,0) node[right] {$l$};
%   \draw[->] (0,-1) -- (0,5) node[above] {$E_m(l)$};
%
%   % Equity line for long position
%   \draw[thick, blue] (-0.5,-0.5) -- (5,4) node[right] {$E_m(l)$};
%
%   % Margin levels
%   \draw[dashed, red] (-0.5,3.5) -- (5.5,3.5) node[right] {IM};
%   \draw[dashed, orange] (-0.5,2) -- (5.5,2) node[right] {MM};
%   \draw[dotted] (-0.5,0) -- (5.5,0);
%
%   % Liquidation zone (shaded)
%   \fill[red!20] (-0.5,-1) rectangle (1.5,2);
%   \node at (0.5,0.5) {\small Liq.};
%
%   % Critical points
%   \draw[dashed] (1.5,0) -- (1.5,2);
%   \node[below] at (1.5,0) {$l_{\text{liq}}$};
%   \draw[dashed] (3,0) -- (3,2.5);
%   \node[below] at (3,0) {$l_{\text{entry}}$};
%
%   % Annotations
%   \node[align=left] at (4,-1.5) {\small Long: $E_m \uparrow$ as $l \uparrow$};
% \end{tikzpicture}
% \caption{Margin thresholds and liquidation region for perpetual positions.}
% \end{figure}
-->

#### 8.5.2 Liquidation as aggressive hedged trades

When the liquidation condition is met for an account $m$, the protocol may initiate a **forced close** of all or part of $q_m$. In the simplest case, the protocol closes the full position size $q_m$ via:

1. An aggressive perp trade $\Delta q_m = - q_m$,
2. An equal and opposite hedge update $\Delta H_{\mathrm{spot}} = - \Delta Q_{\mathrm{net}} = q_m$,
3. Execution of $\Delta H_{\mathrm{spot}}$ as an aggressive spot trade via the unified CLMM + CLOB engine.

By Definition 8.3, this is an atomic hedged perp trade. The resulting spot price path during liquidation is determined entirely by the existing spot liquidity and the forced trade size; no additional liquidation-specific microstructure is needed at the derivative layer.

This reinforces the view that liquidations are **just another source of aggressive spot order flow** along the log-price axis.

### 8.6 Extension to Expiring Futures and Options

The same approach extends naturally to expiring futures and options whose payoffs are functions of the log-price at expiry.

#### 8.6.1 Expiring futures

An expiring future on $(A, B)$ with expiry $T$ and strike $K$ (in units of $A$) is defined by payoff:

$$
\mathrm{Payoff}_m^{\mathrm{fut}} = q_m \big( P(l(T)) - K \big)
$$

with mark-to-market equity updated continuously against the spot price $P(l(t))$, for example via standard futures margining:

$$
E_m(t) = M_m(t) + q_m(t)\big( P(l(t)) - P_{\mathrm{ref}}(t) \big) + F_m(t)
$$

where $P_{\mathrm{ref}}(t)$ is an appropriate reference (e.g. prior settlement).

The protocol may maintain delta-neutrality in the same way as for perps, by setting:

$$
H_{\mathrm{spot}}(t) = - Q_{\mathrm{net}}^{\mathrm{fut}}(t)
$$

and executing hedge adjustments via the spot engine.

#### 8.6.2 European options

A European option with expiry $T$ and payoff $\Phi(P(l(T)))$ (e.g. \max{P(l(T)) - K, 0} for a call) can be represented as a position $\pi_m$ with terminal payoff:

$$
\mathrm{Payoff}_m^{\mathrm{opt}} = \pi_m \cdot \Phi(P(l(T)))
$$

The protocol may choose to:

- remain unhedged (warehousing option risk),
- or implement **delta-hedging** strategies based on the derivative of $\Phi$ with respect to $P$, i.e. using the Greeks ($\Delta$, $\Gamma$, etc.) from classical option pricing theory [@black1973pricing; @merton1973theory] computed against the same spot process $P(l(t))$.

In the latter case, each hedging update is again just a spot trade via the CLMM + CLOB engine, and the analysis of LP risk exposure carries over.

### 8.7 Summary

This section has defined a **spot-anchored derivative layer** in which:

- all derivatives (perps, futures, options) are priced and margined exclusively against the spot price $P(l(t)) = 2^{2 l(t)}$ on the log-price axis,
- there is no independent derivative price curve and hence no basis or funding term is needed,
- the protocol maintains delta-neutrality by hedging net derivative exposure via the same unified CLMM + CLOB engine, executing hedges as ordinary spot trades,
- LPs in the spot layer do not warehouse net derivative directional risk; from their perspective, derivative activity appears only as additional spot flow, and their wealth depends solely on the spot path and aggregate order flow,
- liquidations are realized as forced aggressive trades along the log-price axis, requiring no special matching mechanism beyond the existing execution semantics.

This establishes that the three-layer log-domain spot architecture (Sections 5-7) naturally supports a fourth, derivative layer that is economically coherent, platform-agnostic, and structurally closer to traditional centralized exchanges (single unified price, shared liquidity, atomic hedging) than existing on-chain perpetual and options designs [@bitmex2016perpetual; @dydx2023v4; @gmx2022technical; @perp2021v2; @drift2023v2; @synthetix2020litepaper].

## 9. Formal Guarantees of the Derivative Layer

In this section, we formalize the stochastic structure, no-arbitrage properties, residual hedging risk, execution validity, fee invariants, and computational complexity of the spot-anchored derivative layer introduced in Section 8. Throughout, we assume the three-layer log-domain spot architecture of Section 5-7 and the spot-anchored perpetual construction of Section 8.

Unless otherwise stated, all quantities are expressed in units of token $A$.

### 9.1 Log-Price Process as a Stochastic Object

We begin by formalizing the log-price process $l(t)$ generated by the unified CLMM + CLOB engine.

#### 9.1.1 Probability space and filtration

Let $(\Omega, \mathcal{F}, \mathbb{P})$ be a probability space endowed with a filtration $\{ \mathcal{F}_t \}_{t \ge 0}$ satisfying the usual conditions (right-continuity, completeness) [@shreve2004stochastic]. We view:

- **Exogenous order flow** (external spot orders, user perp positions, etc.) as a collection of adapted processes $\Xi(t)$ taking values in a suitable state space $\mathcal{X}$ of order streams,
- **Oracle inputs** (if any, e.g., volatility feeds) as adapted processes $\mathcal{O}(t)$.

We assume that the CLMM + CLOB engine and derivative layer define a measurable **execution operator**:

$$
\mathcal{E}: (\Xi(\cdot), \mathcal{O}(\cdot)) \mapsto (l(\cdot), \mathcal{S}(\cdot))
$$

where $\mathcal{S}(t)$ denotes the full on-chain state at time $t$ (pool reserves, order books, positions, margins, etc.).

#### 9.1.2 Log-price path regularity

We model the log-price path $l(t)$ as:

- **adapted**: $l(t)$ is $\mathcal{F}_t$-measurable for all $t$,
- **càdlàg**: right-continuous with left limits [@shreve2004stochastic],
- **piecewise monotone** between trade events.

Intuitively:

- between executions, $l(t)$ is constant;
- when a (possibly aggregated) trade executes, $l(t)$ moves monotonically along the log-axis according to the CLMM + CLOB execution semantics of Section 7, possibly crossing finitely many CLOB nodes, and mesh segments;
- at each trade time $\tau$, we allow $l(t)$ to have a jump discontinuity at $\tau$, corresponding to the end-point of executed path.

For convenience, we collect these properties.

**Assumption 9.1 (Execution-induced price path).**

The unified CLMM + CLOB execution engine induces a log-price process $l(t)_{t \ge 0}$ such that:

1. $l(t)$ is ($\mathcal{F}_t$)-adapted and càdlàg,
2. There exists a countable (locally finite) set of trade times ${ \tau_n }_{n \in \mathbb{N}}$ with $\tau_n \uparrow \infty$, such that:

   - on each interval $(\tau_n, \tau_{n+1})$, $l(t)$ is constant,
   - at each $\tau_n$, $l(t)$ moves along a piecewise-monotone path in $l$-space determined by the corresponding order flow in $\Xi(\tau_n)$, with finitely many node crossings.

This is consistent with the Layer-2 mesh semantics of Section 5.3 and the per-trade execution rules of Section 7.

### 9.2 No-Arbitrage and Uniqueness of the Spot-Anchored Mark Price

We now show that, in the presence of frictionless trading in the spot engine, the spot-anchored mark price used in Section 8 is the unique choice compatible with absence of static arbitrage between spot and perps.

#### 9.2.1 Setup

Fix a time $t$ and suppress time subscripts for clarity. Let:

- $P_{\mathrm{spot}} := P(l(t)) = 2^{2 l(t)}$ be the spot price determined by the unified CLMM + CLOB engine,
- $P_{\mathrm{mark}}$ be the **perp mark price** used to value and margin perp positions at that time.

We assume the following:

- Traders can open or close perp positions at execution prices equal (up to small fees and slippage, which we momentarily ignore) to the current mark price $P_{\mathrm{mark}}$,
- Traders can trade the underlying spot via the CLMM + CLOB engine at the prevailing spot price $P_{\mathrm{spot}}$, again ignoring small bid-ask spreads for the arbitrage argument.
- Perp PnL is settled (continuously or discretely) against the mark price.

#### 9.2.2 Arbitrage construction for misaligned marks

We now formalize the intuitive statement that $P_{\mathrm{mark}} \neq P_{\mathrm{spot}}$ allows an instantaneous arbitrage.

**Proposition 9.2 (Uniqueness of spot-anchored mark price).**

Suppose the markets support frictionless trading as above and that the perp mark price $P_{\mathrm{mark}}$ is used both for execution and for PnL/margin. If $P_{\mathrm{mark}} \neq P_{\mathrm{spot}}$, then there exists a self-financing trading strategy with non-negative initial cost and strictly positive terminal wealth with positive probability. In particular, absence of such static arbitrage implies:

$$
P_{\mathrm{mark}} = P_{\mathrm{spot}} = P(l(t))
$$

**_Proof Sketch._**

Consider two cases.

**Case 1: $P_{\mathrm{mark}} > P_{\mathrm{spot}}$.**

Consider an agent who:

1. At time $t$, **shorts** one unit of the perp at price $P_{\mathrm{mark}}$. This yields an effective short exposure of (-1) in units of $B$, with mark-to-market at $P_{\mathrm{mark}}$.
2. Simultaneously, **buys** one unit of the underlying via the spot engine at price $P_{\mathrm{spot}}$, paying $P_{\mathrm{spot}}$ units of $A$.

Net initial cash flow:

- From step 1: effectively zero net cash if we assume symmetric margin (or positive if initial margin is posted in a separate account and not fully funded).
- From step 2: $- P_{\mathrm{spot}}$

Now consider an instantaneous re-marketing at the same time $t$. The perp short's PnL, when marking at $P_{\mathrm{mark}}$, is zero at inception; however, the **economic value** of the position relative to the spot hedge is:

- The agent is short "one unit of $B$ at $P_{\mathrm{mark}}$" and long one unit of $B$ at $P_{\mathrm{spot}}$. If they close both legs at spot price $P_{\mathrm{spot}}$, they:

  - buy back the perp exposure (by going long one unit) at the mark $P_{\mathrm{mark}}$,
  - sell their spot unit at $P_{\mathrm{spot}}$.

At closure, the combined payoff (ignoring margin mechanics) is:

$$
\Pi = P_{\mathrm{mark}} - P_{\mathrm{spot}} > 0
$$

since they synthetically sold at $P_{\mathrm{mark}}$ and bought at $P_{\mathrm{spot}}$. Thus, a round-trip strategy yields strictly positive profit with zero net risk.

**Case 2: $P_{\mathrm{mark}} < P_{\mathrm{spot}}$.**

The agent reverses the strategy:

1. **Long** one unit of perp at $P_{\mathrm{mark}}$,
2. **Short** one unit of spot at $P_{\mathrm{spot}}$.

Closing at spot yields payoff:

$$
\Pi = P_{\mathrm{spot}} - P_{\mathrm{mark}} > 0
$$

In both cases, an instantaneous arbitrage exists if $P_{\mathrm{mark}} \neq P_{\mathrm{spot}}$. Therefore, in any model excluding such arbitrage, we must have:

$$
P_{\mathrm{mark}} = P_{\mathrm{spot}} = P(l(t))
$$

$\square$

In Section 8, we set $P_{\mathrm{mark}} := P(l(t))$; Proposition 9.2 shows this is not merely a design choice but the **unique** no-arbitrage prescription under frictionless trading between spot and perps.

> **Intuition:** Proposition 9.2 says that if the perpetual's mark price ever drifts away from the spot price, even briefly, traders can instantly profit by going long on the cheaper market and short on the more expensive one. Since both positions have the same underlying exposure, the mismatch creates free money. The only way to prevent this is to lock the mark price to the spot price at all times—which is exactly what the spot-anchored design does.

### 9.3 Residual Hedging Risk Under Discrete Execution

In Section 8.3, we idealized hedging as continuous and exact. We now quantify the error introduced by **discrete hedging** and finite mesh resolution.

#### 9.3.1 Discrete hedging schedule

Let $\{ \tau_n \}_{n \in \mathbb{N}}$ be the (possibly random) times at which the protocol updates its spot hedge for perps, with $0 = \tau_0 < \tau_1 < \tau_2 < \cdots \leq T$. On each interval $(\tau_n, \tau_{n+1})$, the protocol maintains a hedge position:

$$
H_{\mathrm{spot}}(t) = H_n, \quad t \in [\tau_n,\tau_{n+1})
$$

while the net perp exposure $Q_{\mathrm{net}}(t)$ may vary due to user trades.

Define the **hedging error** at time $T$ as:

$$
\mathrm{HE}(T) := \mathrm{PnL}^{\mathrm{perp}}(T) + \mathrm{PnL}^{\mathrm{hedge}}(T)
$$

i.e. the combined PnL of perps and hedge. In the continuous ideal from Lemma 8.4, we have $\mathrm{HE}(T) = 0$; here we bound $\mathrm{HE}(T)$ in the discrete case.

For analytic convenience, we work at the level of infinitesimal dynamics. Let $P(t) = P(l(t))$ and suppose $P$ is of finite variation on $[0,T]$ (this is natural under the trade-driven execution model of Section 9.1). Then the combined PnL can be written as:

$$
\mathrm{HE}(T)
= \int_0^T Q_{\mathrm{net}}(t)\, dP(t) + \int_0^T H_{\mathrm{spot}}(t)\, dP(t)
$$

Under continuous hedging with $H_{\mathrm{spot}}(t) = - Q_{\mathrm{net}}(t)$, we have $\mathrm{HE}(T) = 0$. Under discrete hedging, we instead have piecewise constant $H_n$.

#### 9.3.2 Hedging error decomposition

On each interval $[\tau_n, \tau_{n+1})$, we have:

$$
Q_{\mathrm{net}}(t) + H_{\mathrm{spot}}(t) = Q_{\mathrm{net}}(t) + H_n
$$

Introduce the **exposure mismatch**:

$$
\Delta_n(t) := Q_{\mathrm{net}}(t) + H_n
$$

so that:

$$
\mathrm{HE}(T) = \sum_{n=0}^{N-1} \int_{\tau_n}^{\tau_{n+1}} \Delta_n(t) \, dP(t)
$$

We now bound this sum under mild regularity assumptions.

#### 9.3.3 Lipschitz continuity in log-space

From the log-price mapping $P(l) = 2^{2 l}$, we have:

$$
\frac{dP}{dl} = 2^{2 l} \cdot 2 \ln(2) = 2 \ln(2) P(l)
$$

so on any bounded log-price interval $[l_{\min}, l_{\max}]$, we obtain the Lipschitz bound:

$$
|P(l_2) - P(l_1)| \le L_P |l_2 - l_1|, \quad L_P := 2 (\ln 2) \sup_{l \in [l_{\min}, l_{\max}]} P(l)
$$

From the mesh convergence and fixed-point error analysis in Section 6, we know that:

- the numerical log-price $\hat l(t)$ deviates from the ideal log-price by at most $O(1/\sigma + \varepsilon_{\exp_2} + \varepsilon_{\mathrm{arith}})$,
- and the mesh step $\delta = \max_k (l_{k+1} - l_k)$ is chosen small.

We now aggregate these into a single **effective log-resolution** $\bar\delta$, defined as:

$$
\bar\delta := \delta + C_{\mathrm{num}} \Big( \tfrac{1}{\sigma} + \varepsilon_{\exp_2} + \varepsilon_{\log_2} + \varepsilon_{\mathrm{arith}} \Big)
$$

for a constant $C_{\mathrm{num}}>0$ depending only on the bounds of Section 6.6.

#### 9.3.4 Residual risk bound

Assume:

- the log-price $l(t)$ remains in a bounded interval $[l_{\min}, l_{\max}]$ on $[0,T]$,
- on each hedging interval $[\tau_n, \tau_{n+1})$, the mismatch $\Delta_n(t)$ is uniformly bounded:

$$
|\Delta_n(t)| \leq \Delta_{\max}, \quad \forall t\in[\tau_n,\tau_{n+1}), \forall n
$$

- the total variation of $l(t)$ is bounded by $V_l$, i.e.

$$
\mathrm{TV}_0^T(l) := \sup_{\Pi} \sum_{i} |l(t_{i+1}) - l(t_i)| \leq V_l
$$

where the supremum is over partitions $\Pi$.

We can now state the main bound.

**Theorem 9.3 (Residual hedging error under discrete hedging).**

Under the assumptions above, the absolute combined PnL of perps and hedge satisfies:

$$
|\mathrm{HE}(T)| \le \Delta_{\max} \cdot L_P \cdot \bar\delta \cdot N_{\mathrm{eff}}
$$

where

- $L_P = 2 (\ln 2) \sup_{l \in [l_{\min}, l_{\max}]} P(l)$ is the Lipschitz constant of $P(l)$,
- $\bar\delta$ is the effective log-resolution as defined above,
- $N_{\mathrm{eff}} \leq \mathrm{TV}_0^T(l) / \bar\delta$ is an upper bound on the effective number of log-steps.

Equivalently, there exists a constant $C>0$, depending only on $V_l$ and the price bounds, such that:

$$
|\mathrm{HE}(T)| \le C \cdot \Delta_{\max} \Big( \delta + \tfrac{1}{\sigma} + \varepsilon_{\exp_2} + \varepsilon_{\log_2} + \varepsilon_{\mathrm{arith}} \Big)
$$

**_Proof Sketch._**

On each interval $[\tau_n, \tau_{n+1})$:

$$
\left|\int_{\tau_n}^{\tau_{n+1}} \Delta_n(t)\, dP(t)\right|
\le \sup_{t\in[\tau_n,\tau_{n+1})} |\Delta_n(t)| \cdot \mathrm{TV}_{\tau_n}^{\tau_{n+1}}(P)
$$

where $\mathrm{TV}_{a}^{b}(P)$ denotes total variation of $P$ on $[a,b]$. Using $|\Delta_n(t)| \leq \Delta_{\max}$ and the Lipschitz bound for $P$ in terms of $l$, we get:

$$
\mathrm{TV}_{\tau_n}^{\tau_{n+1}}(P) \le L_P \cdot \mathrm{TV}_{\tau_n}^{\tau_{n+1}}(l)
$$

From the mesh and numerical discretization, each effective price move on the mesh is at most $\bar\delta$ in log-space; thus the total number of effective steps required to achieve the total variation $V_l$ is at most $N_{\mathrm{eff}} \leq V_l / \bar\delta$. Summing over all intervals:

$$
|\mathrm{HE}(T)|
\le \sum_n \Delta_{\max} L_P \mathrm{TV}_{\tau_n}^{\tau_{n+1}}(l)
\le \Delta_{\max} L_P \cdot V_l
\le \Delta_{\max} L_P \bar\delta \cdot \frac{V_l}{\bar\delta}
\le \Delta_{\max} L_P \bar\delta N_{\mathrm{eff}}
$$

The final inequality with constant $C$ is obtained by absorbing $L_P V_l$ into $C$ and substituting the expression for $\bar\delta$. $\square$

> **Intuition:** Theorem 9.3 quantifies "how much money could the protocol lose" by not hedging continuously. The answer depends on (1) how badly out-of-sync the hedge gets ($\Delta_{\max}$), (2) how granular the price mesh is ($\bar\delta$), and (3) how volatile the market is ($V_l$). The bound says: if you hedge frequently enough and use fine enough price resolution, the accumulated error stays small. In practice, this means the protocol can tune mesh granularity and hedge frequency to achieve any desired level of hedging accuracy—at the cost of more computation.

Thus, residual hedging risk can be made arbitrarily small by refining:

- the mesh resolution $\delta$,
- the fixed-point scaling $\sigma$,
- and the numerical routines $\exp_2, \log_2$,

consistent with the convergence results of Theorem 6.8.

### 9.4 Liquidity Stress and Execution Validity Under Hedge Flow

We now show that large hedge trades, routed through the spot engine, respect the structural invariants of the CLMM + CLOB execution model: monotone price movement, non-negative reserves, and node-by-node traversal.

We work under the execution semantics of Section 7 (CLOB at nodes, CLMM along segments), extended to include hedge-originated market orders.

**Lemma 9.4 (Monotone execution under large market orders).**

Let a hedge trade of size $\Delta B$ (in token $B$, sign included) be executed as an aggressive order through the CLMM + CLOB engine at current log-price $l_*$. Assume:

1. The total available liquidity (CLOB + CLMM) in the direction of the trade is strictly positive on the relevant log-interval $[l_{\min}, l_{\max}]$,
2. The per-segment CLMM formulas (Section 5.5) and node execution ordering (Section 7.1) are observed.

Then the resulting log-price path $l(\cdot)$ during the execution of this market order is **piecewise monotone**, and each mesh node or CLOB level is crossed at most once.

**_Proof Sketch._**

Within a CLMM segment $[l_k, l_{k+1}]$ with liquidity $L_k > 0$, the per-segment execution formulas (Section 5.5.1) express token deltas as monotone functions of $l$; for a fixed trade direction, $l$ moves monotonically until either the segment liquidity is exhausted or the requested quantity is filled. At mesh nodes, the node execution ordering processes CLOB orders at that node at a single price level, without reversing direction. The composition of these monotone moves - segments followed by nodes followed by segments - yields a piecewise monotone path. Since each segment or node is crossed only while moving in a single direction, it cannot be revisited within the execution of the same market order. $\square$

**Lemma 9.5 (Reserve non-negativity under hedge flow).**

Assume that for all CLMM segments and bands, the initial reserves and liquidity satisfy the conditions for the standard CLMM formulas (Section 5.4). Let a hedge trade of size $\Delta B$ execute as above, and suppose that the available liquidity is sufficient to absorb $\Delta B$ without exhausting the pool. Then, at every intermediate step of execution, pool reserves remain non-negative.

**_Proof Sketch._**

On each segment, the CLMM update rules are identical to those of a standard constant-liquidity region, which preserve non-negative provided the segment is not over-traded. At CLOB nodes, reserve changes arise from matching against limit orders whose settlement is accounted for in the pool and user balances; these updates are merely transfers and cannot create negative reserves provided sufficient balances exist. Global sufficiency of liquidity ensures that no segment or aggregate reserve is over-drawn. $\square$

Together, Lemmas 9.4-9.5 show that:

- even under large hedging flow,
- the spot engine executes trades along valid, monotone paths,
- without violating invariant constraints.

In particular, hedging cannot "break" the spot engine; it behaves exactly as an ordinary large market order.

### 9.5 Liquidation Correctness on the Log-Price Axis

We now formalize that liquidations in the perp layer (Section 8.5) correspond to valid aggressive trades in spot and do not introduce anomalous behavior such as price flicker or inconsistent execution.

#### 9.5.1 Liquidation events as structured orders

Fix an account $m$ and suppose that at time $t_0$ its equity falls below maintenance margin, triggering liquidation. We consider a simple full liquidation policy: the protocol submits a **liquidation instruction** to:

- close a fraction $\theta \in (0,1]$ of the perp position $q_m(t_0)$,
- adjust the global hedge $H_{\mathrm{spot}}$ accordingly,
- execute the corresponding spot trade via the CLMM + CLOB engine.

As in Section 8.5, this can be viewed as an atomic hedged trade of size $\Delta q_m = - \theta q_m(t_0)$.

#### 9.5.2 Price path and fairness

We define **fair liquidation price** informally as:

- the (volume-weighted) average spot execution price along the actual price path generated by the liquidation trade, given the current liquidity.

More precisely, if the liquidation trade executes over a log-interval $[l_{\mathrm{start}}, l_{\mathrm{end}}]$ and consumes both CLOB and CLMM liquidity according to the execution rules, then the realized average price is:

$$
\bar P_{\mathrm{liq}} = \frac{ \text{total cost in token } A }{ \text{total quantity in token } B }
$$

We now state correctness.

**Theorem 9.6 (Liquidation correctness and path regularity).**

Under the execution semantics of Section 7 and Assumptions 9.1, any liquidation event implemented as an atomic hedged trade (Section 8.5) satisfies:

1. The liquidation's spot leg executes along a piecewise monotone log-price path $l(\cdot)$ with at most finitely many node crossings (Lemma 9.4),
2. Pool reserves remain non-negative throughout execution (Lemma 9.5),
3. The realized liquidation price $\bar P_{\mathrm{liq}}$ is exactly the volume-weighted average price over the path, computed from the CLMM + CLOB liquidity actually consumed.
4. There is no "flicker" price: the observable spot price just after liquidation, $P(l(t_0^+))$, is the terminal spot price after the final segment of the path; there is no instantaneous reversion.

**_Proof Sketch._**

(1) and (2) follow directly from Lemmas 9.4-9.5, applied to the liquidation's spot leg. (3) is by construction: the liquidation trade is executed using the same accounting as any spot market order, so the realized price is by definition the ratio of total token $A$ spent to token $B$ obtained (or vice versa). (4) follows from Assumption 9.1: $l(t)$ is updated by the execution operator to the final value of the path and remains at this value after the trade; there is no independent post-trade re-marking that could change the price discontinuously. $\square$

Thus, liquidations are economically indistinguishable from large market orders initiated by an external trader, except that they are triggered by margin conditions rather than explicit user intent.

### 9.6 Fee Allocation and Wealth Invariants

We now formalize the fee model and show that LP fee sharing is compatible with wealth conservation and the absence of hidden directional risk transfer.

#### 9.6.1 Fee functional

Let $\mathcal{E}_t$ denote the set of execution events (spot trades, perp opens/closes, liquidations) at time $t$. We define a **fee functional**:

$$
\mathcal{F}: \mathcal{E}_t \mapsto \mathbb{R}_{\ge 0}^3
$$

which maps each event $e \in \mathcal{E}_t$ to a triple:

$$
\mathcal{F}(e) = \big( f_{\mathrm{LP}}(e), f_{\mathrm{proto}}(e), f_{\mathrm{ins}}(e) \big)
$$

corresponding to:

- $f_{\mathrm{LP}}(e)$: fee paid to liquidity providers (spot LPs),
- $f_{\mathrm{proto}}(e)$: protocol revenue,
- $f_{\mathrm{ins}}(e)$: contributions to an insurance/reserve fund.

We assume non-negativity:

$$
f_{\mathrm{LP}}(e), f_{\mathrm{proto}}(e), f_{\mathrm{ins}}(e) \ge 0
$$

and a simple proportional model, e.g.:

- for a spot trade of notional $V_e$ in token $A$:

  $$
  f_{\mathrm{spot}}(e) = \phi_{\mathrm{spot}} V_e, \quad f_{\mathrm{LP}}(e) = \alpha_{\mathrm{spot}} f_{\mathrm{spot}}(e)
  $$

  $$
  f_{\mathrm{proto}}(e) = \beta_{\mathrm{spot}} f_{\mathrm{spot}}(e), \quad f_{\mathrm{ins}}(e) = \gamma_{\mathrm{spot}} f_{\mathrm{spot}}(e)
  $$

  with $\alpha_{\mathrm{spot}} + \beta_{\mathrm{spot}} + \gamma_{\mathrm{spot}} = 1$,

- for a perp open/close of notional $V_e^{\mathrm{perp}}$:

  $$
  f_{\mathrm{perp}}(e) = \phi_{\mathrm{perp}} V_e^{\mathrm{perp}}, \quad f_{\mathrm{LP}}(e) = \alpha_{\mathrm{perp}} f_{\mathrm{perp}}(e)
  $$

  and similarly for $f_{\mathrm{proto}}, f_{\mathrm{ins}}$.

The exact values of $\phi_{\mathrm{spot}}, \phi_{\mathrm{perp}}, \alpha, \beta, \gamma$ are protocol parameters.

#### 9.6.2 Wealth accounting

Let:

- $\mathcal{W}_{\mathrm{LP}}(t)$ be the aggregate wealth of all LPs,
- $\mathcal{W}_{\mathrm{tr}}(t)$ be the aggregate wealth of all traders (spot + derivatives),
- $\mathcal{W}_{\mathrm{proto}}(t)$ be the protocol-held wealth (treasury, fee balances),
- $\mathcal{W}_{\mathrm{ins}}(t)$ be the insurance fund wealth.

We define the **total internal wealth**:

$$
\mathcal{W}_{\mathrm{tot}}(t) = \mathcal{W}_{\mathrm{LP}}(t) + \mathcal{W}_{\mathrm{tr}}(t) + \mathcal{W}_{\mathrm{proto}}(t) + \mathcal{W}_{\mathrm{ins}}(t)
$$

And treat any external deposits/withdrawals or cross-asset conversions as exogenous.

**Proposition 9.7 (Internal wealth conservation with fees).**

Ignoring external deposits/withdrawals and valuation changes in reference numeraire, the total internal wealth satisfies:

$$
\mathcal{W}_{\mathrm{tot}}(T) = \mathcal{W}_{\mathrm{tot}}(0) - \sum_{e \in \mathcal{E}[0,T]} f_{\mathrm{slip}}(e)
$$

where $f_{\mathrm{slip}}(e) \geq 0$ denotes price impact/slippage losses incurred by traders relative to a hypothetical frictionless execution at mid-price. In particular, fee allocation via $\mathcal{F}$ is a pure redistribution internal to $\mathcal{W}_{\mathrm{tot}}$; it does not create or destroy wealth.

**_Proof Sketch._**

Each execution event $e$ consists of a set of transfers of tokens $A, B$ between participants (traders, LPs, protocol, insurance fund). By construction:

- $\sum$ of balances over all internal accounts changes only by:

  - execution price deviations from mid (slippage / impact),
  - plus any external inflow/outflow.

Fees $f_{\mathrm{LP}}(e), f_{\mathrm{proto}}(e), f_{\mathrm{ins}}(e)$ are accounted as explicit transfers from traders' wealth to LPs/protocol/insurance. Hence, they cancel out in the sum $\mathcal{W}_{\mathrm{tot}}$. The only net loss relative to an ideal frictionless setting is due to slippage relative to mid, captured by $\sum_e f_{\mathrm{slip}}(e)$. $\square$

Combined with Proposition 8.5, this shows:

- LPs can be rewarded with a share of perp fees $\alpha_{\mathrm{perp}} f_{\mathrm{perp}}(e)$,
- without implying that they carry the net directional exposure of the derivative layer.

Their additional revenue is justified by their role in providing the shared spot liquidity through which hedges and liquidations are executed.

### 9.7 Computational Complexity of Atomic Hedged Perp Trades

We now sketch an asymptotic bound on the computational complexity of atomic hedged perp trades on top of the three-layer log-domain architecture.

#### 9.7.1 Data structures

Recall:

- Layer 1 defines a global slot lattice indexed by integers $s \in \mathbb{Z}$ with anchors $l_s = l_{\mathrm{ref}} + s \Delta l$,
- each slot stores:

  - a list $\mathcal{I}_s$ of LP bands overlapping the slot (Section 5.2.3),
  - a list $\mathcal{J}_s^{\mathrm{buy}}, \mathcal{J}_s^{\mathrm{sell}}$ of limit orders nearest to that slot (Section 5.2.4),

- Layer 2 constructs a local mesh $\mathcal{M} = \{l_0 < \ldots < l_N\}$ and per-segment liquidity $L_{AMM,k}$ over an interval $I_{\mathrm{path}}$ (Section 5.3).

We assume slot indices, CLOB queues, and LP sets are stored in data structures supporting:

- $O(1)$ or $O(\log S)$ access to the current slot index given $l$,
- $O(\log | \mathcal{J}_s|)$ access to best price orders in each slot,
- iteration over neighboring slots in amortized constant time.

#### 9.7.2 Complexity bound

Consider a single atomic hedged perp trade (Definition 8.3) of notional size $V$. It consists of:

1. Internal perp accounting (update $q_m, P_{\mathrm{entry}, m}, M_m$) - $O(1)$,
2. Computation of hedge size $\Delta H_{\mathrm{spot}}$ - $O(1)$,
3. Execution of an aggressive spot trade of size $\Delta H_{\mathrm{spot}}$ through the CLMM + CLOB engine.

We focus on step (3). Let:

- $N$ be the number of mesh nodes in the relevant price interval $I_{\mathrm{path}}$,
- $M$ be the number of LP bands intersecting $I_{\mathrm{path}}$,
- $J$ be the number of active limit orders in $I_{\mathrm{path}}$.

Under the monotone execution semantics (Section 7 and Lemma 9.4), the hedge trade traverses a **contiguous sub-path** of the mesh nodes, crossing at most $K$ segments and nodes, where $K$ is proportional to the log-price range induced by $\Delta H_{\mathrm{spot}}$ and inversely proportional to local liquidity density. In particular, for a given slippage tolerance and liquidity profile, we can bound $K$ by a protocol parameter $K_{\max}$.

**Theorem 9.8 (Amortized complexity of atomic hedged perp trades).**

Let a single atomic hedged perp trade induce a hedge of size $\Delta H_{\mathrm{spot}}$, resulting in traversal of at most $K$ mesh segments and nodes. Assume:

- slot index lookup for the starting node is $O(1)$ or $O(\log S)$,
- per-node access to best CLOB orders and LP lists is $O(1)$ amortized (e.g. heap or linked list structures).

Then the amortized time complexity of the spot execution step is:

$$
T_{\mathrm{exec}} = O(K + \log S)
$$

and under a protocol-chosen bound $K \leq K_{\max}$, this is $O(\log S)$ per atomic hedged perp trade.

**_Proof Sketch._**

Execution proceeds by:

1. Locating the starting slot or node for $l_*$: $O(1)$ or $O(\log S)$,
2. At each node:

   - matching against at most a constant number of best-price orders (orderbook top of book can be maintained in $O(1)$ amortized),
   - computing the maximum CLMM capacity along the next segment (constant-time arithmetic using precomputed $L_{AMM,k}$),
   - moving to the next node in the direction of price movement.

Since the path is monotone and traverses at most $K$ segments/nodes, and each step is $O(1)$ amortized, the total cost is $O(K) + O(\log S)$. Bounding $K$ by a protocol parameter $K_{\max}$ yields the final claim. $\square$

This suggests that, for reasonable bounds on per-trade slippage and path length, the cost of integrating atomic hedged perp trades into the CLMM + CLOB engine is essentially the same asymptotic order as ordinary large spot market orders, and scales logarithmically with global slot count.

### 9.8 Funding-Rate Elimination in Spot-Anchored Perpetuals

We now formalize the intuition that, in the spot-anchored construction of Section 8, **funding rates are not needed** to keep perps aligned with spot, and in fact the _fair_ funding rate is identically zero.

#### 9.8.1 Stylized continuous-time model

Work on the filtered probability space $(\Omega, \mathcal{F}, \{ \mathcal{F}_t \}_{t \ge 0}, \mathbb{P})$ of Section 9.1. Let

- $P(t) := P(l(t))$ be the spot price of token $B$ in units of $A$,
- $B_t$ be the risk-free bank account in token $A$, with dynamics:

$$
dB_t = r_t B_t \, dt
$$

where $r_t$ is a (bounded) short-rate process, adapted to $(\mathcal{F}_t)$.

Assume that under the physical measure $\mathbb{P}$, the spot follows an Itô process [@shreve2004stochastic]:

$$
dP_t = \mu_t P_t \, dt + \sigma_t P_t \, dW_t
$$

with adapted drift $\mu_t$ and volatility $\sigma_t > 0$, and a Brownian motion $W_t$ [@shreve2004stochastic]. These assumptions are standard in continuous-time asset pricing and serve as a stylized approximation to the discrete trade-driven dynamics of Section 9.1.

Under **no-arbitrage** [@delbaen1994general] and mild technical conditions (e.g. boundedness, Novikov), there exists an **equivalent martingale measure** $\mathbb{Q}$ [@shreve2004stochastic; @delbaen1994general] such that the **discounted spot**:

$$
\tilde P_t := \frac{P_t}{B_t}
$$

is a $\mathbb{Q}$-martingale:

$$
d\tilde P_t = \tilde \sigma_t \tilde P_t \, dW_t^{\mathbb{Q}}
$$

for some volatility $\tilde \sigma_t$ and $\mathbb{Q}$-Brownian motion $W^{\mathbb{Q}}_t$. This is the standard risk-neutralization argument [@shreve2004stochastic].

#### 9.8.2 Perpetual swap under spot anchoring

Consider a **perpetual swap contract** with the following properties:

- At any time $t$, a unit long position has instantaneous PnL (in token $A$):

$$
d\Pi_t^{\mathrm{perp}} = dP_t
$$

i.e. identical to holding one unit of the underlying.

- There is **no separate funding rate process** $f_t$.
- The contract has no fixed maturity; positions can be opened and closed at any time at the prevailing mark price.

Under the **spot-anchored design**, execution price and mark price are both equal to spot:

$$
P^{\mathrm{mark}}_t = P_t
$$

Thus, holding a unit of perp has the same cashflow behavior as holding a unit of spot, in every infinitesimal interval.

We denote by $F_t$ the **fair value process** (in token $A$) of one unit of perp at time $t$.

#### 9.8.3 Risk-neutral pricing of the perp

In a frictionless, fully collateralized setting, a long-one-unit perp position has the same instantaneous payoff as a long-one-unit spot position, with no additional cashflows. Under the risk-neutral measure $\mathbb{Q}$ [@shreve2004stochastic], any self-financing portfolio's discounted value must be a martingale [@delbaen1994general]. Since the perp is fully replicable by holding one unit of spot (and vice versa), we expect $F_t = P_t$.

We now formalize this.

**Theorem 9.9 (Funding-rate elimination under spot anchoring).**

Assume:

1. There exists an equivalent martingale measure $\mathbb{Q}$ under which the discounted spot $\tilde P_t = P_t / B_t$ is a martingale,
2. The perp contract has infinitesimal PnL $d\Pi_t^{\mathrm{perp}} = dP_t$ per unit, with no separate funding term.
3. The perp is tradable at price $F_t$, and any self-financing strategy must have discounted wealth that is a $\mathbb{Q}$-martingale.

Then the unique no-arbitrage price of the perpetual swap is:

$$
F_t = P_t, \quad \forall t
$$

and any non-zero funding process is unnecessary in equilibrium.

**_Proof Sketch._**
Let $\tilde F_t := F_t / B_t$ be the discounted perp price. Under risk-neutral pricing, $\tilde F_t$ must be a $\mathbb{Q}$-martingale for any tradeable asset without exogenous cashflows. Meanwhile, perp's PnL per unit is exactly $dP_t$, matching the underlying.

Consider two portfolios:

- **Portfolio A**: long one unit of perp,
- **Portfolio B**: long one unit of spot.

Both have the same infinitesimal payoff in units of $A$, namely $dP_t$. Thus, their discounted wealth processes $\tilde F_t$ and $\tilde P_t$ must satisfy the same SDE under $\mathbb{Q}$. In particular, the process:

$$
X_t := \tilde F_t - \tilde P_t
$$

satisfies:

$$
dX_t = d\tilde F_t - d\tilde P_t = 0
$$

since any non-zero drift or diffusion in $X_t$ would contradict the self-financing martingale property (we could form a zero-cost strategy long A, short B, and earn free money). Therefore, $X_t$ is $\mathbb{Q}$-almost surely constant. By no arbitrage at inception, we must have $F_0 = P_0$, hence $X_0 = 0$. Thus, $X_t \equiv 0$ for all $t$, so:

$$
\tilde F_t = \tilde P_t \quad \Rightarrow \quad F_t = P_t, \quad \forall t
$$

Suppose we introduce a non-zero funding process $f_t$ such that the perp PnL becomes:

$$
d\Pi_t^{\mathrm{perp}} = dP_t + f_t,dt
$$

If $f_t \neq 0$ on a set of positive measure, then the discounted perp wealth deviates from that of spot, allowing a self-financing strategy long one and short the other to generate a drift in discounted terms, violating the martingale condition. Thus, in an arbitrage-free equilibrium consistent with assumptions (1)-(3), we must have $f_t \equiv 0$ and $F_t = P_t$. $\square$

**Interpretation.** In the spot-anchored design, the perp is **literally the underlying** from a cashflow perspective. Funding rates in traditional perps compensate for a deviation between perp price and spot; here, by construction and by no-arbitrage, that deviation is zero, so the fair funding rate is identically zero.

> **Intuition:** Traditional perpetual swaps [@bitmex2016perpetual; @dydx2023v4; @gmx2022technical; @perp2021v2] need funding rates because their prices can drift away from spot—longs pay shorts (or vice versa) to nudge the perp price back toward the underlying. Theorem 9.9 shows that in our spot-anchored design, this drift is mechanically impossible: holding a perp gives you exactly the same dollar-for-dollar exposure as holding the spot asset. Since there's no gap to close, there's no need for a funding payment. This eliminates a major source of complexity and unpredictable costs for traders.

### 9.9 Risk-Neutral Measure and Market Completeness (Stylized)

We now sketch how the unified spot + perp system fits into the standard framework of risk-neutral pricing and market completeness.

#### 9.9.1 Basic traded assets

We consider the following traded assets:

- Bank account $B_t$ (numeraire),
- Spot token $B$ with price $P_t$,
- Perp contracts, each equivalent to a linear exposure to $P_t$ (Theorem 9.9).

Under the assumptions of Section 9.9, the pair $(B_t, P_t)$ forms a standard two-asset market with one source of uncertainty (Brownian motion), hence is **complete** in the usual Itô sense [@shreve2004stochastic]: any contingent claim with maturity $T$ and payoff $\Phi(P_T)$ can be replicated by dynamic trading in spot and bank account.

Perpetual swaps are then **redundant** assets in the classical replication sense: they do not introduce new span of payoffs beyond what spot provides.

#### 9.9.2 Existence of a risk-neutral measure

We record the standard result we rely on, in this context.

**Assumption 9.10 (NFLVR and EMM existence).**

We assume the unified spot market generated by the CLMM + CLOB engine has **no free lunch with vanishing risk** (NFLVR) in the sense of Delbaen-Schachermayer [@delbaen1994general] when abstracted into the continuous-time model of Section 9.9. Then there exists at least one equivalent martingale measure $\mathbb{Q}$ under which all discounted traded asset prices are $\mathbb{Q}$-martingales.

Under Assumption 9.10 and the dynamics of Section 9.9, the pair $(B_t, P_t)$ defines a **complete** market, and the derivative layer does not change completeness: it merely exposes a different interface (margining, leverage, cash-settled PnL) to the same underlying $P_t$.

### 9.10 Stability Under Adversarial Order Flow

We next address robustness: can an adversary manipulate perp valuations **without** paying real cost in the spot engine? The answer, by design, is no.

#### 9.10.1 Adversary model

Let an adversary $\mathcal{A}$ be any (possibly adaptive) strategy that:

- submits arbitrary spot orders (market or limit) to the CLMM + CLOB engine,
- submits arbitrary perp instructions (opens, closes, liquidations via oracle manipulation attempts, etc.),
- but cannot modify protocol rules or bypass the requirement that all hedge trades and liquidations execute via the spot engine.

We assume the adversary is capital-constrained: its token holdings must remain non-negative for all time; otherwise, we could ignore budget constraints and trivialize market manipulation.

#### 9.10.2 No perp-only manipulation theorem

We now formalize the statement:

> You cannot move perp PnL or perp marks independently from spot, unless you actually move spot by consuming liquidity.

Recall from Theorem 9.9 that fair perp value equals spot: $F_t = P_t$ and from Proposition 9.2 that any deviation would yield an arbitrage.

**Theorem 9.11 (No perp-only manipulation).**

Suppose:

1. The perp mark price is spot-anchored: $P^{\mathrm{mark}}_t = P_t$,
2. Hedge trades and liquidations are executed as real spot orders through the CLMM + CLOB engine,
3. The adversary $\mathcal{A}$ is capital-constrained and cannot bypass hedge execution.

Then any change in the adversary's perp PnL must coincide with a realized change in spot price $P_t$, generated by actual composition of liquidity. In particular, $\mathcal{A}$ cannot alter perp marks or PnL without incurring real trading costs in the spot engine.

**_Proof Sketch._**

At any time $t$, the perp PnL for the adversary's position is:

$$
\Pi_t^{\mathrm{perp}} = \int_0^t Q_{\mathcal{A}}(u) \, dP_u
$$

where $Q_{\mathcal{A}}(u)$ is the adversary's net perp exposure. This expression holds because:

- PnL is defined using mark price $P^{\mathrm{mark}}_u = P_u$,
- and each incremental PnL unit is exactly $Q_{\mathcal{A}}(u) , dP_u$.

The process $P_u$ itself is defined by the CLMM + CLOB execution engine (Section 9.1). Any change in $P_u$ is induced by net order flow through the spot engine, including possibly the adversary's own hedge trades and manipulative spot orders.

In particular:

- If the adversary only trades perps and does **not** cause any net spot execution (because hedges are required and executed on users' behalf by the protocol in the spot engine), then the spot price path $P_u$ is unaffected by these perp-only actions; hence $dP_u = 0$ for those actions, and they yield no PnL.
- If the adversary submits spot orders (either directly or through induced hedges) which move $P_u$, then all such moves correspond to real trades against the CLMM + CLOB liquidity. Any incremental perp PnL gained by the adversary is accompanied by a corresponding cost in buying high / selling low in the spot engine.

Thus, there is no degree of freedom to "push" perp marks independently: by definition and by execution design, the perp state is a deterministic functional of the spot price path $P(\cdot)$, and $P(\cdot)$ itself is only updated through real trades with the pool or other traders. A capital-constrained adversary cannot sustain profitable manipulation loops without eventually paying those trading costs. $\square$

**Corollary 9.12 (Oracle manipulation resistance).**

If all derivative valuation uses the on-chain spot axis $l(t)$ and its induced price $P_t = P(l(t))$, then there is no separate "price oracle" to manipulate. Any price impact must come from consuming real on-chain liquidity.

### 9.11 Extensions to Futures and Options on the Log Axis

We briefly outline how **dated futures** and **options** fit into the same spot-anchored framework, and why standard replication pricing applies.

#### 9.11.1 Dated futures

Fix a maturity $T > 0$. A **dated futures contract** on token $B$ with maturity $T$ and unit notional has payoff:

$$
\Phi^{\mathrm{fut}}(P_T) = P_T
$$

Under the same assumptions as Section 9.9 (complete market, risk-neutral measure $\mathbb{Q}$), the no-arbitrage futures price $F_t^{(T)}$ at time $t \leq T$ is given by the standard risk-neutral pricing formula [@shreve2004stochastic]:

$$
F_t^{(T)} = B_t \, \mathbb{E}^{\mathbb{Q}}\!\left[ \frac{P_T}{B_T} \,\bigg|\, \mathcal{F}_t \right]
$$

If the underlying has no carry and the short rate (r_t) is deterministic, the classical result gives:

$$
F_t^{(T)} = P_t \exp\left( \int_t^T r_u \, du \right)
$$

and in a zero-rate or token-$A$-numeraire, this reduces to $F_t^{(T)} = P_t$.

Implementation-wise, the unified CLMM + CLOB engine provides direct access to $P_t = P(l(t))$, so the futures can be:

- margin-settled using the same mark price,
- delta-hedged using the spot engine,
- priced in a risk-neutral sense using the stored log-process.

#### 9.11.2 Options

Let a **European option** with maturity $T$ have payoff:

$$
\Phi^{\mathrm{opt}}(P_T) = \phi(P_T)
$$

for some measurable function $\phi:\mathbb{R}_+ \to \mathbb{R}_+$ (e.g. $\max\{P_T - K, 0\}$ for a call, $\max\{K - P_T, 0\}$ for a put). The risk-neutral price at time $t$ is given by [@black1973pricing; @shreve2004stochastic]:

$$
C_t = B_t \, \mathbb{E}^{\mathbb{Q}}\!\left[ \frac{\phi(P_T)}{B_T} \,\bigg|\, \mathcal{F}_t \right]
$$

If we further assume a log-normal diffusion for $P_t$ under $\mathbb{Q}$, e.g.

$$
dP_t = r P_t dt + \sigma P_t dW_t^{\mathbb{Q}}
$$

then we recover the classical Black-Scholes PDE [@black1973pricing; @merton1973theory] in the **log-price variable** $l_t = \frac{1}{2} \log_2 P_t$. To see this, apply Itô's lemma to $l_t = \frac{1}{2} \log_2 P_t = \frac{\ln P_t}{2 \ln 2}$:

$$
dl_t = \frac{1}{2 \ln 2} \left( \frac{dP_t}{P_t} - \frac{1}{2} \frac{(dP_t)^2}{P_t^2} \right) = \frac{1}{2 \ln 2} \left( (r - \tfrac{1}{2}\sigma^2) dt + \sigma dW_t^{\mathbb{Q}} \right)
$$

Defining $\tilde\mu(l,t) := \frac{r - \frac{1}{2}\sigma^2}{2 \ln 2}$ and $\tilde\sigma(l,t) := \frac{\sigma}{2 \ln 2}$, the option price $u(t, l) = C_t$ satisfies the PDE with generator:

$$
\partial_t u + \frac{1}{2}\tilde\sigma^2(l,t) \partial_{ll} u + \tilde\mu(l,t) \partial_l u - r u = 0
$$

with terminal condition $u(T,l) = \phi(P(l))$. The key point is that:

- the unified log-axis $l$,
- the spot engine,
- and the risk-neutral measure $\mathbb{Q}$,

allow us to define standard derivative pricing in the usual way.

#### 9.11.3 Mesh-based approximation

On-chain, any option or futures can be:

1. Represented as a function of the log-price $l$,
2. Approximated on the Layer-2 mesh via nodal values $\phi(P(l_k))$,
3. Priced by numerical schemes (finite differences, Monte Carlo) on $l$-space, leveraging the same log-domain representation used for CLMM dynamics.

Convergence of these approximations to the continuous model follows from standard numerical analysis arguments, combined with the convergence guarantees in Section 5 and the log-price regularity in Section 9.1.

### 9.12 Margin Dynamics and Collateral Invariants

We now formalize the **margin process** for perp traders and show that equity is well-behaved (pathwise solvent under the protocol's liquidation rule) and convex in the underlying price.

#### 9.12.1 Accounts and positions

Fix a trader index $i$. For a single perpetual market on token $B$ (quoted in $A$), define:

- $q_i(t) \in \mathbb{R}$: net perp position at time $t$

  - $q_i(t) > 0$: net long,
  - $q_i(t) < 0$: net short.

- $M_i(t) \geq 0$: margin balance in token $A$ (collateral) at time $t$,
- $P(t) = P(l(t))$: spot/mark price as in Section 7-8 (unified log-price axis).
- $E_i(t)$: equity (or net account value) of trader $i$ in token $A$, defined by:

$$
E_i(t) = M_i(t) + q_i(t) \cdot P(t)
$$

We ignore cross-collateralization across markets here; it can be added by summing over markets with appropriate haircuts, but the single-market case captures the essential structure.

#### 9.12.2 Margin requirements

Let:

- $\Gamma_{\mathrm{init}}(\cdot)$: initial margin requirement function,
- $\Gamma_{\mathrm{maint}}(\cdot)$: maintenance margin requirement function,

We model these as:

$$
\Gamma^{\mathrm{init}}(P_t, q_i(t)) = \alpha_{\mathrm{init}} \, |q_i(t)| P_t
$$

$$
\Gamma^{\mathrm{maint}}(P_t, q_i(t)) = \alpha_{\mathrm{maint}} \, |q_i(t)| P_t
$$

with constants $0 < \alpha_{\mathrm{maint}} < \alpha_{\mathrm{init}} < 1$. More general state-dependent $\alpha(\cdot)$ (e.g. volatility-based) are possible but not necessary for the formal properties below.

We say that account $i$ is:

- **initially admissible** at opening time $t_0$ if:

  $$
  E_i(t_0) \ge \Gamma^{\mathrm{init}}(P_{t_0}, q_i(t_0))
  $$

- **maintained** at time $t$ if:

  $$
  E_i(t) \ge \Gamma^{\mathrm{maint}}(P_t, q_i(t))
  $$

#### 9.12.3 Equity dynamics

Under the spot-anchored design, per-unit perp PnL is $dP_t$ (Theorem 9.9). If the trader's net position is $q_i(t)$, the incremental change in equity satisfies:

$$
dE_i(t) = dM_i(t) + q_i(t) \, dP_t + P_t \, dq_i(t)
$$

- Changes in **margin** $dM_i(t)$ come from deposits/withdrawals and realized PnL (settlements).
- Changes in **position** $dq_i(t)$ come from opening/closing perps (executed at price $P_t$ with fees).
- The term $q_i(t) \, dP_t$ is unrealized PnL.

Assuming no external deposits/withdrawals in $[t, t+dt]$ and no immediate realization of PnL, we have:

$$
dM_i(t) = 0, \quad dq_i(t) = 0 \quad \Rightarrow \quad dE_i(t) = q_i(t) \, dP_t
$$

Thus for fixed $q_i$ over a time window:

$$
E_i(t) = E_i(t_0) + q_i \cdot (P_t - P_{t_0})
$$

**Lemma 9.13 (Equity linearity).**

Fix $q_i$ constant on $[t_0, t_1]$. Then $E_i(t)$ is an affine (linear + constant) function of $P_t$ on that interval:

$$
E_i(t) = M_i(t_0) + q_i P_t, \quad t\in [t_0,t_1]
$$

In particular, for a long $q_i > 0$, equity is strictly increasing in $P_t$; for a short $q_i < 0$, equity is strictly decreasing in $P_t$.

**_Proof Sketch._**

Immediate from the definition $E_i = M_i + q_i P_t$ and constant $q_i, M_i$ over the interval. $\square$

#### 9.12.4 Liquidation rule and invariants

We formalize a simple liquidation rule:

- If at some time $\tau$:

  $$
  E_i(\tau) < \Gamma^{\mathrm{maint}}(P_\tau, q_i(\tau))
  $$

  then the account is flagged for liquidation.

- The protocol issues a **liquidation order** of size $q_i(\tau)$ (full close) or some partial amount $q^{\mathrm{liq}}_i(\tau)$ that restores margin above maintenance.
- The liquidation order is hedged through the spot engine (CLMM + CLOB) as described in Section 9, with final executed average price $\overline{P}^{\mathrm{liq}}_i(\tau)$ and fees.

Let $\varphi_i$ denote the (deterministic, protocol-specified) function that maps the pre-liquidation state $(P_\tau, q_i(\tau), M_i(\tau))$ and execution price $\overline{P}^{\mathrm{liq}}_i(\tau)$ into post-liquidation state $(q_i(\tau^+), M_i(\tau^+))$. For a full liquidation we have $q_i(\tau^+) = 0$ and:

$$
M_i(\tau^+) = E_i(\tau) + \Delta^{\mathrm{slip+fee}}_i(\tau)
$$

where $\Delta^{\mathrm{slip+fee}}_i(\tau)$ is the net cost of slippage and fees for hedging the position via spot.

We model:

$$
\Delta^{\mathrm{slip+fee}}_i(\tau) \ge -C_{\max} |q_i(\tau)| P_\tau
$$

for some protocol-chosen constant $C_{\max} \ge 0$ capturing worst-case adverse execution (bounded slippage + fees due to mesh resolution, fixed-point error, and limited depth). The existence of such a bound follows from the convergence and error control results in Section 5 and finite-capacity structure of the CLMM segments (Section 5.5).

**Proposition 9.14 (Pathwise non-negative post-liquidation equity, absent tail loss).**

Assume:

1. At trade inception, $E_i(t_0) \ge \Gamma^{\mathrm{init}}(P_{t_0}, q_i(t_0))$.
2. The account is only liquidated when $E_i(\tau) < \Gamma^{\mathrm{maint}}(P_\tau, q_i(\tau))$.
3. The slippage+fee term satisfies $\Delta^{\mathrm{slip+fee}}_i(\tau) \ge -C_{\max} |q_i(\tau)| P_\tau$.
4. Maintenance margin satisfies $\alpha_{\mathrm{maint}} > C_{\max}$.

Then, after full liquidation at time $\tau$:

$$
E_i(\tau^+) = M_i(\tau^+) \ge 0
$$

**_Proof Sketch._**

At liquidation time $\tau$, pre-liquidation equity is:

$$
E_i(\tau) = M_i(\tau) + q_i(\tau)P_\tau
$$

After closing position fully:

$$
E_i(\tau^+) = M_i(\tau^+) = E_i(\tau) + \Delta^{\mathrm{slip+fee}}_i(\tau)
$$

By assumption (3):

$$
E_i(\tau^+) \ge E_i(\tau) - C_{\max}|q_i(\tau)|P_\tau
$$

At the moment of liquidation, we have $E_i(\tau) < \Gamma^{\mathrm{maint}}(P_\tau,q_i(\tau)) = \alpha_{\mathrm{maint}} |q_i(\tau)|P_\tau$, but by definition of the trigger the account has not defaulted yet, so we still assume $E_i(\tau) \ge 0$. We can bound:

$$
E_i(\tau^+) \ge - C_{\max}|q_i(\tau)|P_\tau
$$

To guarantee $E_i(\tau^+) \ge 0$, we impose a **solvency margin condition**:

$$
E_i(\tau) \ge C_{\max}|q_i(\tau)|P_\tau
$$

at the liquidation trigger. This is equivalent to:

$$
\Gamma^{\mathrm{maint}}(P_\tau,q_i(\tau)) = \alpha_{\mathrm{maint}}|q_i(\tau)|P_\tau \ge C_{\max}|q_i(\tau)|P_\tau
\quad \Rightarrow \quad
\alpha_{\mathrm{maint}} \ge C_{\max}
$$

Thus if $\alpha_{\mathrm{maint}} > C_{\max}$, then even in the worst-case slippage+fee realization, the account's equity after liquidation is non-negative. $\square$

**Interpretation.** As long as maintenance margin is chosen strictly above worst-case per-unit liquidation cost, the protocol enforces **pathwise solvency**: in the absence of systemic tail events beyond modeled $C_{\max}$, individual accounts cannot go negative; i.e., no "socialized loss" arises from normal liquidations.

> **Intuition:** Think of the maintenance margin as a "safety buffer" that the protocol requires you to keep. Proposition 9.14 says: if this buffer is bigger than the worst-case cost of closing your position (slippage + fees), then even if you get liquidated, there's always enough collateral left to cover the close. No debt gets passed to other users. The key insight is that by setting margin requirements high enough relative to execution costs, the protocol guarantees that liquidations are always "clean"—they never leave a hole in the system.

The remaining residual risk (failures of the $C_{\max}$ bound, oracle halts, extreme gaps) is handled by the insurance fund in the next subsection.

### 9.13 Insurance Fund, Bankruptcy Model, and Tail Bounds

We now formalize the insurance fund and show how it absorbs **tail liquidation shortfalls** that exceed the worst-case execution bound modeled in Section 9.12.

#### 9.13.1 Liquidation deficit and bankruptcy

Define the **liquidation deficit** for trader $i$ at time $\tau$ as:

$$
\Delta_i^{\mathrm{def}}(\tau) := \max{0, -E_i(\tau^+)}
$$

Under the idealized assumptions of Proposition 9.14 and $\alpha_{\mathrm{maint}} > C_{\max}$, we have $\Delta_i^{\mathrm{def}}(\tau) = 0$. In reality, extreme conditions may violate the bound implicitly encoded in $C_{\max}$, leading to $E_i(\tau^+) < 0$. In such cases:

- The protocol **declares the account bankrupt**,
- The negative equity is covered by the **insurance fund**.

Let $I(t) \ge 0$ denote the insurance fund balance (in token $A$) at time $t$. Let $\tau$ be a liquidation time. We update:

$$
I(\tau^+) = I(\tau) - \sum_{i \in \mathcal{B}_\tau} \Delta_i^{\mathrm{def}}(\tau)
$$

where $\mathcal{B}_\tau$ is the set of accounts that incur deficits at $\tau$. We require:

$$
I(\tau^+) \ge 0
$$

for all $\tau$ in normal operation; if not, the protocol enters an **emergency mode** (e.g. halt perps, haircuts profits, or uses backstop liquidity).

#### 9.13.2 Aggregate bound from execution error

From Section 6 and 9, total liquidation error can be decomposed into:

- mesh discretization error,
- fixed-point error,
- exp/log approximation error,
- and finite liquidity-induced slippage.

We can package these contributions into a per-unit error bound $\varepsilon_{\mathrm{liq}}$, such that for a trader with position $q_i$:

$$
E_i(\tau^+) \ge - \varepsilon_{\mathrm{liq}} |q_i(\tau)| P_\tau
$$

Thus:

$$
\Delta_i^{\mathrm{def}}(\tau) \le \varepsilon_{\mathrm{liq}} |q_i(\tau)| P_\tau
$$

Summing across all liquidated accounts in an event $\tau$:

$$
\Delta^{\mathrm{def}}_{\mathrm{tot}}(\tau) := \sum_{i \in \mathcal{B}_\tau} \Delta_i^{\mathrm{def}}(\tau)
\le \varepsilon_{\mathrm{liq}} \sum_{i \in \mathcal{B}_\tau} |q_i(\tau)| P_\tau
$$

Define the **liquidated open interest** at $\tau$:

$$
\mathrm{LOI}(\tau) := \sum_{i \in \mathcal{B}_\tau} |q_i(\tau)|
$$

then:

$$
\Delta^{\mathrm{def}}_{\mathrm{tot}}(\tau) \le \varepsilon_{\mathrm{liq}} \, \mathrm{LOI}(\tau) \, P_\tau
$$

#### 9.13.3 Insurance fund solvency condition

We can now specify a sufficient condition for insurance-fund solvency.

**Theorem 9.15 (Insurance fund solvency under bounded liquidation error).**

Suppose that:

1. At time $0$, the insurance fund has level $I(0)$.
2. For all liquidation events $\tau \in [0,T]$, the aggregate deficit satisfies:

   $$
   \Delta^{\mathrm{def}}_{\mathrm{tot}}(\tau) \le \varepsilon_{\mathrm{liq}} \, \mathrm{LOI}(\tau) \, P_\tau
   $$

   where $\varepsilon_{\mathrm{liq}}$ is a global bound determined by mesh resolution and numerical precision.

3. The total liquidated open interest over the horizon is bounded by:

   $$
   \sum_{\tau \le T} \mathrm{LOI}(\tau) \le \mathrm{LOI}_{\max}
   $$

   for some protocol parameter $\mathrm{LOI}_{\max}$.

If the initial insurance fund satisfies:

$$
I(0) \ge \varepsilon_{\mathrm{liq}} \, \mathrm{LOI}_{\max} \, P_{\max}
$$

where $P_{\max}$ is an upper bound on spot over $[0,T]$, then:

$$
I(t) \ge 0 \quad \forall t \in [0,T]
$$

**_Proof Sketch._**

Each liquidation event $\tau$ reduces the fund by at most $\Delta^{\mathrm{def}}_{\mathrm{tot}}(\tau)$, which is bounded by $\varepsilon_{\mathrm{liq}} \, \mathrm{LOI}(\tau) \, P_\tau \leq \varepsilon_{\mathrm{liq}} \, \mathrm{LOI}(\tau) \, P_{\max}$. Thus the cumulative reduction over $[0,T]$ is at most:

$$
\sum_{\tau \le T} \Delta^{\mathrm{def}}_{\mathrm{tot}}(\tau)
\le \varepsilon_{\mathrm{liq}} P_{\max} \sum_{\tau \le T} \mathrm{LOI}(\tau)
\le \varepsilon_{\mathrm{liq}} P_{\max} \, \mathrm{LOI}_{\max}
$$

If $I(0)$ is at least this amount, then for all $t \in [0,T]$, we have:

$$
I(t) \ge I(0) - \sum_{\tau \le t} \Delta^{\mathrm{def}}_{\mathrm{tot}}(\tau) \ge 0
$$

$\square$

**Interpretation.**

The bound $\varepsilon_{\mathrm{liq}}$ is directly linked to:

- mesh diameter $\delta$,
- fixed-point scaling $\sigma$,
- numerical errors $\varepsilon_{\exp_2}, \varepsilon_{\log_2}, \varepsilon_{\mathrm{arith}}$,
- and liquidity depth along the log-axis.

By tuning these parameters and bounding total liquidated OI, the protocol can _provably_ size the insurance fund so that tail liquidation deficits do not bankrupt the system over a target horizon.

> **Intuition:** Theorem 9.15 gives the protocol a concrete "insurance budget" formula. It says: if you know (1) how big the worst-case execution error is ($\varepsilon_{\mathrm{liq}}$), (2) the maximum price you might see ($P_{\max}$), and (3) the total amount of positions that could get liquidated in a bad scenario ($\mathrm{LOI}_{\max}$), then multiply them together—that's how much money you need in the insurance fund to stay solvent. This transforms risk management from guesswork into engineering: pick your target safety horizon, estimate the parameters, fund the insurance accordingly.

```text
                     INSURANCE FUND DYNAMICS
    ════════════════════════════════════════════════════════

                         ┌─────────────┐
         INFLOWS         │             │         OUTFLOWS
                         │  Insurance  │
    ┌──────────────┐     │    Fund     │     ┌──────────────┐
    │ Trading Fees │────►│    I(t)     │────►│  Liquidation │
    │   (portion)  │     │             │     │   Deficits   │
    └──────────────┘     │             │     │  Δ^def(τ)    │
                         │             │     └──────────────┘
    ┌──────────────┐     │             │
    │ Liquidation  │────►│             │
    │   Penalties  │     │             │
    └──────────────┘     └──────────────┘

    ════════════════════════════════════════════════════════

    FUND BALANCE EVOLUTION:
    ┌─────────────────────────────────────────────────────────┐
    │                                                         │
    │  I(t) = I(0) + Σ inflows - Σ Δ^def(τ)                   │
    │                τ≤t         τ≤t                          │
    │                                                         │
    └─────────────────────────────────────────────────────────┘

    SOLVENCY CONDITION (Theorem 9.15):
    ┌─────────────────────────────────────────────────────────┐
    │                                                         │
    │  I(0) ≥ ε_liq · P_max · LOI_max                         │
    │         ▲        ▲         ▲                            │
    │         │        │         └─ Max liquidated open       │
    │         │        │            interest over horizon     │
    │         │        └─ Maximum price in period             │
    │         └─ Worst-case execution error                   │
    │                                                         │
    │  ═══════════════════════════════════════════════════    │
    │  ⟹  I(t) ≥ 0  for all t ∈ [0, T]  (fund stays solvent) │
    │                                                         │
    └─────────────────────────────────────────────────────────┘

    PARAMETER DEPENDENCIES:
    ┌───────────────┐
    │    ε_liq      │◄─── Mesh diameter δ
    │  (execution   │◄─── Fixed-point scaling σ
    │    error)     │◄─── Numerical errors ε_exp2, ε_log2
    │               │◄─── Liquidity depth along log-axis
    └───────────────┘
```

_Figure 13: Insurance fund dynamics and solvency condition. The fund receives inflows from trading fees and liquidation penalties, and pays out deficits when liquidations execute at prices worse than equity. Theorem 9.15 provides a closed-form lower bound for initial capitalization to guarantee solvency over horizon $[0,T]$._

<!--
% LaTeX/TikZ version for PDF rendering:
% \begin{figure}[h]
% \centering
% \begin{tikzpicture}[node distance=2cm, >=stealth, thick]
%   % Fund box
%   \node[draw, rectangle, minimum width=3cm, minimum height=2cm, fill=blue!10] (fund) {Insurance Fund $I(t)$};
%
%   % Inflows
%   \node[draw, rectangle, left=of fund, fill=green!20] (fees) {Trading Fees};
%   \node[draw, rectangle, above left=1cm and 2cm of fund, fill=green!20] (penalties) {Liq. Penalties};
%
%   % Outflows
%   \node[draw, rectangle, right=of fund, fill=red!20] (deficits) {Deficits $\Delta^{\text{def}}$};
%
%   % Arrows
%   \draw[->] (fees) -- (fund);
%   \draw[->] (penalties) -- (fund);
%   \draw[->] (fund) -- (deficits);
%
%   % Solvency condition
%   \node[below=2cm of fund, align=center] (solvency) {
%     $I(0) \ge \epsilon_{\text{liq}} \cdot P_{\max} \cdot \text{LOI}_{\max}$\\
%     $\Rightarrow I(t) \ge 0 \ \forall t$
%   };
%   \draw[dashed, ->] (fund) -- (solvency);
% \end{tikzpicture}
% \caption{Insurance fund dynamics and solvency condition.}
% \end{figure}
-->

### 9.14 Compositionality and Determinism of Concurrent Liquidations

We now show that **multiple liquidations / hedge flows in the same interval** can be treated compositionally, and that the unified log-axis execution yields a deterministic final state independent (in a precise sense) of the internal ordering of these hedges.

#### 9.14.1 Aggregate hedge flow

Consider a collection of liquidation events within a short time window $[t, t+\Delta]$:

$$
\mathcal{L} = \{ (i, q_i^{\mathrm{liq}}) \}_{i \in \mathcal{I}}
$$

where each $q_i^{\mathrm{liq}}$ is the perp position to be closed (sign indicating long vs short). The protocol translates each liquidation into a **hedge order** in the spot engine:

$$
\Delta B^{\mathrm{hedge}}_i = -q_i^{\mathrm{liq}}, \quad \Delta A^{\mathrm{hedge}}_i = -q_i^{\mathrm{liq}} P_{\mathrm{exec},i}
$$

where $P_{\mathrm{exec},i}$ is the effective execution price for the hedge. On the log-axis, this corresponds to a signed quantity to be traded along the path in $l$-space.

Let:

$$
q^{\mathrm{agg}} := \sum_{i \in \mathcal{I}} q_i^{\mathrm{liq}}
$$

be the **net hedge size** (in units of $B$) of all liquidations in this window. For definiteness, suppose $q^{\mathrm{agg}} < 0$: net selling of $B$ into the CLMM + CLOB engine (downward price pressure). The case $q^{\mathrm{agg}} > 0$ is symmetric.

#### 9.14.2 Path-independence in the CLMM

Recall from Section 5.4 that for AMM execution along a log-path from $l_a$ to $l_b$, the pool deltas are:

$$
\Delta A(l_a \to l_b; \lambda) = -(\ln 2) \int_{l_a}^{l_b} \lambda(l) 2^{-l} dl
$$

$$
\Delta B(l_a \to l_b; \lambda) = (\ln 2) \int_{l_a}^{l_b} \lambda(l) 2^{l} dl
$$

If the total trade size in token $B$ is $\Delta B$, the final log-price $l_b$ is determined by solving:

$$
\Delta B = (\ln 2) \int_{l_a}^{l_b} \lambda(l) 2^{l} dl
$$

This integral is **additive** across multiple trades: if we apply trades with sizes $\Delta B_1, \Delta B_2$ sequentially in the same direction, the total move is determined by:

$$
\Delta B_1 + \Delta B_2 = (\ln 2) \int_{l_a}^{l_c} \lambda(l) 2^{l} dl
$$

for some $l_c$. This does not depend on the internal ordering of $\Delta B_1, \Delta B_2$; only on their sum. This is the standard path-independence of integrating a deterministic function $\lambda(l) 2^{l}$ along a one-dimensional axis.

**Lemma 9.16 (CLMM aggregate execution equivalence).**

Let $l_0$ be the starting log-price, and let ${\Delta B_j}_{j=1}^n$ be a finite set of trade sizes in the same direction (all buy or all sell). Let $l^{(1)}_n$ be the final log-price after executing them sequentially in any order, and let $l^{(2)}$ be the final log-price after executing a single aggregated trade of size $\Delta B^{\mathrm{agg}} = \sum_j \Delta B_j$. Then:

$$
l^{(1)}_n = l^{(2)}
$$

**_Proof Sketch._**

The CLMM dynamics on the log-axis are described by a first-order ODE in price vs. cumulative $\Delta B$, with right-hand side determined by $\lambda(l)$. For trades in the same direction, the solution mapping from cumulative size to final log-price is monotone and uniquely determined by integrating $\lambda(l) 2^l$. The order of partitioning a fixed total $\Delta B^{\mathrm{agg}}$ does not change the integral's endpoint; hence the final $l$ is independent of the split/order. $\square$

#### 9.14.3 CLOB node aggregation

At CLOB nodes, we must respect price-time priority. However, for a given set of hedge orders $\mathcal{L}$ executed in the same batch, the effective CLOB consumption is fully determined by:

- the total net size $q^{\mathrm{agg}}$,
- the existing resting orders and their fixed priorities.

Given that the CLOB book at node $l_k$ is specified as a sorted list (by price, then time), the **total amount filled** at each node for the aggregate size $q^{\mathrm{agg}}$ is independent of the internal ordering of the hedging "sub-orders", as long as all hedges are treated as arriving at the same block-time and are matched as a single meta-order.

This can be enforced at the protocol level by:

- aggregating all liquidation hedges in a block into one batch per direction per market,
- applying matching deterministically to that aggregated size.

#### 9.14.4 Determinism theorem

We combine AMM path-independence and CLOB aggregation.

**Theorem 9.17 (Deterministic final state for concurrent liquidations).**

Let $\mathcal{L}$ be a finite set of liquidation orders in a block, inducing hedge sizes $\{q_i^{\mathrm{liq}}\}_{i \in \mathcal{I}}$ in the same direction (net selling or net buying). Suppose the protocol:

1. Aggregates all these hedges into a single net size $q^{\mathrm{agg}} = \sum_{i} q_i^{\mathrm{liq}}$,
2. Executes this net hedge via the unified CLMM + CLOB engine, respecting CLOB price-time priority for the pre-existing resting book.

Then:

- The final log-price $l_{\mathrm{final}}$,
- The final CLMM reserves $(A_{\mathrm{pool}}, B_{\mathrm{pool}})$,
- The set of fully/partially filled resting CLOB orders,

are uniquely determined by:

- the initial state (CLMM liquidity $\lambda(l)$, resting CLOB book),
- and the net hedge size $q^{\mathrm{agg}}$,

and are **independent** of internal ordering or identity of the individual liquidations in $\mathcal{L}$.

**_Proof Sketch._**

1. **CLOB part.**

   Aggregating hedges into a single meta-order reduces the problem to matching a single order of size $q^{\mathrm{agg}}$ against a fixed, static CLOB book. Price-time priority and deterministic tie-breaking guarantee uniqueness of the resulting matched set and remaining book.

2. **CLMM part.**

   Any portion of the net hedge not filled on the CLOB side is executed via the CLMM along the log-axis. By Lemma 9.16, the final log-price and pool reserves depend only on the total AMM $\Delta B$, not on how it is partitioned among sub-orders.

3. **Composition.**

   Combined CLOB+CLMM flow is a composition of two deterministic maps applied to the aggregate size. Since the protocol explicitly aggregates hedges before execution and the initial state is fixed, the final state is uniquely determined.

Identity of which liquidator is assigned which slice of the aggregate execution and what average price each gets can then be calculated as a pure accounting step, without affecting the global state. $\square$

**Interpretation.**

From the perspective of the _global state_ (price, pool reserves, order book), concurrent liquidations in a block are equivalent to a single, deterministic net hedge. This makes the perp layer **serializable**, which is important for both implementation and formal verification.

> **Intuition:** Imagine 100 traders all getting liquidated in the same block. You might worry that the order in which these liquidations happen affects the final price—maybe the first liquidations get better prices, and later ones get worse. Theorem 9.17 says: it doesn't matter. Because the AMM's response to trades depends only on the total volume (not who sent it or in what order), you can mentally "combine" all 100 liquidations into one big trade. The final price and reserves are the same regardless of sequencing. This is crucial for blockchain systems where transaction ordering within a block can be unpredictable.

### 9.15 Dynamic Stability and Leverage Constraints

We address **system-level stability**: ensuring that leverage in the perp layer does not induce unbounded feedback loops into the spot engine.

#### 9.15.1 Effective leverage and depth

Let:

- $D^{\downarrow}(l)$: aggregate _downward_ spot depth at log-price $l$, i.e. maximum amount of token $B$ that can be sold (net) starting from $l$ before price drops by a given log-interval $\Delta l$. Analogously, $D^{\uparrow}(l)$ for upward moves.
- For simplicity, restrict attention to downward risk; upward is symmetric.

We define **local effective depth** at $l$:

$$
\mathcal{L}^{\downarrow}(l; \Delta l) := \int_l^{l - \Delta l} \lambda(u) (\ln 2)2^{u}du + \text{CLOB depth in }[l-\Delta l,l]
$$

measured in units of token $B$. This is the maximum net sell quantity that can be absorbed while moving price from $l$ to $l - \Delta l$.

Let $\{q_i\}$ be the open perp positions at log-price $l$, and define the **notional** of position $i$:

$$
N_i(l) := |q_i| P(l)
$$

Define effective **leverage** of account $i$:

$$
\ell_i(l) := \frac{N_i(l)}{E_i(l)} = \frac{|q_i| P(l)}{E_i(l)}
$$

Let $\mathfrak{L}(l)$ be an aggregate leverage measure, e.g.:

$$
\mathfrak{L}(l) := \sum_i \ell_i(l) \wedge \ell_{\max}
$$

with some hard cap $\ell_{\max}$.

#### 9.15.2 Stability condition

Intuitively, a downward price move triggers liquidations; liquidations execute hedges; hedges sell more into spot, causing further downward pressure. To avoid runaway spirals, we need to keep **total forced sell volume** from any shock bounded by the absorbing capacity of the depth profile.

We formalize a simplified, conservative condition.

Let $\Delta l_{\max}^{\mathrm{liq}}$ be a protocol-targeted **maximum allowed log-price drop** due to liquidation cascades from a bounded shock (e.g., a 10% move). Assume that:

- Any liquidation event at $l$ closes at most a fraction $\beta$ of total notional (due to partial liquidation rules or collateralization),
- The worst-case liquidation volume (in token $B$) generated by a one-sided move of size $\Delta l_{\max}^{\mathrm{liq}}$ is bounded by:

$$
V^{\mathrm{liq}}_{\downarrow}(l; \Delta l_{\max}^{\mathrm{liq}}) \le \beta \sum_i |q_i|
$$

The system remains dynamically stable if this worst-case liquidation volume does not exceed the total absorbing depth over the same interval:

$$
V^{\mathrm{liq}}_{\downarrow}(l; \Delta l_{\max}^{\mathrm{liq}}) \le \mathcal{L}^{\downarrow}(l; \Delta l_{\max}^{\mathrm{liq}})
$$

This yields a **leverage constraint**:

**Definition 9.18 (Stability-compatible leverage).**

We say that the system is **stability-compatible** at log-price $l$ if for some chosen $\Delta l_{\max}^{\mathrm{liq}} > 0$:

$$
\beta \sum_i |q_i| \le \mathcal{L}^{\downarrow}(l; \Delta l_{\max}^{\mathrm{liq}})
$$

Equivalently, in notional terms:

$$
\beta \sum_i N_i(l) \le P(l) \cdot \mathcal{L}^{\downarrow}(l; \Delta l_{\max}^{\mathrm{liq}})
$$

This inequality can be enforced via per-account leverage caps or dynamic margin multipliers as a function of observable depth.

#### 9.15.3 Stability theorem (no unbounded liquidation cascades)

We now state a stylized result: if the above inequality holds at each step, a liquidation cascade cannot produce an unbounded sequence of price jumps.

**Theorem 9.19 (Bounded liquidation cascades under leverage constraint).**

Suppose:

1. At log-price $l_0$, the market is stability-compatible in the sense of Definition 9.18 with some $\Delta l_{\max}^{\mathrm{liq}}$.
2. A downward external shock moves log-price from $l_0$ to $l_0 - \delta l$, with $0 < \delta l \le \Delta l_{\max}^{\mathrm{liq}}$.
3. All liquidations triggered by this move are executed via hedges in the spot engine, generating additional net sell volume $V^{\mathrm{liq}}_{\downarrow}$ in token $B$ which satisfies:

$$
V^{\mathrm{liq}}_{\downarrow} \le \beta \sum_i |q_i|
$$

Then the **total additional log-price movement** due purely to liquidation hedges is bounded from below $-\Delta l_{\max}^{\mathrm{liq}}$; i.e., the post-cascade log-price $l_{\mathrm{final}}$ satisfies:

$$
l_0 - 2 \Delta l_{\max}^{\mathrm{liq}} \le l_{\mathrm{final}} \le l_0
$$

In particular, the cascade cannot produce an unbounded collapse in a single shock step.

**_Proof Sketch._**

The initial external shock moves price from $l_0$ to $l_0 - \delta l$. At the new price, some positions breach maintenance margin and are liquidated. By assumption, total liquidation volume is at most $\beta \sum_i |q_i|$. Spot depth over the interval $[l_0, l_0 - \Delta l_{\max}^{\mathrm{liq}}]$ can absorb at least $\mathcal{L}^{\downarrow}(l_0; \Delta l_{\max}^{\mathrm{liq}})$ volume. Stability-compatibility guarantees:

$$
\beta \sum_i |q_i| \le \mathcal{L}^{\downarrow}(l_0; \Delta l_{\max}^{\mathrm{liq}})
$$

Thus, executing the liquidation hedges from $l_0 - \delta l$ cannot push the price below $l_0 - \Delta l_{\max}^{\mathrm{liq}}$: the total volume of forced sells is insufficient to exhaust the entire depth down to that level.

Therefore, the post-liquidation price $l_{\mathrm{final}}$ lies in the interval $[l_0 - \Delta l_{\max}^{\mathrm{liq}}, l_0 - \delta l]$. The total movement from the pre-shock price $l_0$ is then at most $-\Delta l_{\max}^{\mathrm{liq}} - \delta l \ge -2 \Delta l_{\max}^{\mathrm{liq}}$. A more refined argument using local depth and updated leverage can further tighten the bound, but this coarse bound suffices to show non-divergence. $\square$

**Interpretation.**

The protocol can treat $\Delta l_{\max}^{\mathrm{liq}}$ as a **design parameter** (e.g. corresponding to a 10-20% move) and calibrate maximum leverage / margin floors such that the inequality in Definition 9.18 holds. This gives a formal, tunable guarantee that:

- A single shock cannot trigger an unbounded liquidation spiral,
- The spot engine has enough depth to absorb the induced hedges.

This is _not_ a statement that price cannot move further over time (new shocks may arrive), but that the **feedback loop between leverage and spot depth is controlled** at each stage.

> **Intuition:** This theorem addresses the nightmare scenario: price drops → liquidations fire → liquidation hedges sell into the market → price drops more → more liquidations → and so on in a death spiral. The key insight is that if the total leveraged position size is small relative to the market's absorbing capacity (how much selling the AMM + order book can handle), then even if all positions get liquidated at once, there's "enough liquidity" to absorb them without pushing the price to zero. By capping leverage based on available depth, the protocol mathematically bounds how far any cascade can go.

```text
    LIQUIDATION CASCADE FEEDBACK LOOP (WITHOUT CONSTRAINT)
    ═══════════════════════════════════════════════════════

       ┌──────────────┐         ┌──────────────┐
       │   External   │         │   Price      │
       │    Shock     │────────►│   Drops      │
       │   (Δl < 0)   │         │   l → l - δl │
       └──────────────┘         └──────┬───────┘
                                       │
                                       ▼
       ┌──────────────┐         ┌──────────────┐
       │   Further    │         │  Positions   │
       │    Price     │◄────────│  Breach MM   │
       │    Drop      │         │ (Liquidate)  │
       └──────┬───────┘         └──────┬───────┘
              │                        │
              │    DANGER:             ▼
              │    FEEDBACK      ┌──────────────┐
              │    LOOP!         │  Hedge Sells │
              │                  │   Execute    │
              └──────────────────│ (V^liq vol)  │
                                 └──────────────┘


    STABILITY-COMPATIBLE SYSTEM (WITH CONSTRAINT)
    ═══════════════════════════════════════════════════════

        LEVERAGE CONSTRAINT (Definition 9.18):
        ┌─────────────────────────────────────────────────┐
        │                                                 │
        │   β · Σ|q_i|  ≤  𝓛↓(l; Δl_max^liq)             │
        │   ▲                    ▲                        │
        │   │                    │                        │
        │   Worst-case           Spot depth that can      │
        │   liquidation          absorb selling over      │
        │   volume               interval Δl_max^liq      │
        │                                                 │
        └─────────────────────────────────────────────────┘

        DEPTH ABSORBS LIQUIDATION FLOW:
        ┌─────────────────────────────────────────────────────┐
        │                                                     │
        │   Price l                                           │
        │      │                                              │
        │  l_0 ┼───────────●  ◄─── Pre-shock price            │
        │      │           ║                                  │
        │      │           ║  External shock (δl)             │
        │      │           ▼                                  │
        │      ┼───────────●  ◄─── Post-shock, liq triggered  │
        │      │       ████║████                              │
        │      │       ████║████  Depth 𝓛↓ absorbs V^liq     │
        │      │       ████▼████                              │
        │      ┼───────●───────   ◄─── l_final (bounded!)     │
        │      │       ▲                                      │
        │      │       │                                      │
        │      │  Δl_max^liq (max allowed drop from cascade)  │
        │      │                                              │
        └─────────────────────────────────────────────────────┘

        THEOREM 9.19 GUARANTEE:
        ┌─────────────────────────────────────────────────────┐
        │                                                     │
        │   l_0 - 2·Δl_max^liq  ≤  l_final  ≤  l_0           │
        │                                                     │
        │   ⟹  Cascade CANNOT produce unbounded collapse     │
        │   ⟹  Feedback loop is CONTROLLED at each stage     │
        │                                                     │
        └─────────────────────────────────────────────────────┘

        PROTOCOL CALIBRATION:
        ┌────────────────────────────────────────────────────┐
        │  Choose Δl_max^liq (e.g. 10-20% price move)        │
        │                 ▼                                  │
        │  Measure spot depth 𝓛↓ over that interval         │
        │                 ▼                                  │
        │  Set leverage caps: β·Σ|q_i| ≤ 𝓛↓                 │
        │                 ▼                                  │
        │  RESULT: Formal guarantee against death spirals    │
        └────────────────────────────────────────────────────┘
```

_Figure 14: Leverage constraint and liquidation cascade stability. Without constraints (top), liquidations can create a dangerous feedback loop where hedges cause further price drops triggering more liquidations. The stability condition (Definition 9.18) ensures that worst-case liquidation volume is bounded by spot market depth, preventing unbounded cascades. Theorem 9.19 guarantees that the total price movement remains bounded even under maximum stress._

<!--
% LaTeX/TikZ version for PDF rendering:
% \begin{figure}[h]
% \centering
% \begin{tikzpicture}[>=stealth, node distance=1.5cm]
%   % Feedback loop diagram
%   \node[draw, rectangle] (shock) {External Shock};
%   \node[draw, rectangle, right=of shock] (drop) {Price Drops};
%   \node[draw, rectangle, below=of drop] (liq) {Liquidations};
%   \node[draw, rectangle, left=of liq] (hedge) {Hedge Sells};
%   \draw[->] (shock) -- (drop);
%   \draw[->] (drop) -- (liq);
%   \draw[->] (liq) -- (hedge);
%   \draw[->, dashed, red] (hedge) -- (drop) node[midway, left] {Feedback};
%
%   % Stability condition box
%   \node[below=3cm of shock, draw, rectangle, fill=green!10, text width=6cm, align=center] {
%     \textbf{Stability Constraint:}\\
%     $\beta \sum_i |q_i| \le \mathcal{L}^{\downarrow}$\\
%     $\Rightarrow$ Cascade bounded by $2\Delta l_{\max}^{\text{liq}}$
%   };
% \end{tikzpicture}
% \caption{Leverage constraint prevents liquidation cascades.}
% \end{figure}
-->

## 10. Related Work

This section surveys prior work across four areas relevant to our design: constant function market maker (CFMM) theory, tick-based versus continuous price representations, on-chain perpetual swap protocols, and numerical methods for log-domain computation in trading systems.

### 10.1 Constant Function Market Makers

The theoretical foundations of automated market makers were established by early work on prediction markets and subsequently formalized in the context of decentralized exchanges. Uniswap v1 and v2 [@adams2020uniswap] popularized the constant product invariant $x \cdot y = k$, where reserves of two tokens are constrained to lie on a hyperbola. Subsequent theoretical analysis formalized CFMM behavior, including price oracle properties [@angeris2020improved], market dynamics and equilibrium [@angeris2021analysis], and the relationship between curvature and market making [@angeris2022tail]. This elegantly simple design provides liquidity across all prices but suffers from capital inefficiency: most liquidity sits at prices far from the current market rate.

Uniswap v3 [@adams2021uniswap] introduced **concentrated liquidity**, allowing liquidity providers to specify price ranges $[p_a, p_b]$ within which their capital is active. The core innovation is to represent positions via the square-root price $\sqrt{P}$ and to discretize the price axis into **ticks** spaced at fixed percentage intervals. Each tick $i$ corresponds to a price $P_i = 1.0001^i$, giving approximately 1 basis point granularity. The virtual reserves within a tick follow a local constant-product curve, and crossing a tick boundary triggers discrete liquidity updates.

Our log-axis model can be viewed as a continuous generalization of Uniswap v3's tick system [@adams2021uniswap]. Where v3 uses discrete ticks at $P_i = 1.0001^i$, we work directly with the log-sqrt-price $l = \log_2 \sqrt{P}$ as a continuous coordinate. The relationship is:

$$
l = \frac{1}{2} \log_2 P = \frac{i \cdot \log_2(1.0001)}{2} \approx 7.2 \times 10^{-5} \cdot i
$$

Our approach eliminates the discrete tick-crossing logic in favor of smooth integration over a liquidity density function $\lambda(l)$. This simplifies the mathematical treatment and enables tighter error bounds, though both approaches converge to the same economic behavior as tick spacing shrinks.

The economics of liquidity provision under AMMs have been studied extensively, including LP returns in geometric mean markets [@evans2020liquidity] and the phenomenon of loss-versus-rebalancing (LVR), which quantifies the cost to LPs from trading against informed arbitrageurs [@milionis2022automated; @milionis2023automated]. Our log-domain representation does not eliminate LVR—it is inherent to any AMM design—but provides a clean framework for analyzing its magnitude.

**Balancer** [@martinelli2019balancer] generalizes the two-token constant product to weighted pools with $n$ assets and arbitrary weights $w_i$, maintaining the invariant $\prod_i R_i^{w_i} = k$. This enables index-fund-like products but does not address concentrated liquidity.

**Curve Finance** [@egorov2019stableswap] pioneered the **StableSwap** invariant, which interpolates between constant-product and constant-sum behavior. For stablecoin pairs expected to trade near parity, this concentrates liquidity around the peg while still providing deep markets if prices diverge. The Curve v2 design [@egorov2021automatic] extends this to volatile asset pairs using an internal oracle to dynamically recenter the liquidity concentration. Our log-axis representation is orthogonal to such invariant design choices and could in principle accommodate StableSwap-like curves by defining an appropriate $\lambda(l)$ profile.

**Orca Whirlpools** [@orca2022whirlpools] and **Raydium** [@raydium2023clmm] implement concentrated liquidity on Solana, closely following Uniswap v3's tick-based model but adapted for Solana's account structure and compute constraints. The design uses 64-bit fixed-point arithmetic for $\sqrt{P}$ (the `sqrt_price_x64` representation) and maintains per-tick liquidity in a sparse data structure. Our work differs by (i) using a log-domain representation that converts multiplicative price moves to additive shifts, (ii) providing explicit fixed-point error analysis with convergence guarantees, and (iii) integrating spot and derivative execution into a unified framework.

### 10.2 Tick-Based Designs vs. Log-Axis Models

The dominant paradigm in concentrated liquidity AMMs is the **tick-based** model, where the price axis is discretized into a lattice of ticks and liquidity is specified per-tick or per-tick-range. Uniswap v3 [@adams2021uniswap], Orca Whirlpools [@orca2022whirlpools], Raydium [@raydium2023clmm], and similar designs all follow this pattern. The choice of tick spacing (e.g., 1 bp, 5 bp, 30 bp, 100 bp) trades off between granularity and gas/compute cost.

Our **log-axis** model takes a different approach: rather than discretizing prices into ticks, we treat the log-sqrt-price $l$ as a continuous coordinate and represent liquidity as a density function $\lambda(l)$. Swap execution is computed by integrating this density, and the on-chain representation uses high-precision fixed-point encoding of $l$ rather than a tick index.

The practical differences are:

1. **Granularity.** Tick-based systems have inherent price granularity determined by tick spacing. Our log-axis model can represent prices at arbitrary precision limited only by the fixed-point encoding (e.g., 64 or 96 fractional bits).

2. **Liquidity representation.** Tick-based systems store liquidity per tick, requiring $O(T)$ storage for $T$ active ticks. Our model can represent smooth liquidity profiles with $O(1)$ parameters (e.g., uniform liquidity in a range) or use band-based discretization when desired.

3. **Execution logic.** Tick-based systems must handle tick-crossing events, updating active liquidity at each boundary. Our model integrates continuously over $\lambda(l)$, with discrete bands as an optional optimization rather than a fundamental constraint.

4. **Error analysis.** Our fixed-point specification (Section 6) provides explicit error bounds showing that on-chain execution converges to the continuous model as precision increases. Tick-based systems have implicit discretization error from tick spacing but typically lack formal convergence guarantees.

The log-axis representation also has natural connections to **geometric Brownian motion** models in finance, where log-prices follow arithmetic random walks. This makes our framework amenable to standard stochastic calculus techniques [@shreve2004stochastic], as exploited in Section 9.

### 10.3 On-Chain Perpetual Protocols

Perpetual swaps originated in centralized cryptocurrency exchanges (notably BitMEX [@bitmex2016perpetual] in 2016) and have since been adapted for on-chain execution. We survey the major approaches and contrast them with our spot-anchored design.

#### 10.3.1 Oracle-based perpetuals

**GMX** [@gmx2022technical] (Arbitrum, Avalanche) and **GNS/Gains Network** use external price oracles (typically Chainlink) to determine mark prices for perpetual positions. Traders open leveraged long/short positions against a shared liquidity pool, with PnL settled against pool reserves. Key characteristics include:

- **No on-chain price discovery:** The mark price is imported from external oracles, not determined by on-chain trading activity.
- **Funding rate:** GMX [@gmx2022technical] uses hourly funding payments to balance long/short open interest, similar to centralized exchange perps.
- **Liquidity pool as counterparty:** The GLP (GMX Liquidity Provider) pool absorbs trader PnL, creating an adversarial dynamic between traders and LPs.

Our spot-anchored model differs fundamentally: the mark price equals the on-chain spot price by construction, eliminating basis risk and the need for funding rate mechanisms. The protocol hedges derivative positions in real-time against the spot market, so LPs are not directly exposed to directional trader PnL.

#### 10.3.2 vAMM-based perpetuals

**Perpetual Protocol** [@perp2021v2] (v1 and v2) pioneered the **virtual AMM** (vAMM) approach, where perpetual positions are priced against a virtual constant-product curve that exists only for price discovery—no actual token swaps occur in the vAMM. Key characteristics:

- **Synthetic price curve:** The vAMM maintains virtual reserves $x_v, y_v$ satisfying $x_v \cdot y_v = k_v$. Opening a long position "buys" from the vAMM, moving its virtual price.
- **Funding rate:** Positions pay/receive funding based on the difference between vAMM price and an external index (oracle) price, incentivizing convergence.
- **Insurance fund:** Covers shortfalls when liquidations are unprofitable.

The v2 design (on Optimism) introduced **concentrated liquidity vAMMs** using Uniswap v3-style mechanics, allowing market makers to provide virtual liquidity in ranges.

Our approach avoids the vAMM construction entirely. Because derivative positions are atomically hedged against the real spot market, the mark price is anchored to spot by mechanism design rather than by funding rate incentives. This eliminates the basis risk inherent in vAMM systems, where the vAMM price can diverge from fair value during periods of high demand.

#### 10.3.3 Order book perpetuals

**dYdX** [@dydx2023v4] operates an off-chain order book with on-chain settlement (originally on Ethereum L1, now on a dedicated Cosmos appchain). This provides a centralized-exchange-like experience with:

- **Limit order book:** Prices are determined by order matching, not AMM curves.
- **Funding rate:** Standard 8-hour funding payments based on mark-index spread.
- **Hybrid custody:** Orders are matched off-chain, with periodic settlement and proofs posted on-chain.

**Drift Protocol** [@drift2023v2] (Solana) implements a hybrid model combining a central limit order book (CLOB) with a backstop AMM. The CLOB provides price discovery and tight spreads when active market makers are present; the AMM ensures liquidity is always available.

Our unified execution model (Section 7) is closest in spirit to Drift's hybrid approach, combining CLOB and CLMM liquidity. The key difference is our log-axis representation, which provides a consistent coordinate system across both venues, and our spot-anchored derivative design, which eliminates funding rates.

#### 10.3.4 Basis and funding rate mechanisms

A central challenge for perpetual swaps is maintaining price convergence to the underlying spot market. Traditional funding rate mechanisms (used by BitMEX [@bitmex2016perpetual], Binance, dYdX [@dydx2023v4], GMX [@gmx2022technical], Perpetual Protocol [@perp2021v2], etc.) impose periodic payments between longs and shorts based on the mark-index spread:

$$
\text{FundingPayment} = \text{PositionSize} \times \text{FundingRate} \times \text{IndexPrice}
$$

where the funding rate is typically proportional to the time-weighted average premium/discount of the perp price to the index. This mechanism relies on **arbitrageurs** to enforce convergence: when the perp trades at a premium, longs pay shorts, incentivizing arbitrageurs to short the perp and long the spot.

Our spot-anchored model eliminates funding rates entirely (Theorem 9.9). Because each derivative trade is atomically hedged against the spot market, the perp mark price equals the post-trade spot price by construction. There is no basis to arbitrage and no funding payments to administer. The economic cost of this design is that derivative trades incur execution costs in the spot market, which are passed through to traders as part of the transaction.

### 10.4 Log-Domain Representations and Fixed-Point Arithmetic

The use of logarithmic representations for prices and quantities has a long history in quantitative finance and high-frequency trading systems.

#### 10.4.1 Log-price models in finance

The Black-Scholes-Merton framework [@black1973pricing; @merton1973theory] and its descendants model asset prices as geometric Brownian motion, which is equivalent to arithmetic Brownian motion in the log-price. This makes log-prices the natural coordinate for:

- **Volatility estimation:** Log-returns are approximately normally distributed under GBM, simplifying statistical analysis.
- **Option pricing:** The Black-Scholes formula is most naturally expressed in terms of $\log(S/K)$ and $\sigma\sqrt{T}$.
- **Risk management:** Value-at-Risk and related metrics are typically computed using log-return distributions.

Standard textbook treatments of these topics include Hull [@hull2021options] and Wilmott et al. [@wilmott1995mathematics]. Our choice of log-sqrt-price $l = \log_2 \sqrt{P}$ follows this tradition while adapting to the specific needs of on-chain AMM execution.

#### 10.4.2 Fixed-point arithmetic in trading systems

High-frequency trading systems have long used fixed-point arithmetic to avoid the latency and non-determinism of floating-point operations. Key considerations include:

- **Determinism:** Fixed-point operations produce identical results across all hardware, essential for consensus in blockchain systems.
- **Precision:** The number of fractional bits determines the granularity of representable values. Our analysis (Section 6) shows that 64-96 fractional bits provide sufficient precision for practical AMM applications.
- **Overflow:** Fixed-point multiplication can overflow; careful range analysis is required. Our encoding bounds $|L| \le 2^{63}$ (for 64-bit signed integers) correspond to price ratios up to approximately $2^{2^{63}/\sigma}$, which is astronomically large for reasonable $\sigma$.

**Uniswap v3** [@adams2021uniswap] uses `Q64.64` and `Q128.128` fixed-point formats for intermediate calculations, with the square-root price stored as `sqrtPriceX96` (a Q64.96 format). Our approach differs by storing the _log_ of the square-root price, which converts multiplicative price updates to additive operations and simplifies the analysis of compounding errors.

#### 10.4.3 Exponential and logarithm implementations

Computing $2^x$ and $\log_2(x)$ efficiently in fixed-point arithmetic is a well-studied problem [@muller2016elementary]. Common approaches include:

- **Table lookup with interpolation:** Precompute $2^{k/N}$ for $k = 0, \ldots, N-1$ and use linear or polynomial interpolation [@tang1990logarithm; @sablier2021prbmath]. This is fast but requires $O(N)$ storage.
- **Polynomial approximation:** Use minimax polynomials (e.g., Remez algorithm) to approximate $2^x$ on $[0, 1)$ and reduce arbitrary arguments via $2^{n+f} = 2^n \cdot 2^f$. This is the approach used in most high-performance implementations [@muller2016elementary].
- **CORDIC-style algorithms:** Iterative shift-and-add algorithms that converge to exponentials/logarithms. Less common in modern implementations due to latency.

Our specification (Section 6) abstracts over the choice of algorithm, requiring only that the implemented $\exp_2$ and $\log_2$ functions satisfy explicit error bounds. This allows implementers flexibility while providing the formal guarantees needed for our convergence theorems.

#### 10.4.4 Prior work on DeFi numerics

Several projects have addressed numerical precision in DeFi protocols:

- **Balancer v2** [@balancer2021math] provides extensive documentation on fixed-point math for weighted pool calculations, including error analysis for power functions.
- **Yield Protocol** uses fixed-point representations for yield-bearing tokens with careful attention to rounding.
- **PRBMath** [@sablier2021prbmath] and **ABDKMath64x64** [@abdk2019math64x64] are Solidity libraries providing fixed-point logarithms and exponentials, widely used in DeFi development.

Our contribution extends this line of work by (i) providing a complete formal specification linking on-chain fixed-point execution to a continuous mathematical model, (ii) proving explicit convergence bounds, and (iii) integrating the numerical framework with a unified spot+derivative execution engine.

### 10.5 Summary of Distinctions

To summarize the key distinctions between our approach and prior work:

| Aspect               | Prior Art                                                                                              | Our Approach                                |
| -------------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------- |
| Price representation | Square-root price (Uni v3 [@adams2021uniswap]), tick index                                             | Log-sqrt-price as continuous coordinate     |
| Liquidity model      | Per-tick discrete (Uni v3 [@adams2021uniswap], Orca [@orca2022whirlpools], Raydium [@raydium2023clmm]) | Density function $\lambda(l)$ over log-axis |
| Perp mark price      | Oracle (GMX [@gmx2022technical]), vAMM (Perp [@perp2021v2]), order book (dYdX [@dydx2023v4])           | Spot-anchored (equals post-trade spot)      |
| Basis convergence    | Funding rate mechanism                                                                                 | Eliminated by construction                  |
| Derivative hedging   | External arbitrageurs                                                                                  | Protocol-level atomic hedging               |
| Numerical framework  | Ad-hoc fixed-point                                                                                     | Formal spec with convergence proofs         |
| CLOB+AMM integration | Hybrid (Drift [@drift2023v2])                                                                          | Unified execution on log-axis               |

The combination of log-domain representation, spot-anchored derivatives, and formal numerical guarantees distinguishes our design from existing protocols and provides a foundation for rigorous analysis of complex DeFi mechanisms.

## 11. Conclusion and Future Work

### 11.1 Summary of Contributions

This paper has presented a unified framework for decentralized exchange design based on logarithmic price representation and spot-anchored derivatives. We summarize the main conceptual and technical contributions:

**Logarithmic price axis.** We introduced the log-sqrt-price coordinate $l = \log_2 \sqrt{P}$ as the fundamental representation for prices throughout the system. This choice transforms multiplicative price movements into additive shifts, simplifies the mathematics of liquidity provision and swap execution, and provides a natural coordinate system compatible with stochastic calculus [@shreve2004stochastic]. The continuous log-axis generalizes the discrete tick-based models used in existing concentrated liquidity AMMs while enabling tighter theoretical analysis.

**Three-layer architecture.** We defined a principled separation between the continuous mathematical model (Layer 0), the discretized on-chain representation (Layer 1), and the approximation routines for transcendental functions (Layer 2). This layered approach clarifies the sources of numerical error and enables modular analysis: properties proved at the continuous level carry over to the discrete implementation with explicit, bounded degradation.

**Formal fixed-point specification.** We provided abstract specifications for the $\exp_2$ and $\log_2$ routines required by on-chain execution, along with explicit error bounds. The main convergence theorem (Theorem 6.8) establishes that the discrete fixed-point implementation approaches the continuous model as precision increases, with error scaling as $O(2^{-F})$ in the number of fractional bits.

**Unified execution semantics.** We developed a unified execution model that combines concentrated liquidity market maker (CLMM) and central limit order book (CLOB) liquidity on a common price axis. The deterministic node execution ordering rule (Rule 7.1) ensures consistent, arbitrage-free execution across both venues, while the price path continuity lemma (Lemma 7.3) guarantees that execution traces are well-behaved.

**Spot-anchored perpetuals.** We introduced a novel derivative design in which perpetual swap positions are marked to the on-chain spot price and hedged atomically against the spot market at trade time. This construction eliminates basis risk and funding rate mechanisms by design (Theorem 9.9), transferring the burden of price convergence from external arbitrageurs to the protocol itself.

**Formal guarantees.** We established a comprehensive set of formal results for the derivative layer, including:

- No-arbitrage uniqueness of the spot-anchored mark price (Proposition 9.2)
- Residual hedging error bounds under discrete execution (Theorem 9.3)
- Liquidation correctness and path regularity (Theorem 9.6)
- Fee allocation and wealth conservation invariants (Proposition 9.7)
- Computational complexity bounds for hedged perp trades (Theorem 9.8)
- Margin dynamics and collateral invariants (Section 9.12)
- Insurance fund solvency conditions (Theorem 9.15)
- Determinism of concurrent liquidations (Theorem 9.17)
- Dynamic stability under leverage constraints (Theorem 9.19)

Together, these results provide a rigorous foundation for implementing and reasoning about complex DeFi mechanisms.

### 11.2 Limitations

We acknowledge several limitations of the present work that scope the applicability of our results:

**Single trading pair.** The formal model considers a single token pair $(A, B)$ throughout. Real DEX deployments support hundreds or thousands of trading pairs, often with shared quote assets and routing through intermediate tokens. Extending the framework to multi-pair settings introduces cross-market effects (e.g., triangular arbitrage constraints, correlated liquidations across pairs) that are not captured by our single-pair analysis.

**Idealized execution model.** We assume that trades execute atomically and instantaneously at the prices determined by the AMM curve and order book state. In practice, on-chain execution involves:

- **Transaction ordering:** Transactions within a block may be reordered by block producers, creating opportunities for front-running and sandwich attacks.
- **MEV (Maximal Extractable Value):** Sophisticated actors can extract value by observing pending transactions and strategically inserting their own orders [@daian2020flash; @flashbots2021mev; @kulkarni2022theory; @babel2023clockwork].
- **Latency and finality:** Cross-chain or L2 deployments introduce additional latency between order submission and execution.

Our model abstracts away these concerns, treating execution as deterministic given the pre-trade state. A complete implementation must address MEV resistance through mechanisms such as encrypted mempools, batch auctions, or commit-reveal schemes.

**Gas and compute costs.** We analyze computational complexity in terms of asymptotic operation counts (Theorem 9.8) but do not model the specific gas costs or compute unit budgets of any particular blockchain. The practical feasibility of our design depends on the target execution environment:

- **EVM chains:** Gas costs for storage reads/writes and arithmetic operations constrain the complexity of on-chain logic.
- **Solana:** Compute unit limits and account data constraints impose different tradeoffs.
- **App-specific chains:** Custom execution environments (e.g., Cosmos SDK chains) may relax some constraints while introducing others.

**Continuous liquidity assumption.** Several results assume a continuous, well-behaved liquidity density $\lambda(l)$. In practice, liquidity is provided by discrete LPs who may withdraw at any time, creating potential gaps or discontinuities in the liquidity profile. While our band-based discretization addresses this to some extent, extreme market conditions could violate the smoothness assumptions underlying some error bounds.

**Simplified market microstructure.** The model does not capture several features of real market microstructure:

- **Informed vs. uninformed order flow:** We do not distinguish between noise traders and informed traders, nor model adverse selection costs to LPs.
- **Dynamic LP behavior:** LPs are assumed to passively provide liquidity according to a fixed profile, rather than actively repositioning in response to market conditions.
- **Cross-venue arbitrage:** Interactions with external markets (centralized exchanges, other DEXs) are not modeled.

**Funding rate elimination scope.** The funding rate elimination theorem (Theorem 9.9) applies to our specific spot-anchored construction where hedging occurs atomically at trade time. Alternative derivative designs with delayed or batched hedging would reintroduce basis dynamics and potentially require funding mechanisms.

### 11.3 Future Directions

We outline several directions for future research and development:

#### 11.3.1 Multi-asset extension

Extending the framework to support multiple trading pairs introduces several challenges:

**Shared liquidity and routing.** Many DEX designs allow liquidity to be shared across pairs (e.g., through a common quote asset) and support multi-hop routing (e.g., trading $X \to Y$ via $X \to A \to Y$). The log-axis representation would need to be extended to a multi-dimensional space or a graph of pairwise log-price coordinates.

**Cross-pair derivatives.** Derivatives on baskets, spreads, or indices (e.g., a BTC/ETH ratio perpetual) require mark prices derived from multiple spot markets. The spot-anchoring principle could be extended by hedging against a portfolio of underlying assets.

**Correlated risk management.** Margin and liquidation systems must account for correlations between positions across pairs. A portfolio margin system based on log-price coordinates could leverage the additive structure of our representation.

**Triangular arbitrage constraints.** For three tokens $A$, $B$, $C$, the log-prices must satisfy consistency constraints: $l_{A,B} + l_{B,C} + l_{C,A} = 0$ (up to fees). Incorporating these constraints into the execution and hedging logic is an open problem.

#### 11.3.2 Implementation on specific platforms

The abstract specification in this paper is designed to be platform-agnostic. Concrete implementations would need to address platform-specific concerns:

**Solana.** The high throughput and low latency of Solana [@yakovenko2018solana] make it attractive for implementing the unified CLOB+CLMM execution model. Key considerations include:

- Account structure for storing liquidity bands and order book state
- Compute unit budgets for complex swap and hedging operations
- Integration with existing Solana DeFi infrastructure (e.g., Serum/OpenBook [@openbook2022] order books, Orca [@orca2022whirlpools] and Raydium [@raydium2023clmm] pools)

**Ethereum L2s.** Optimistic and ZK rollups provide lower gas costs while inheriting Ethereum's [@wood2014ethereum] security. Considerations include:

- Storage costs for liquidity state
- Proving costs for ZK implementations
- Sequencer centralization and its implications for MEV

**App-specific chains.** Building a dedicated blockchain (e.g., using Cosmos SDK or Substrate) allows custom execution logic but requires bootstrapping validator sets and liquidity. This approach may be suitable for specialized derivative applications.

**Formal verification.** The layered specification in this paper is designed to facilitate formal verification of implementations. Future work could develop machine-checked proofs (e.g., in Coq, Lean, or Isabelle) that concrete smart contract code satisfies the abstract specification.

#### 11.3.3 Empirical validation and backtesting

The theoretical guarantees in this paper should be validated against empirical data:

**Historical backtesting.** Simulating the protocol against historical price and volume data would reveal:

- Realized hedging errors versus theoretical bounds
- Insurance fund accumulation/depletion under various market conditions
- Liquidation cascade dynamics during historical stress events (e.g., March 2020, May 2021, November 2022)

**Agent-based simulation.** Modeling strategic behavior by LPs, traders, and liquidators could reveal emergent dynamics not captured by the single-agent analysis:

- LP repositioning strategies and their impact on liquidity depth
- Arbitrageur behavior and MEV extraction
- Attack vectors and their profitability

**Testnet deployment.** Deploying the protocol on testnets with synthetic or forked mainnet state would provide practical experience with implementation challenges and edge cases.

#### 11.3.4 Extensions to options and structured products

Section 9.11 sketched extensions to dated futures and options. A full treatment would include:

**Option pricing and hedging.** Adapting Black-Scholes [@black1973pricing; @merton1973theory] or local volatility models to the log-axis representation, with explicit treatment of discrete hedging and fixed-point errors.

**Volatility surfaces.** Representing implied volatility as a function of log-moneyness $l - l_{\text{atm}}$ and time to expiry, with appropriate interpolation and extrapolation schemes.

**Exotic derivatives.** Path-dependent options (barriers, lookbacks, Asians) require careful treatment of the discrete execution model and its interaction with continuous-time pricing theory.

**Structured products.** Combining spot, perp, and option positions into structured products (e.g., principal-protected notes, yield enhancement strategies) with formal risk characterization.

#### 11.3.5 MEV resistance and fair ordering

Addressing the MEV concerns noted in Section 11.2 is critical for practical deployment:

**Batch auctions.** Collecting orders over a time window and executing them at a uniform clearing price can eliminate front-running within batches.

**Encrypted mempools.** Threshold encryption schemes can hide order details until block finalization, preventing content-based MEV extraction.

**Fair ordering protocols.** Consensus mechanisms that enforce fair ordering (e.g., based on commit timestamps) can reduce sequencer discretion.

**MEV redistribution.** Mechanisms that capture and redistribute MEV to users or LPs (e.g., MEV-Share, OEV auctions) could be integrated with the protocol.

#### 11.3.6 Governance and parameter optimization

The protocol involves numerous parameters (precision bits $F$, margin ratios, leverage caps, fee rates, insurance fund targets) that affect system behavior:

**Parameter sensitivity analysis.** Understanding how outcomes depend on parameter choices, and identifying robust parameter regimes.

**Adaptive parameters.** Mechanisms for adjusting parameters in response to market conditions (e.g., increasing margin requirements during high volatility).

**Decentralized governance.** Designing governance mechanisms for parameter updates that balance responsiveness with stability and resistance to capture.

### 11.4 Closing Remarks

The design space for decentralized exchanges remains vast and largely unexplored. This paper contributes a formal framework that unifies spot and derivative markets on a common mathematical foundation, with explicit attention to the numerical realities of on-chain execution. We hope this work provides a useful reference for researchers and practitioners seeking to build more rigorous, transparent, and efficient DeFi infrastructure.

The logarithmic representation at the heart of our design is not merely a technical convenience but reflects a deep connection between the multiplicative nature of financial prices and the additive structure amenable to digital computation. By making this connection explicit and proving that it can be preserved through layers of discretization and approximation, we aim to bridge the gap between elegant mathematical models and practical blockchain implementations.

As decentralized finance continues to mature, we believe that formal methods and rigorous specification will become increasingly important—not as academic exercises, but as practical tools for building systems that users can trust. The stakes are high: DeFi protocols collectively secure billions of dollars in user assets, and failures can be catastrophic and irreversible. We offer this work as a step toward a more principled approach to DeFi protocol design.

---
