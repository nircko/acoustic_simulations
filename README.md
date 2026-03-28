# Acoustic Simulations



## TDOA Microphone Array Calibration and Robust Source Localization

This repository studies **TDOA-based microphone calibration** and **source localization** under realistic acoustic conditions, with an emphasis on:

- nonlinear calibration error propagation,
- robustness to measurement noise,
- downstream localization accuracy,
- and simulation tools for testing difficult scenarios such as **non-line-of-sight (NLOS)** propagation.

A key idea throughout the project is that **good initialization alone is not enough**. In nonlinear TDOA calibration, the final error is determined by both:

1. the **local geometry and conditioning** of the problem, and  
2. the **measurement noise** that enters the residual model.

To study these effects in a controlled way, we combine:

- a nonlinear TDOA calibration / optimization pipeline,
- Cramer--Rao style diagnostics,
- and **Python k-Wave acoustic simulations** of a **rotating microphone array** mounted on a **PMMA slab**, including difficult propagation conditions.

---

## What this repository is about

We work on the calibration and use of a microphone array for acoustic source localization.

The general workflow is:

1. **Calibrate the microphone array geometry** from TDOA measurements.
2. **Quantify uncertainty** using Jacobian-based error propagation and Fisher / CRB analysis.
3. **Use the calibrated array for source localization**.
4. **Test robustness in simulation**, including rotating-array operation and non-line-of-sight support.

The project focuses on the fact that microphone geometry uncertainty does not disappear after calibration. If we later localize sources using an imperfect array estimate, the remaining geometry error becomes a floor on downstream localization accuracy.

---

## Main goals

The repository has four main goals:

### 1. Microphone geometry calibration

We estimate microphone positions from TDOA constraints and additional geometric information when available. The calibration problem is nonlinear and typically requires:

- gauge fixing,
- careful initialization,
- repeated or multi-pose measurements,
- and robust optimization.

### 2. Error propagation analysis

We analyze how measurement noise propagates into estimated microphone and source positions. The central question is:

> Does the final calibration error follow the initialization perturbation, or does it follow the measurement noise and geometry?

The answer is that, at leading order, the local estimation covariance is controlled by the **measurement model**, the **Jacobian**, and the **weighting**, not by the initialization perturbation itself.

### 3. Downstream source localization

We study what happens after calibration, when we treat the estimated microphone array as known and use it to localize sources. This stage is sensitive to residual calibration error, especially in weak or ambiguous geometries.

### 4. Physics-based acoustic simulation

We use **Python k-Wave simulation** to generate realistic acoustic data for testing the localization pipeline in conditions that are difficult to model analytically, especially:

- rotating-array acquisition,
- multipath,
- occlusion,
- non-line-of-sight propagation,
- and structured support materials such as a PMMA slab.

---

## Repository structure

A typical repository structure is organized around:

- calibration and optimization code,
- Jacobian / CRB utilities,
- simulation scripts,
- experiment configs,
- and plotting / debugging tools.

For example, utilities such as:

- `src/utils/cramer_rao_calibration.py`

are used to evaluate local Fisher-information style bounds and compare empirical error against linearized prediction.

Depending on the experiment setup, other modules typically handle:

- residual construction,
- Jacobian evaluation,
- unknown-state packing / unpacking,
- weighting,
- optimization,
- and simulation orchestration.

---

## Core technical idea

The main conceptual distinction in this repository is the difference between:

- **initialization noise**, and
- **measurement noise**.

Suppose we initialize the optimizer at

$$
\hat{\boldsymbol{\theta}}^{(0)} = \boldsymbol{\theta}^{\mathrm{true}} + \boldsymbol{\eta},
$$

where $\boldsymbol{\eta}$ is the starting-point perturbation.

This perturbation matters because it affects the nonlinear solve:

- which basin we enter,
- whether we reach a good solution,
- whether we get trapped in a poor local minimum.

But once we linearize near a converged solution, the first-order estimation error is controlled by the measurement noise:

$$
\mathbf{e}(\boldsymbol{\theta}^{\mathrm{true}} + \delta\boldsymbol{\theta})
\approx
\mathbf{J}\,\delta\boldsymbol{\theta} - \boldsymbol{\epsilon},
$$

where:

- $\mathbf{e}$ is the stacked residual vector,
- $\mathbf{J}$ is the Jacobian of the residuals with respect to the unknowns,
- $\boldsymbol{\epsilon}$ is the measurement-noise vector.

Then the first-order Gauss--Newton solution satisfies

$$
\mathbf{J}^\top \mathbf{W} \mathbf{J}\,\delta\boldsymbol{\theta}
=
\mathbf{J}^\top \mathbf{W}\,\boldsymbol{\epsilon},
$$

so that the local covariance is approximately

$$
\mathrm{Cov}(\delta\boldsymbol{\theta})
\approx
(\mathbf{J}^\top \mathbf{W} \mathbf{J})^{-1}
\mathbf{J}^\top \mathbf{W}\,
\mathbf{\Sigma}_{\epsilon}\,
\mathbf{W}\mathbf{J}
(\mathbf{J}^\top \mathbf{W} \mathbf{J})^{-1}.
$$

If the weighting is matched to the noise model, this simplifies to

$$
\mathrm{Cov}(\delta\boldsymbol{\theta})
\approx
(\mathbf{J}^\top \mathbf{W} \mathbf{J})^{-1}.
$$

This means:

> The final local uncertainty is driven by **measurement noise and geometry**, not by the size of the initialization perturbation.

That is why the final calibration error can be larger than the initialization noise and still be perfectly consistent with theory.

---

## TDOA calibration model

For source $i$ and microphone $j \neq 0$, a standard TDOA forward model in time units is

$$
h_{ij}(\boldsymbol{\theta})
=
\frac{\|\mathbf{x}_j - \mathbf{s}_i\| - \|\mathbf{x}_0 - \mathbf{s}_i\|}{c},
$$

where:

- $\mathbf{x}_0$ is the reference microphone,
- $\mathbf{x}_j$ is microphone $j$,
- $\mathbf{s}_i$ is source $i$,
- $c$ is the speed of sound.

The measured TDOA is

$$
\Delta t_{ij}^{\mathrm{meas}}
=
\Delta t_{ij}^{\mathrm{true}} + \epsilon_{ij},
$$

and the residual is

$$
e_{ij}(\boldsymbol{\theta})
=
h_{ij}(\boldsymbol{\theta}) - \Delta t_{ij}^{\mathrm{meas}}.
$$

All residuals are stacked into a single vector and minimized in weighted least squares, optionally together with additional geometric constraints such as panel-distance terms.


## Rotating microphone array for NLOS support

A major motivation for the rotating-array simulation is **non-line-of-sight support**.

In NLOS scenarios, direct-path assumptions are often violated. A source may be:

- partially occluded,
- visible only through indirect propagation,
- or observed with distorted first-arrival structure.


Slab model

The array (mounted device) is simulated together with a PMMA-like properties slab, which serves as a structured support / propagation element in the acoustic model.

The purpose of including the slab is to better match realistic experimental hardware and to capture non-ideal acoustic effects that can influence TDOA extraction.

Depending on the exact simulation parameters, the PMMA slab may contribute to:

- altered propagation paths,
- interface reflections,
- transmission delays,
- and pose-dependent acoustic behaviour.

This matters because a localization method that performs well only in ideal free-field conditions may fail once material effects are introduced.

By simulating the array together with the PMMA slab, we can evaluate whether the localization pipeline remains stable under more realistic operating conditions.



## Solver and diagnostics

We examine whether the optimized residuals are consistent with the assumed noise model.


We inspect the Jacobian and the normal matrix

$$
\mathbf{J}^\top \mathbf{W} \mathbf{J}
$$



We estimate a local lower bound on covariance to understand the best accuracy we can expect under the model assumptions.


When ground truth is available in simulation, we compare empirical estimation error to the scale predicted by the linearized covariance.

