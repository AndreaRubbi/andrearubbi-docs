# Theory & Interactive Visualizations

## Overview

SP-FM (Shortest-Path Flow Matching) is a conditional flow-matching framework that improves out-of-distribution (OOD) generalisation by **conditioning both the base distribution and the flow field** on the input descriptor. Standard conditional flow matching fixes the base distribution to a single Gaussian and conditions only the velocity field on the descriptor, effectively learning independent flows for each condition. SP-FM replaces this fixed base with a learnable, condition-dependent Gaussian mixture, trained jointly with a shortest-path velocity field.

> **Paper:** [Shortest-Path Flow Matching with Mixture-Conditioned Bases for OOD Generalization to Unseen Conditions](https://arxiv.org/abs/2601.11827) — Rubbi, Akbarnejad, Sanian, Yazdan Parast, Asadollahzadeh, Amani, Akhtar, Cooper, Bassett, Paavolainen, Vakili, Liò, Lotfollahi (2026)

---

## The problem with fixed bases

Standard conditional flow matching samples every condition from the same fixed Gaussian $\mathcal{N}(0, I)$. The velocity field alone must learn the entire mapping from this single base to each target distribution. When a novel condition arrives at test time, there is no mechanism to leverage structural similarities with training conditions — the model can only hope that the conditioned decoder generalises.

This design implicitly treats each condition as an independent estimation problem. The consequences are predictable: performance degrades sharply on unseen conditions, and the degradation worsens as the data dimensionality $D$ grows.

---

## The SP-FM insight

SP-FM replaces the fixed base with a **condition-dependent Gaussian mixture**. The key observation is simple: when two conditions are similar (two drugs with related chemical structures, two genetic perturbations affecting the same pathway), their target distributions are also likely to be similar. By learning a base distribution that reflects this similarity structure, the flow only needs to correct for residual differences rather than learning the entire transport from scratch.

The architecture has two learnable components:

- **$h_\Theta(y)$**: an MLP that predicts GMM mode positions from the condition descriptor
- **$h_p(y)$**: an MLP that predicts mixture weights, passed through a Gumbel-Softmax for differentiable mode selection

At test time, given an unseen descriptor $y_{\text{test}}$, SP-FM predicts a base distribution $\mu_{\text{GMM}}(h_\Theta(y_{\text{test}}), h_p(y_{\text{test}}))$ already positioned near the expected target, then transports it via the learned velocity field.

### Training objectives

SP-FM is trained with two complementary losses:

**Objective 1 — OT flow matching:** Align the learned velocity field $v_\theta$ with the optimal transport path between the target $\rho_n$ and its projected base:

$$\mathcal{L}_{\text{OT}} = \mathbb{E}\left[\sum_{s=1}^{S} \left\| v\left(t, (1-t)x_s^{(0)} + tx_{\tau(s)}, y_n\right) - (x_{\tau(s)} - x_s^{(0)}) \right\|^2 \right]$$

**Objective 2 — Geodesic length:** Encourage the predicted base to remain close to the target by minimising geodesic length on the Wasserstein manifold:

$$\mathcal{L}_{\text{geo}} = \mathbb{E}\left[\sum_{s=1}^{S} \left\| x_{\tau(s)} - x_s^{(0)} \right\|^2 \right]$$

Training alternates between updating the flow model and the base distribution through a scheduled planner with warm-up, alternating, and cool-down phases.

### Explore the architecture

The interactive 3D visualisation below shows how SP-FM's MLP components are wired together — from the condition descriptor input through the base distribution prediction and velocity field to the final generated samples.

<div style="position: relative; width: 100%; height: 0; padding-bottom: 75%; overflow: hidden;">
  <iframe
    src="../spfm_mlp_wired_3d_v2.html"
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border: none;"
    allowfullscreen>
  </iframe>
</div>

!!! tip
    Use your mouse to rotate, zoom, and interact with the 3D visualization.

---

## Degrees of freedom analysis

The theoretical justification for SP-FM comes from analysing the **degrees of freedom** in the transport problem on the Wasserstein manifold.

Assume the target distribution $\rho$ is a GMM with $J$ components and fixed mode locations $\Gamma \in \mathbb{R}^{J \times D}$. The base distribution is a GMM with $I$ modes. At test time, the flow $V^* \in \mathbb{R}^{I \times J}$ must be identified from the predicted base parameters. The key result:

!!! abstract "Statement 1 (from the paper)"
    The velocity field $V$ is identified up to $J - ID$ degrees of freedom. If $I \geq \lceil J/D \rceil$, then $V$ is uniquely identified.

The three constraint sources that pin down the transport:

| Source | Count | Origin |
|--------|-------|--------|
| Barycentre conditions | $ID$ | Each mode position $\Theta_i^*$ is the weighted average of target modes it transports to |
| Optimality conditions | $I$ | Equal transport cost across all active modes |
| **Total constraints** | **$ID + I$** | |
| **Variables** | **$I + J$** | Non-zero entries of $V^*$ |
| **Degrees of freedom** | **$J - ID$** | What remains underdetermined |

The formula $\text{DoF} = J - ID$ reveals two critical properties:

**1. Each additional mode reduces DoF by $D$.** Adding one GMM component eliminates $D$ degrees of freedom from the transport problem. This is why multi-modal bases outperform unimodal ones — it is not just about expressiveness, it is about identifiability.

**2. Higher dimensions help SP-FM.** When $D$ is large, each mode contributes proportionally more constraints. This explains the empirical observation that SP-FM's advantage over vanilla CFM becomes *more* pronounced in high dimensions — the opposite of what one might expect.

### The I = 1 collapse

When $I = 1$ (vanilla CFM), the dual optimal transport problem becomes ill-defined. The single dual variable $z_1$ cancels out of the objective entirely:

$$\sup_{z} z_1 - z_1 \sum_{j=1}^{J} q_j + \sum_{j=1}^{J} q_j \|\Theta_1 - \Gamma_j\|^2 = \text{const}$$

The base provides no information about the transport structure. The model must predict all $J$ mixture weights directly from the descriptor — a problem whose generalisation error scales with $J$.

### Interactive exploration

The visualisation below lets you explore these relationships. The left panel shows the constraint budget as you vary $I$, $D$, and $J$. The right panel shows the Wasserstein manifold geometry: the purple dot is the fixed Gaussian base (long geodesic to target), the orange dots are GMM modes (short geodesics). As you increase $I$ or $D$, watch the modes tighten around the target and the degrees of freedom drop to zero.

<div style="position: relative; width: 100%; height: 0; padding-bottom: 60%; overflow: hidden;">
  <iframe
    src="../spfm_manifold_dof_3d.html"
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border: none;"
    allowfullscreen>
  </iframe>
</div>

---

## Why this matters for high-dimensional biology

Single-cell transcriptomics typically operates in $D = 50$–$500$ dimensions (PCA components of the gene expression space). At these scales, the fixed-base approach is at a severe disadvantage:

- With $D = 50$ and $J = 50$ target components, even $I = 1$ leaves $\text{DoF} = 0$. But this is misleading — the $I=1$ dual is ill-defined regardless (Statement 1, §2 in the paper).
- With $D = 500$, a single additional mode ($I = 2$) eliminates 500 degrees of freedom. The constraint budget is dominated by the $ID$ term, so high-dimensional data makes each mode disproportionately informative.

This is precisely what the ablation experiments confirm (Figure 3b in the paper): CFM's energy distance grows by over an order of magnitude between 50 and 500 PCA components, while SP-FM maintains stable performance across the full range.

---

## Summary

| Property | Vanilla CFM | SP-FM |
|----------|-------------|-------|
| Base distribution | Fixed $\mathcal{N}(0, I)$ | Learned GMM conditioned on $y$ |
| Transport distance | Long (origin to target) | Short (near-target mode to target) |
| DoF at test time | $J - D$ (ill-defined for $I=1$) | $J - ID$ (well-posed for $I \geq \lceil J/D \rceil$) |
| High-$D$ behaviour | Degrades | Improves |
| OOD mechanism | Hope decoder generalises | Base encodes condition similarity |
