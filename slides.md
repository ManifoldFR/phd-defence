---
# You can also start simply with 'default'
theme: neversink
title: "Real-time constrained trajectory optimisation in robotics: theory, implementation, and applications"
info: |
  ## My PhD defence

  Presentation slides for developers.

addons:
  - tldraw

# apply unocss classes to the current slide
class: text-left
# https://sli.dev/features/drawing
drawings:
  persist: false
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# take snapshot for each slide in the overview
overviewSnapshots: true

# First slide
layout: cover
hideInToc: true
---

### Real-time constrained trajectory optimisation in robotics: theory, implementation, and applications

**PhD Defence**

Wilson Jallet<br>
*LAAS-CNRS Gepetto & INRIA Willow*<br>
<Email v="wjallet@laas.fr" />
<Email v="wilson.jallet@inria.fr" />

<div class="abs-br m-6 flex gap-2">
  <button @click="$slidev.nav.openInEditor()" title="Open in Editor" class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon:edit />
  </button>
  <img src="/logo_LAASCNRS_transparent.png" alt="laas" class="h-10"></img>
  <img src="/Logo_universiteToulouse.svg" alt="ut" class="h-10"></img>
  <img src="/ENS-PSL.png" alt="ens" class="h-10"></img>
  <img src="/inr_logo_rouge.svg" alt="inria" class="h-10"></img>
</div>


---
layout: section
---

# Introduction


---
layout: default
---

## From industrial robots... to mobile and fully autonomous robots

<figure class="w-full">
  <img src="/robot-timeline.drawio.png" alt="timeline" class="place-self-center" />
</figure>

**Classical industrial robotics**: known, fixed, closed environments, **hard-coded** routines.

<div v-click class="ns-c-tight">

**Towards more autonomous robots:**

* **unstructured** environments require **adaptiveness**
  * planning and mapping (e.g. SLAM);
  * real-time [online control]{.text-orange-600 .font-bold}
* agile manufacturing: **tasks & objects change**, more complex **open** environments

</div>

<div class="absolute bottom-0 text-0.6em">

  Sources (left-to-right): KUKA: [Wikimedia](https://commons.wikimedia.org/wiki/File:KUKA_Industrial_Robots_IR.jpg); Dyson 360 robot: [dyson.com.au](https://edition.cnn.com/cnn-underscored/reviews/dyson-360-vis-nav-robot-vacuum); Spot picture: *me*; Atlas: [Boston Dynamics](https://www.youtube.com/watch?v=-e1_QhJ1EhQ)
</div>

:: note ::

Sources: KUKA: [Wikipedia](https://commons.wikimedia.org/wiki/File:KUKA_Industrial_Robots_IR.jpg); Dyson 360 robot: [dyson.com.au](https://edition.cnn.com/cnn-underscored/reviews/dyson-360-vis-nav-robot-vacuum)

<!--
Classical industrial robots:
* we know everything about envs
* nothing changes, **same tasks**
* can use **hard-coded** routines for picking, placing, etc

Autonomy means:
* variability, complex environments
*
-->

---
layout: top-title
---
:: title ::

## Robot control: the problem

:: content ::

<div class="grid grid-cols-2 gap-4">
  <div>
    <img src="/control-loop.drawio.svg" alt="control law" class="h-80 place-self-center"></img>
  </div>
  <div class="mt-10">

  * Sensor data: **noisy**
  * Control law: **known transformation** from sensor data
  * Robot: **somewhat known**
  * World: **imperfect model**, **high variability**, **physics** (contact, friction...)

  </div>
</div>

---
layout: top-title
---

:: title ::

## Robot control: two ways

:: content ::

<div class="grid grid-cols-2">
  <div>

  ## Optimal control

  <SlidevVideo loop autoplay class="w-full">
    <source src="/atlas-grip-clip.mp4" type="video/mp4">
  </SlidevVideo>

  <img src="/optctrl.drawio.svg" alt="control law" class="w-80 place-self-center"></img>

  </div>
  <div>

  ## Reinforcement learning

  <SlidevVideo loop autoplay class="w-full">
    <source src="/parkour_no_audio_480p.mp4" type="video/mp4">
  </SlidevVideo>

  <img src="/deeprl.drawio.svg" alt="control law" class="h-40 place-self-center"></img>

  </div>
</div>

<!--
TWO PHILOSOPHIES FOR MOTION GENERATION:
* OCP
* RL

OPTIMAL CONTROL:
* Approach is used to control the plant

REINFORCEMENT LEARNING:
* Model is STILL used for learning thru simulator
* Modern approaches add **more and more information about the underlying system/invariants**

**Hence** model-based control is still quite relevant in some ways
-->

---
layout: top-title
hideInToc: true
---

:: title ::

## Optimal / model-predictive control

:: content ::

<div class="grid grid-cols-3">
  <div class="grid-col-span-1">

  * Model-based mapping from sensors, to **state**, to **control**
  * For $u_t$: solve an **optimal control problem** (OCP).
  * Close the loop: **model-predictive control** (MPC)

  </div>

  <div class="grid-col-span-2 center">
    <img src="/optctrl-zoom.drawio.svg" alt="control law" class="w-100 place-self-center" />
  </div>
</div>

---
layout: top-title
---

::title::

## What is the optimal control problem (OCP) ?

::content::

A **mathematical optimisation model** to drive a system according to a **performance objective**:
$$
\begin{aligned}
  \underset{\bm{x}, \bm{u}}{\operatorname{\mathrm{minimise}}}~%
  &J(\bm{x}, \bm{u}) = \sum_{t=0}^{N-1} \ell_t(x_t, u_t) + \ell_N(x_N) \\
  \mathrm{s.t.}~%
  &f_t(x_t, u_t, x_{t+1}) = 0 \\
  &\textcolor{red}{h_t(x_t, u_t) \leqslant 0} \\
  &\textcolor{red}{h_N(x_N) \leqslant 0}.
\end{aligned}
$$

<v-clicks>

* **Unknowns:** System states $\bm{x} = (x_0,\ldots,x_N)$, control inputs $\bm{u} = (u_0,...,u_{N-1})$
* $J(\bm{x}, \bm{u})$ the **cost function**
* $f_t = 0$: *discrete-time dynamics* $(x_t,u_t) \mapsto x_{t+1}$
* $h_t\leqslant 0$ and $h_N\leqslant 0$: **path and terminal constraints**.

</v-clicks>

<!--
Dynamics written this way **includes implicit time-stepping integrators** i.e. IRK

WHAT WE KNOW IN ROBOTICS:
* solving without constraints, unconstrained OCPs

WHAT I DO:
* add constraint handling in OCPs
-->

---
layout: top-title
---

::title::

## OCPs are large-scale problems

::content::

The OCP is a **nonlinear program** with *lots* of variables but with a **specific structure**.

<div v-click class="grid grid-cols-3 gap-3">

  <img src="/meshcat-ur5-transparent.png" alt="ur5" class="w-fit place-self-center grid-col-span-1" />

  <div class="grid-col-span-2">

  These are the dimensions of an OCP with $N=100$ knots on a simple UR5 robot arm with *just* the dynamics (see left):
  <v-click at="2">

  | **State size** | **Control size** | **Horizon $N$** | Total num. variables | Num. constraints |
  | -------------- | ---------------- |---------------- | -------------------- | ---------------- |
  |  14            |   7              |     100         | 2114                 | 1414             |

  **Hessian of cost:** 2114 $\times$ 2114 = ~4.5M entries (~36MB)!  
  **Only 0.99% are nonzero.**

  **Jacobian of constraints:** 2114 $\times$ 1414 = ~3.0M entries (~24MB)!  
  **Only 1.6% are nonzero.**
  </v-click>
  </div>

</div>

<!--
Napkin math:
* 4.5 M float64 = **36 MEGABYTES**
* 3.0 M float64 = **24 MEGABYTES**

IF WE TAKE A QUADRUPED (SOLO-12) INSTEAD
* nx=36
* nu=12
* **187MB** FOR HESSIAN
* **140MB** FOR JACOBIAN

**For reference: L2 Cache size Apple M3**
* 16MB on Pro performance cores
* 32MB on Ultra
-->

---
layout: top-title
---

::title::

## Why make tailored solvers?

::content::

<div class="grid grid-cols-2">

<div>

**Large-scale** and **structured** problem means:

<v-clicks>

* Encountering large sparse matrices (like on the right)... **exploit structure** to be fast!
* *Generic* large-scale solvers like IPOPT handle *generic* sparsity patterns
  * use a *generic* sparse solver (MUMPS, MA57...), can't exploit the specific pattern...
  * designed and tuned for *generic* problems
* A tailored solver:
  * faster by exploiting **specific structure**, become [real-time.]{.text-red-600 .font-bold}

</v-clicks>
</div>

<div class="center">
  <img src="/ur5_short_sparsity.svg" alt="ur5_pattern" class="w-100 center" />

  <p>

  **Sparsity pattern for a simple reach task on UR5 robot.** The pattern repeats itself for larger $N$.
  </p>
</div>
</div>

<!--
On UR5:

* shown only short horizon because longer makes the plot unreadable
* matrix has about 28k entries (most of them are zero, nonzeros are blue)
* mix of dense and sparse structures

WHY FASTER?
* Dense blocks deserve **dense** operations (to get SIMD instructions on modern CPUs...)
-->

---

## Strategies for nonlinear constraints

The OCP is a *nonlinear problem* with *nonlinear constraints*.
**How to deal with this?**

<v-click>

Literature for strategies on this goes back years. Main families of methods are:

<div class="ns-c-tight">

* (sequential) quadratic programming (SQP)[^4]
* interior-point methods (IPM)[^2]
* [augmented Lagrangian methods (ALM)]{.font-bold .text-orange}[^1][^3]

</div>

**All can be implemented with specific attention to OCPs.**

</v-click>

<div v-click class="ns-c-tight">

Our choice is ALM because:

* no difficult aspects for implementation
* fairly straightforward to **warm-start** (unlike IPM)
* versatile (implicit dynamics, equality constraints, inequality constraints, more...)

</div>

<style>
  .footnote-item {
    font-size: 11px;
  }
  .footnote-item p {
    font-size: 11px;
    margin-top: 4px;
    margin-bottom: 0;
  }
  .footnotes {
    position: absolute;
    bottom: 0;
    margin-right: 4em;
  }
</style>

[^1]: D. P. Bertsekas, Constrained Optimization and Lagrange Multiplier Methods. Athena Scientific, 1996.
[^2]: S. Mehrotra, ‘On the Implementation of a Primal-Dual Interior Point Method’, SIAM J. Optim., vol. 2, no. 4, pp. 575–601, Nov. 1992, doi: 10.1137/0802028.
[^3]: M. J. Powell, ‘A method for nonlinear constraints in minimization problems’, Optimization, pp. 283–298, 1969.
[^4]: P. Gill, W. Murray, and M. Saunders, ‘SNOPT: An SQP Algorithm for large-scale constrained optimization’, SIAM Journal on Optimization, vol. 12, pp. 979–1006, Apr. 2002, doi: 10.2307/20453604.


---

## Problem statement

**Questions:**

Can we design a new solver for constrained OCPs that:

* is simple
* can solve **multiple problems well**
* can run in **real-time** for MPC?

<hr/>
<v-click>

**Contributions**

* A new, primal-dual ALM algorithm for solving constrained OCPs
* A performant library that can be used for **solving problems offline** *or* for **real-time control**
* A parallel linear solver for OCP

</v-click>

---

## Related work: solvers for OCPs

| *Unconstrained* | DDP, FDDP[^0] |
| --- | ----- |
| SQP | CSQP[^jordana] |
| Interior-point | FATROP[^1], HPIPM[^Hpipm] (for QPs) |
| ALM | ALTRO/ALTRO-C[^3][^4] (ALM+SQP refinement step), QPALM-OCP[^qpalm] (for QPs) |

[^0]: C. Mastalli et al., ‘Crocoddyl: An Efficient and Versatile Framework for Multi-Contact Optimal Control’, 2020 IEEE International Conference on Robotics and Automation (ICRA), pp. 2536–2542, May 2020, doi: 10.1109/ICRA40945.2020.9196673.
[^1]: L. Vanroye, A. Sathya, J. De Schutter, and W. Decré, ‘FATROP : A Fast Constrained Optimal Control Problem Solver for Robot Trajectory Optimization and Control’, in 2023 IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS), Detroit, MI, USA: IEEE, Oct. 2023. doi: 10.1109/IROS55552.2023.10342336.
[^2]: G. Lantoine and R. P. Russell, ‘A Hybrid Differential Dynamic Programming Algorithm for Constrained Optimal Control Problems. Part 1: Theory’, J Optim Theory Appl, vol. 154, no. 2, pp. 382–417, Aug. 2012, doi: 10.1007/s10957-012-0039-0.
[^3]: T. A. Howell, B. E. Jackson, and Z. Manchester, ‘ALTRO: A Fast Solver for Constrained Trajectory Optimization’, in 2019 IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS), Macau, China: IEEE, Nov. 2019, pp. 7674–7679. doi: 10.1109/IROS40897.2019.8967788.
[^4]: B. E. Jackson, T. Punnoose, D. Neamati, K. Tracy, R. Jitosho, and Z. Manchester, ‘ALTRO-C: A Fast Solver for Conic Model-Predictive Control’, in 2021 IEEE International Conference on Robotics and Automation (ICRA), May 2021, pp. 7357–7364. doi: 10.1109/ICRA48506.2021.9561438.
[^jordana]: A. Jordana, S. Kleff, A. Meduri, J. Carpentier, N. Mansard, and L. Righetti, ‘Stagewise Implementations of Sequential Quadratic Programming for Model-Predictive Control’, Dec. 2023.
[^Hpipm]: G. Frison and M. Diehl, ‘HPIPM: a high-performance quadratic programming framework for model predictive control’, IFAC-PapersOnLine, vol. 53, no. 2, pp. 6563–6569, Jan. 2020.
[^qpalm]: K. F. Løwenstein, D. Bernardini, and P. Patrinos, ‘QPALM-OCP: A Newton-Type Proximal Augmented Lagrangian Solver Tailored for Quadratic Programs Arising in Model Predictive Control’, IEEE Control Systems Letters, vol. 8, pp. 1349–1354, 2024, doi: 10.1109/LCSYS.2024.3410638.

<style>
  .footnote-item {
    font-size: 9px;
  }
  .footnote-item p {
    font-size: 9px;
    margin-top: 4px;
    margin-bottom: 0;
  }
  .footnotes {
    position: absolute;
    bottom: 0;
    margin-right: 4em;
  }
</style>

---
layout: top-title-two-cols
hideInToc: true
columns: is-8
---

::title::

## Publications

::left::

<div class="!children:text-0.56em -mt-4">

### Main and related contributions

1. **WJ**, N. Mansard, and J. Carpentier, ‘Implicit Differential Dynamic Programming’, in 2022 International Conference on Robotics and Automation (ICRA), Philadelphia, United States: IEEE, May 2022. doi: 10.1109/ICRA46639.2022.9811647.
2. **WJ**, A. Bambade, N. Mansard, and J. Carpentier, ‘ProxNLP: a primal-dual augmented Lagrangian solver for nonlinear programming in Robotics and beyond’, in 6th Workshop on Legged Robots, Philadelphia, Pennsylvania, United States, May 2022.
3. **WJ**, A. Bambade, N. Mansard, and J. Carpentier, ‘Constrained Differential Dynamic Programming: A primal-dual augmented Lagrangian approach’, in 2022 IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS), Kyoto, Japan.
4. **WJ**, A. Bambade, E. Arlaud, S. El-Kazdadi, N. Mansard, and J. Carpentier, ‘PROXDDP: Proximal Constrained Trajectory Optimization’, 2023, *submitted to IEEE Transactions on Robotics (T-RO), (under revision)*
5. **WJ**, E. Dantec, E. Arlaud, N. Mansard, and J. Carpentier, ‘Parallel and Proximal Constrained Linear-Quadratic Methods for Real-Time Nonlinear MPC’, in Proceedings of Robotics: Science and Systems (RSS), Delft, Netherlands, Jul. 2024. doi: 10.15607/RSS.2024.XX.002.
6. E. Ménager, A. Bilger, **WJ**, J. Carpentier, and C. Duriez, ‘Condensed semi-implicit dynamics for trajectory optimization in soft robotics’, in IEEE International Conference on Soft Robotics (RoboSoft), San Diego (CA), United States.
7. E. Dantec, **WJ**, and J. Carpentier, ‘From centroidal to whole-body models for legged locomotion: a comparative analysis’, 2024 IEEE-RAS International Conference on Humanoid Robots, Nancy, France: IEEE, Jul. 2024.

### Side contributions

8. Q. Le Lidec, **WJ**, I. Laptev, C. Schmid, and J. Carpentier, ‘Enforcing the consensus between Trajectory Optimization and Policy Learning for precise robot control’, in 2023 IEEE International Conference on Robotics and Automation (ICRA), May 2023, pp. 946–952.
9. Q. Le Lidec, **WJ**, L. Montaut, I. Laptev, C. Schmid, and J. Carpentier, ‘Contact Models in Robotics: a Comparative Analysis’, IEEE Transactions on Robotics, vol. 40, pp. 3716–3733, Jul. 2024, doi: 10.1109/TRO.2024.3434208.

</div>

::right::

<div class="flex flex-wrap">
<div>
  <img src="/trombi/antoine.png" alt="antoine" class="w-30"/></div>
<div>
  <img src="/trombi/etienne.png" alt="etienne" class="w-30"/></div>
<div>
  <img src="/trombi/sarah.png" alt="sarah" class="w-30"/></div>
<div>
  <img src="/trombi/ewen.jpg" alt="ewen"   class="w-30"/></div>
</div>


<style>
h3 {
  font-weight: bold;
}
</style>

---
layout: section
hideInToc: true
---

## Overview

<hr>

<Toc columns=1 minDepth="1" maxDepth="1" mode="all" />

<!--
We just went over the introduction...

Next is part II: proximal algorithms
-->

---
layout: section
---

# Proximal algorithms for Nonlinear Programming and Optimal Control

<hr>

<Toc columns=1 minDepth="2" maxDepth="2" mode="onlyCurrentTree" />

---

## Nonlinear programming with the augmented Lagrangian method

Consider a mathematical program:

$$
\begin{aligned}
  \min_{z\in \mathbb{R}^n}~& f(z) \\
  \mathrm{s.t. }~& g(z) = 0 \\
                 & h(z) \leqslant 0
\end{aligned}
$$

where $f$ is the objective and $g: \mathbb{R}^n \to \mathbb{R}^m$, $h:\mathbb{R}^n \to \mathbb{R}^p$ are the constraints.

---

### The augmented Lagrangian method (ALM)

**Algorithm outline.** See Rockafellar '73[^1]

1. minimise the **augmented Lagrangian function $\mathcal{L}_\mu$**:
   $$
   \begin{aligned}
     \textcolor{blue}{z^{k+1}} \in
     \argmin_z \mathcal{L}_{\mu_k}(z, \lambda^k, \nu^k) := f(z) &+ (\lambda^k)^\top g(z) + \tfrac{1}{2\mu_k} \| g(z) \|^2 \\
     &+ \tfrac{1}{2\mu_k}\| [h(z) + \mu_k\nu^k]_+ \|^2 - \tfrac{\mu_k}{2}\| \nu^k \|^2.
   \end{aligned}
   $$
2. set $\begin{aligned}\lambda^{k+1} &= \lambda^k + \tfrac{1}{\mu_k}g(\textcolor{blue}{z^{k+1}} ), \\ \nu^{k+1} &= [\nu^k + \tfrac{1}{\mu_k}h(\textcolor{blue}{z^{k+1}} )]_+\end{aligned}$ **(proximal iteration)**.
3. (optionally) decrease $\mu_k$
4. $k \leftarrow k+1$ and go back to Step 1.

<div class="text-red-600">

**Question** How to minimise $\mathcal{L}_\mu$?

</div>

[^1]: R. T. Rockafellar, ‘A dual approach to solving nonlinear programming problems by unconstrained optimization’, Mathematical Programming, vol. 5, no. 1, pp. 354–373, Dec. 1973, doi: 10.1007/BF01580138.

<style>
  .footnote-item {
    font-size: 12px;
  }
  .footnotes {
    position: absolute;
    bottom: 0;
    margin-right: 4em;
  }
</style>

<!-- 
Rockafellar 73 is an extension of ALM to inequality constrained problems
 -->

---

### ALM: computing directions

$\mathcal{L}_\mu$ is only *continuously differentiable* but not *twice*-differentiable.

No Newton method but [SEMI-SMOOTH NEWTON]{.text-orange .font-bold}, using a "**generalised Hessian**".

<v-click>

**Generalised Hessian:** given $P$ = projection matrix on active constraints, we can compute
$$
\begin{equation*}
  H_{\mu_k} = H + \textcolor{red}{\frac{1}{\mu_k}} (A^\top A + B^\top P B)
  \in \partial^2_z \mathcal{L}_{\mu_k},
  H = \nabla_z^2 \mathcal{L}_0,
  A = {\partial g}/{\partial z},  
  B = {\partial h}/{\partial z}.
\end{equation*}
$$

Get direction $\delta z$ from
$$
\begin{equation}
  \boxed{
  H_{\mu_k}\delta z = -\nabla_z\mathcal{L}_{\mu_k}(z, \lambda^k, \nu^k).
  }
\end{equation}
$$

</v-click>

<v-click>

* **What happens as $\mu_k \to 0$?** <span v-click=3 class="text-red-500 font-bold">(conditioning issues)</span>
* Do we **need** to send $\mu_k \to 0$? To converge to a minimum? <span v-click=3 class="text-green font-bold">No!</span>

</v-click>

<!--
* Keep in mind the red inverse mu term
* This term creates problems because H_mu's conditioning gets worse as mu -> 0

THERE ARE THINGS WE CAN TWEAK WITH NOMINAL ALGO.
 -->


---

#### First tweak: Primal-dual system

**Rearrange the terms in Newton equation.**
Given any $(\lambda, \nu)$, we can rewrite $(1)$ in primal-dual form using the block system:

$$
  \underbrace{%
  \begin{bmatrix}
    H & A^\top & B^\top \\
    A & -\mu_k I & \\
    PB &  & -\mu_k I
  \end{bmatrix}
  }_{=\mathcal{K}_k}
  \begin{bmatrix}
    \delta z \\ \delta\lambda \\ \delta\nu
  \end{bmatrix} =
  -\begin{bmatrix}
    \nabla f + A^\top \lambda + B^\top \nu \\
    g + \mu_k(\lambda^k - \lambda) \\
    [h + \mu_k\nu^k]_+ - \mu_k\nu
  \end{bmatrix}
$$

* possible to **symmetrise the system** ($P$ is a projection matrix)
* apply some [stable factorisation routine]{.text-green-600 .font-bold} (e.g. indefinite Cholesky)
* **numerically stable** as we decrease $\mu$

---

#### Second tweak: keep $\mu$ away from 0 + inexact minimisation

**Idea from 1991 paper by Conn, Gould & Toint**[^1]

<div class="grid grid-cols-2">
<div>

* Solve ALM subproblem **inexactly** within tol. $\omega_k$
* Only accept $(\lambda^{k+1}, \nu^{k+1})$ when ALM iteration progressed on constraints
* Otherwise, decrease $\mu$ (**increase penalty**)

<hr>

<v-click>

**Original paper** just for *equality* constraints:

* used in generic NLP solver **LANCELOT**
* [Contribution:]{.font-bold .text-red-600} **we generalise** to inequalities (in papers)
* same method behind **ProxQP** (from coauthors)

</v-click>
</div>

<figure>
<img src="/papers/conn-gould-toint-1991.png" alt="conn1991" class="w-full">
</figure>
</div>

[^1]: A. Conn, N. I. M. Gould, and P. Toint, ‘A Globally Convergent Augmented Lagrangian Algorithm for Optimization with General Constraints and Simple Bounds’, SIAM Journal on Numerical Analysis, vol. 28, Apr. 1991.

<style>
  .footnote-item {
    font-size: 12px;
  }
  .footnotes {
    position: absolute;
    bottom: 0;
    margin-right: 4em;
  }
</style>

---
layout: top-title-two-cols
columns: is-5
---

::title::

## Software contribution 1: implementation in a new NLP library

::left::

* a generic NLP solver **ProxNLP**
* open-source C++ library `proxsuite-nlp`
* [support for Lie groups (not in papers)]{.text-pink-600 .italic}
* support for equality, inequality, [box & other constraints (not in papers)]{.text-pink-600 .italic}

On GitHub: https://github.com/Simple-Robotics/proxsuite-nlp/

::right::

<img src="/proxsuite-logo.png" alt="proxsuite-logo" class="w-70 place-self-center mb-2">
<img src="/proxnlp-github-page.png" alt="proxnlp-github" class="w-140">

---

### Recap

<div v-click>

* designed an ALM algorithm for nonlinear programming:
  * generic
  * simple
  * handles equality & inequality constraints
  * robust to conditioning issues

**Goal: adapt to structure of OCPs.**

</div>

<div class="absolute bottom-0 text-0.66em mr-20">

**Reference papers for this section**

1. **WJ**, A. Bambade, N. Mansard, and J. Carpentier, ‘Constrained Differential Dynamic Programming: A primal-dual augmented Lagrangian approach’, in 2022 IEEE/RSJ International Conference on Intelligent Robots and Systems
2. **WJ**, N. Mansard, and J. Carpentier, ‘Implicit Differential Dynamic Programming’, in 2022 International Conference on Robotics and Automation (ICRA), Philadelphia, United States
3. **WJ**, A. Bambade, E. Arlaud, S. El-Kazdadi, N. Mansard, and J. Carpentier, ‘PROXDDP: Proximal Constrained Trajectory Optimization’, 2023. _Submitted to Transactions on Robotics (under revision)_

</div>

---

## ProxDDP: an ALM algorithm for solving constrained OCPs

**Terminal stage:** $V_N(x) = \ell_N(x) + \frac{1}{2\mu} \| [h_N(x) + \mu\nu_e]_+ \|^2 - \frac{\mu}{2} \|\nu_e\|^2.$

**Backwards recursion:** $Q$-function
$$
\begin{equation*}
  Q_t(x,u,y,\lambda,\nu) = \ell_t(x,u) + \lambda^\top f_t(x,u,y) + \nu^\top h_t(x, u) + V_{t+1}(y)
\end{equation*}
$$

* derivatives of the regularised $Q$-function (with $w=(u,y,\lambda,\nu)$):
  $$
    \mathcal{K}_\mu = Q_{ww} =
    \underbrace{
    \begin{bmatrix}
      Q_{uu}& Q_{uy} & f_u^\top & h_u^\top \\
      Q_{yu}& Q_{yy} & f_y^\top & 0 \\
      f_u   & f_y & -\mu I & \\
      Ph_u  & 0   &        & -\mu I
    \end{bmatrix}
    }_{\textsf{same matrix as ProxNLP!}},\
    \mathbf{m}^k = Q_{w},\
    \textsf{and}\quad
    \mathbf{M}^k = Q_{wx},
  $$
* solve **as function of $\delta x$**,
  $$
    [\delta u, \delta y, \delta\lambda, \delta\nu] = -\mathcal{K}_\mu^{-1}(\mathbf{M}^k \delta x + \mathbf{m}^k )
    = \textcolor{red}{\mathbf{\Gamma}}\delta x + \textcolor{red}{\bm{\gamma}} \quad \textsf{[feedback/feedforward gains]}
  $$
* **extract quadratic model of $V_t$**

<!-- 
The matrix K_\mu is symmetrizable
* you can use Cholesky on it
* numerically stable as \mu -> 0
-->

---

#### Forward pass

**Option I:** [the linear rollout]{.text-orange-600 .font-bold}

* $(\delta x_0, \delta\lambda_0) \leftarrow$ initial condition
* iterate forward: $[\delta u_t, \delta x_{t+1}, \delta \lambda_{t+1}, \delta\nu_t] = \mathbf{\Gamma}\delta x_t + \bm{\gamma}$ (using gains).
* **Advantages:**
  * rather robust to stiff nonlinear dynamics,
  * recovers the step from the direct multiple-shooting literature

<hr>

<v-click>

**Option II:** [the nonlinear rollout]{.text-pink-600 .font-bold} (for *explicit* dynamics $y = f^\mathrm{ex}(x, u)$)

* **same as linear**, but correct $\delta x$ using **true dynamics**
* **Advantage:** dynamics satisfied closely after few iterates! (might be a drawback)
* [Contribution:]{.text-orange-600 .font-bold} ALM + nonlinear rollout yields **multiple shooting**

</v-click>

---
layout: top-title-two-cols
columns: is-7
---

:: title ::

## Software contribution 2: aligator, a nonlinear OC library

:: left ::

<img src="/aligator-github-page.png" alt="ali-github" class="w-140">

:: right ::

`aligator` is a C++ OCP library for solving OCPs in robotics and beyond.

* implements **ProxDDP** algorithm (reimplements FDDP as baseline)
* using **Pinocchio for robot modelling**
* comes with [Python]{.font-bold .text-blue} bindings
* <span v-mark.underline="0">suitable for MPC!</span>
* under **active** development: 27k lines of code so far.

Open-source on GitHub! https://github.com/Simple-Robotics/aligator

---
layout: top-title
---

:: title ::

## Benchmarks

:: content ::

*New in revision of T-RO submission.*

* Compare against other nonlinear solvers: OCP solver ALTRO, generic solver IPOPT
* Implemented wrappers around ALTRO and IPOPT

**Methodology.**

<v-clicks>

* Solve problems within tolerance $\epsilon_\text{tol}$ for constraints & optimality
* Assess multiple configurations:
  * **ALTRO:** change subproblem tolerance $\omega$
  * **IPOPT:** change Hessian approximation
  * **ProxDDP:** different initial AL penalty $\mu_0 > 0$, linear vs. nonlinear rollout, linesearch parameters...
* *more details in the revised T-RO submission.*

</v-clicks>

<!--
Only present 2 benchmark problems
* More info in the revised submission paper (preprint online soon)
-->

---

### UR10 "ballistic" task

<div class="ns-c-tight">

* Underactuated system
* Hard constraint on projectile final's position, on joint torques & velocities (no ballistic cost...),

</div>

<video controls loop autoplay class="h-10/12 place-self-center">
  <source src="/ur10_mug_throw.mp4" type="video/mp4">
</video>

---

### UR10 "ballistic" task: solve times vs problems solved

<img src="/bench/ur10_ballistic_solve_times.svg" alt="ur10_ballistic" class="place-self-center">

<!-- ### UR10 "ballistic" task: performance profile

<img src="/bench/ur10_ballistic_perfprofile_time.svg" alt="ur10_ballistic" class="place-self-center"> -->

---

### SOLO-12 "Yoga" task

A very nonlinear task for a whole-body model, 4 contact phases.

<video controls loop autoplay class="h-10/12 place-self-center">
  <source src="/solo12_lift_paw.mp4" type="video/mp4">
</video>

---

### SOLO-12 "Yoga" task: solve times vs problems solved

<img src="/bench/solo_yoga_solve_times.svg" alt="solo_times" class="w-150 place-self-center" />

**Notes:**

* ALTRO failed on all instances
* ProxDDP w/ nonlinear rollout always failed; results on plot are *linear* rollout only

<!--
ALTRO is not present there because... no instances converged at all.
-->

<!-- ### SOLO-12 "Yoga" task: performance profile

<img src="/bench/solo_yoga_perfprofile_time.svg" alt="solo_times" class="place-self-center" />

<div class="absolute bottom-0">
  
  [Performance ratio:]{.italic} how slower you are wrt fastest solver (in wall time) on a given problem.
</div> -->

---

### UR5 "Avoidance" task

A collision avoidance task.

<video controls loop autoplay class="h-10/12 place-self-center">
  <source src="/ur_slalom.mp4" type="video/mp4">
</video>

---

### UR5 "Avoidance" task: solve times vs problems solved

<img src="/bench/ur_slalom_solve_times.svg" alt="ur10_ballistic" class="place-self-center">

**Note:** IPOPT failed on all instances.

---

### ProxDDP and benchmarks: conclusion

[Contributions recap:]{.text-orange .font-bold}

* reformulate ALM for numerical OC with **stable** primal-dual systems
* **generalise** the heuristic from LANCELOT (to inequalities + OCP setting)
* benchmark against other ALM attempt (ALTRO) and well-known generic NLP solver (IPOPT)

<hr>
<v-click>

**Benchmarks:** ProxDDP overall performs well on the benchmark suite.

* Competitive with IPOPT on the test set (in solve time)
* More robust across problems than ALTRO

Benchmarks (including wrappers for ALTRO & IPOPT) available online:

<p class="text-center text-blue">

https://github.com/Simple-Robotics/aligator-bench
</p>

</v-click>

---
layout: section
---

# Going parallel for constrained OCPs

<hr>

<Toc columns=1 minDepth="2" maxDepth="2" mode="onlyCurrentTree" />

---

## Recap

**So far:**

* Established DDP recursion-like method for solving constrained OCPs
* Have to **solve larger linear systems at each stage**
* Provided open-source **fast C++ implementations**
* Validated on test bench against other solvers

---

## A limitation of Riccati: multicore architectures

DDP-type methods (or any method based on **Riccati**), have a *major* limitation:

* inherently **linear in time** $\mathcal{O}(N)$
* no way of exploiting **multicore architectures**! (except evaluating e.g. gradients in parallel)

<figure>
  <img src="/riccati-serial.drawio.svg" alt="serial_riccati" class="place-self-center w-150"/>
</figure>

<v-click>
<span class="text-blue-700">

**Questions**

* How to parallelise the Riccati recursion?
* What kind of speedup can we expect?
* What are the tradeoffs?

</span>
</v-click>

<div class="absolute bottom-0 text-0.6em mr-20">
  
  **Reference papers for this section:**

  1. **WJ**, E. Dantec, E. Arlaud, N. Mansard, and J. Carpentier, ‘Parallel and Proximal Constrained Linear-Quadratic Methods for Real-Time Nonlinear MPC’, in *Proceedings of Robotics: Science and Systems*, Delft, Netherlands, Jul. 2024
  2. E. Dantec, **WJ**, and J. Carpentier, ‘From centroidal to whole-body models for legged locomotion: a comparative analysis’, presented at the *2024 IEEE-RAS International Conference on Humanoid Robots*, Nancy, France: IEEE, Jul. 2024

</div>

---

## Our approach

* Focus on the structure-exploiting **linear solver**, not the nonlinear problem
* Our aim: **exact** method to solve the linear problem/Newton step.
  * iterative approaches (e.g. conjugate gradient) $\Rightarrow$ appropriate for e.g. GPUs[^1]
* The main idea is **"Parametrise to parallelise"**
* [Contributions:]{.text-orange .font-bold}
  * algorithm for **exact method**
  * deployment for MPC
  * modern C++ implementation ready for use

[^1]: E. Adabag, M. Atal, W. Gerard, and B. Plancher, ‘MPCGPU: Real-Time Nonlinear Model Predictive Control through Preconditioned Conjugate Gradient on the GPU’, in 2024 IEEE International Conference on Robotics and Automation (ICRA), Yokohama, Japan, May 2024.

<style>
  .footnote-item {
    font-size: 12px;
  }
  .footnote-item p {
    font-size: 12px;
    margin-top: 4px;
    margin-bottom: 0;
  }
  .footnotes {
    position: absolute;
    bottom: 0;
    margin-right: 4em;
  }
</style>

---

### Parametrise

Consider linear-quadratic problem
$$
\begin{aligned}
  \mathcal{E}_0(x_0, \textcolor{red}{\theta}) =
  \min_{\bm{x},\bm{u}}~&
  \sum_{t=0}^{N-1} \ell_t(x_t, u_t) + \ell_N(x_N; \textcolor{red}{\theta})  \\
  \mathrm{s.t.}~& A_tx_t + B_tu_t + E_tx_{t+1} + f_t = 0 \\
                & C_tx_t + D_tu_t + d_t = 0
\end{aligned}
$$
where the $\ell_t$ are all quadratic, and **the terminal cost $\ell_N$ is parametric**:
$$
  \ell_N(x, \textcolor{red}{\theta}) = \frac{1}{2}
  \begin{bmatrix}x \\ \textcolor{red}{\theta} \end{bmatrix}^\top
  \begin{bmatrix}
    Q_N & \Phi_N \\
    \Phi_N^\top & \Gamma_N
  \end{bmatrix}
  \begin{bmatrix}x \\ \textcolor{red}{\theta} \end{bmatrix}
  + q_N^\top x + \gamma_N^\top \textcolor{red}{\theta}.
$$

<figure>
  <img src="/riccati-parametric.drawio.svg" alt="parametric_riccati" class="w-full" />
</figure>

**Property.** $\mathcal{E}_0$ is quadratic in $(x_0, \theta)$.

---

### Parallelise

**Idea: make the co-state $\lambda$ at each "cut" point into parameters.**

<figure>
  <img src="/riccati-parallel.drawio.svg" alt="parallel_riccati" class="w-full" />
</figure>

**NOTE:** the parametric Riccati solve adds **overhead**.

---

#### The general case

<figure>
  <img src="/riccati-parallel-gen.drawio.svg" alt="parallel_riccati" class="w-200 place-self-center" />
</figure>

**The consensus problem:** block-sparse problem with *block-tridiagonal pattern:*
<div v-click class="grid grid-cols-2">
<div>

$$
\begin{bmatrix}
  \circ & \maltese \\
  \maltese & \blacksquare & \blacktriangle & \\
          & \blacktriangle & \circ & \maltese \\
          &          & \maltese & \blacksquare & \ddots \\
          &          &          & \ddots & \ddots & \maltese \\
          &&& & \maltese & \blacksquare
\end{bmatrix}
\begin{bmatrix}
  \lambda_0 \\ x_0 \\ \lambda_{i_1} \\ x_{i_1} \\ \vdots \\ x_{i_J}
\end{bmatrix} = -\begin{bmatrix}
  x^0 \\ p_0 \\ \sigma_0 \\ p_{i_1} \\ \vdots \\ p_{i_J}
\end{bmatrix}
$$
</div>

<div class="mt-10">

  * Warrants **specific solver routine.**
  * We use the **block-tridiagonal algorithm** (classic in the literature).

</div>

</div>


<!--  
* Essentially a block LDU factorization, very similar to Riccati itself.
-->

---

#### Recovering the step

* Condensed problem $\Rightarrow$ intermediate states and $(x_{i_j})_j$ and co-states $(\lambda_{i_j})_j$
* Reinject in **parametric segments**
  * recover **states, controls, path multipliers** from $t=i_j$ to $t=i_{j+1}$ **(IN PARALLEL)**

<div v-click class="mt-10">

**Questions:**

1. Can we get a Riccati gain $\mathbf{\Gamma}$ for $t=0$ (for MPC purposes)? <span v-click=2 class="text-green"> **YES** </span>
2. Can we get a (parallel) **nonlinear rollout**? <span v-click=3 class="text-orange"> **NOT REALLY...** </span>

</div>

---

## Results

### Benchmarks

<div class="grid grid-cols-2">

<figure class="grid-col-span-1">
  <img src="/mac-gar-bench-timings.png" alt="parallel_riccati"
  class="h-110" />
</figure>

<div class="grid-col-span-1 mt-10">

  **Synthetic benchmark.** Timings on an M1 Mac Studio Ultra desktop (16 Perf cores, 4 Eco cores).
  
  * **Problem dimensions:** $n_x=36$, $n_u=12$ (same as SOLO-12 wole-body).
  * Speedup is **not** linear
  * Speedup depends on **number of cores** *and* **horizon length**

</div>
</div>

<!--
* time horizon varies
* if you add more cores, you start losing performance if your problem is too short
* speedup isn't linear due to **overhead** in parametric Riccati solve
 -->

---
layout: two-cols-title
columns: is-4
---

:: title ::
### Deploying multi-phase constrained MPC

:: left ::

**Whole-body jumping on Unitree GO-2:**

<div>

* Fixed flight/contact phases
* **Constraints:** joint position, torque limits, landing $z(t_\textsf{contact}) = 0$...
* Receding-horizon control, **whole problem rotates (including constraints)**
* 1kHz loop with Riccati gain
* Real-time on modern CPU, using the **parallel Riccati recursion (~5ms/iteration)**

</div>

:: right ::

<video controls loop autoplay class="w-screen place-self-center mb-2">
  <source src="/quadru_jump_edit.mp4" type="video/mp4">
</video>

<div class="grid grid-cols-2 gap-6 w-full">
<p class="italic translate-x-50 -translate-y-20 translate rotate-270">Ewen Dantec</p>
<img src="/trombi/ewen.jpg" alt="ewen" class="h-36"/>
</div>

<!--
**For reference**

* 20ms is often enough for walking quadrupeds
* Ewen did biped locomotion on TALOS at ~12ms
* Part of the trick is to *not* reallocate the entire solver data but cycle it
-->

---
layout: section
---

# Conclusion and perspectives

---

## Contributions

* An appropriate variant of ALM can work quite well for solving OCPs
  * Our solver is seeing use for MPC and solving offline OCPs
* Basic blocks for benchmarking OCP solvers
* **Implementations (and heuristics) matter**
* Revamped Riccati to create a **parallel Riccati-based solver for OCPs**
  * our parallel solver gives great improvements on large systems!

---

## Perspectives / A roadmap

### Short term

<v-clicks>

* [More comprehensive benchmarking]{.text-orange .font-bold}
  * Compare more solvers
  * Compare more problems
  * Standardise benchmarks around Pinocchio and AMPL? JuMP? CasADi?
* Need for **battle testing** on real systems
  * How to benchmark MPC?
* Solver for other OCP **topologies**:
  * Tree-structured problems / scenario-tree MPC
  * Cyclic OCPs (e.g. for walking)

</v-clicks>

---

### Long term

<v-clicks>

* [Automatic differentiation]{.text-orange .font-bold} and machine/deep learning applications e.g. **policy learning**
  * **One contribution in this direction** (with Q. Le Lidec)
  * Use ProxDDP as a *differentiable motion generation layer*?
  * See work on differentiating (infeasible) QPs by Bambade[^qplayer]

* [Look at nonsmooth constraints:]{.text-orange .font-bold}
  * Time optimisation
  * Complementarity constraints: contact & frictional contact
  * Look at randomised smoothing?

</v-clicks>

[^qplayer]:  A. Bambade, F. Schramm, A. Taylor, and J. Carpentier, ‘Leveraging augmented-Lagrangian techniques for differentiating over infeasible quadratic programs in machine learning’, The Twelfth International Conference on Learning Representations (ICLR 2024), Oct. 2023.

<style>
  .footnote-item {
    font-size: 12px;
  }
  .footnote-item p {
    font-size: 12px;
    margin-top: 4px;
    margin-bottom: 0;
  }
  .footnotes {
    position: absolute;
    bottom: 0;
    margin-right: 4em;
  }
</style>

<!--
Nonsmooth constraints:
* revamp the UR10 problem with optimizing launch time
-->

---
layout: cover
hideInToc: true
---

# Thank you!

---
hideInToc: true
---

# Backup slides

## ALM and the prox-point iteration

**Instead...** Rewrite problem as a **saddle-point**:
$$
  \min_{z\in \mathbb{R}^n} \max_{\lambda \in \mathbb{R}^m, \nu \geq 0} f(z) + \lambda^\top g(z) + \nu^\top h(z).
$$

Use the **proximal-point algorithm** in the *dual* problem:

<v-click>

1. **"concavify" in $(\lambda, \nu)$** (proximal mapping):
   $$
     \Pi_\mu^+(\textcolor{red}{\lambda_e, \nu_e}) = \arg_\lambda \min_z \max_{\lambda,\nu\geq 0} f(z) + \lambda^\top g(z) + \nu^\top h(z)
     - \frac{\mu}{2} \| (\lambda,\nu) - (\textcolor{red}{\lambda_e,\nu_e}) \|_2^2
   $$
2. **iterate the proximal mapping: $(\lambda^{k+1}, \nu^{k+1}) = \Pi_\mu^+(\lambda^k, \nu^k)$**
3. (optionally) decrease $\mu$
4. **repeat** from Step 1.

</v-click>

---

## ALM and more generic constraints

$$
\begin{aligned}
  \min_z~& f(z) \\
  \mathrm{s.t.}~ &g(z) \in C
\end{aligned}
$$
where $C$ is *convex* subset of $\mathbb{R}^m$.

**Optimality conditions**
$$
\begin{aligned}
  \nabla f(z) + g_z^\top \lambda = 0& \\
  \lambda \in N_C(g(z))&
\end{aligned}
$$
Define $\psi(u) := \imath_C(u)$, so $\partial\psi(u) = N_C(u)$.
The second line $\Leftrightarrow 0 \in -g(z) + \partial\psi^*(\lambda)$, and the KKT conditions are equivalent to:
$$
  0 \in
  \mathcal{T}(z, \lambda) =
  \begin{bmatrix}
    \nabla f(z) + g_z(z)^\top\lambda \\
    -g(z) + \partial\psi^*(\lambda)
  \end{bmatrix}
$$

---

Resolvent in $\lambda$:
$$
  (z', \lambda') =
  {\underbrace{((0, \mathrm{id}) + (1, \mu^{-1})\mathcal{T})}_{= \mu\mathcal{T}_\mu}}^{-1}
  (z_e, \lambda_e).
$$

$$
\begin{equation*}
  0 \in
  \begin{bmatrix}
    \nabla f(z') + g_z(z')^\top \lambda' \\
    \mu(\lambda'-\lambda_e) - g(z') + \partial\psi^*(\lambda')
  \end{bmatrix}
\end{equation*}
$$
and the second line is (single-valued when $C$ convex)
$$
\begin{aligned}
  (\mathrm{id} + \mu^{-1}\partial\psi^*)(\lambda') \ni \lambda_e + \mu^{-1}g(z')
  &\Leftrightarrow
  \boxed{
  \lambda' = \mathrm{prox}_{\mu^{-1} \psi^*}(\lambda_e + \mu^{-1}g(z')).
  }
\end{aligned}
$$

We use the prox operator's property called **Moreau's identity**:
$$
\begin{aligned}
  \mathrm{prox}_{\mu^{-1}h}(u) = \tfrac{1}{\mu}(\mathrm{id} - \mathrm{prox}_{\mu h})(\mu u).
\end{aligned}
$$
And get
$$
\begin{aligned}
  \lambda' = \tfrac{1}{\mu}(\mathrm{id}-\mathrm{proj}_C)(g(z') + \mu\lambda_e) := \textcolor{blue}{\lambda_\mu^+(z', \lambda_e)}
\end{aligned}
$$

---

The resolvent system becomes
$$
  \boxed{
  \mathcal{T}_\mu(z', \lambda'; \lambda_e) :=
  \begin{bmatrix}
    \nabla f(z') + g_z^\top \lambda' \\
    \mu(\lambda_e - \lambda^+_\mu(z', \lambda_e))
  \end{bmatrix} = 0.
  }
$$

How to do solve this and do ALM? **Ingredients:**

* **Semi-smooth Newton**
* Clarke subderivative $I-\Sigma$ of $\mathrm{prox}_{\mu \psi} = \mathrm{proj}_{C}$ (**exists when $C$ convex**).
* Semi-smooth system:
  $$ \begin{bmatrix}
    H & g_z^\top \\
    \Sigma g_z & -\mu \Sigma 
  \end{bmatrix} \begin{bmatrix}
    \delta z \\ \delta \lambda
  \end{bmatrix} =
    \mathcal{T}_\mu(z, \lambda; \lambda_e) $$
* Merit function: generalised AL function works:
  $$
    \mathcal{L}_\mu(z, \lambda_e) = f(z) + \frac{1}{2}\mathrm{dist}_C^2(g(z) + \mu\lambda_e) - \frac{\mu}{2}\|\lambda_e\|^2.
  $$

---

## On heuristics

A collection of heuristics from IPOPT:

<img src="/ipopt-heuristics.png" alt="ur10_ballistic" class="place-self-center">

---

## Preliminary results on other OCP topologies

<div class="grid grid-cols-2">

<img src="/gar-cyclic-lqr-2d.png" alt="cyclic-2d" class="place-self-center w-120" />

<div class="mt-20">

* 2D cyclic problem, $N=20$ knots
* cyclic constraint $x_0 = x_{20}$
* no initial condition $x_0$
* no terminal constraint on $x_{20}$

</div>
</div>
