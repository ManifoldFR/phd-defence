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
  <img src="/robot-timeline.drawio.png">
</figure>

**Classical industrial robotics**: fixed (closed) environment, everything known, **hard-coded** routines.

<div v-click class="ns-c-tight">

**Towards more autonomous robots:**

* **unstructured** environments require **adaptiveness**
* mobile robots: advances in planning and mapping (e.g. SLAM);
* legged robots: advances in real-time **online control**
* agile manufacturing: tasks & objects change, more complex environments

</div>

<div class="absolute bottom-0 text-0.6em">

  Sources: KUKA: [Wikipedia](https://commons.wikimedia.org/wiki/File:KUKA_Industrial_Robots_IR.jpg); Dyson 360 robot: [dyson.com.au](https://edition.cnn.com/cnn-underscored/reviews/dyson-360-vis-nav-robot-vacuum)
</div>

:: note ::

Sources: KUKA: [Wikipedia](https://commons.wikimedia.org/wiki/File:KUKA_Industrial_Robots_IR.jpg); Dyson 360 robot: [dyson.com.au](https://edition.cnn.com/cnn-underscored/reviews/dyson-360-vis-nav-robot-vacuum)

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
    <source src="/eth-anymal-parkout-clip.mp4" type="video/mp4">
  </SlidevVideo>

  <img src="/deeprl.drawio.svg" alt="control law" class="h-40 place-self-center"></img>

  </div>
</div>

<!--
OPTIMAL CONTROL:
* The model is used to control the plant

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
  * For $u_t$: solve an <span v-mark.circle.red>**optimal control problem**</span> (OCP).
  * Close the loop: **model-predictive control** (MPC)

  </div>

  <div class="grid-col-span-2 center">
    <img src="/optctrl-zoom.drawio.svg" alt="control law" class="w-full place-self-center" />
  </div>
</div>

---
layout: top-title
---

::title::

## What is the optimal control problem (OCP) ?

::content::

It is a **mathematical optimisation model** for driving a system according to a **performance objective**:
$$
\begin{aligned}
  \underset{\bm{x}, \bm{u}}{\operatorname{\mathrm{minimise}}}~%
  &J(\bm{x}, \bm{u}) = \sum_{t=0}^{N-1} \ell_t(x_t, u_t) + \ell_N(x_N) \\
  \mathrm{s.t.}~%
  &x_0 = \textcolor{Red}{x^0} \\
  &f_t(x_t, u_t, x_{t+1}) = 0 \\
  &g_t(x_t, u_t) \leqslant 0 \\
  &g_N(x_N) \leqslant 0.
\end{aligned}
$$

<v-clicks>

* **Unknowns:** System states $\bm{x} = (x_0,\ldots,x_N)$, control inputs $\bm{u} = (u_0,...,u_{N-1})$
* $J(\bm{x}, \bm{u})$ the **cost function**
* $f_t = 0$: *discrete-time dynamics* $(x_t,u_t) \mapsto x_{t+1}$
* $g_t\leqslant 0$ and $g_N\leqslant 0$: **path and terminal constraints**.  
  No $g_t, g_N \Rightarrow$ OCP is "**unconstrained**".

</v-clicks>

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

  **Hessian of cost:** 2114 $\times$ 2114 = ~4.5M entries!
  **Only 0.99% are nonzero.**

  **Jacobian of constraints:** 2114 $\times$ 1414 = ~3.0M entries!
  **Only 1.6% are nonzero.**
  </v-click>
  </div>

</div>

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
* Need for **large-scale** nonlinear solvers
* *Generic* solvers like IPOPT handle *generic* sparsity patterns
  * use a *generic* sparse solver (MUMPS, MA57...), can't exploit the specific pattern...
  * pragmatic design and tuning for *generic* problems
* A tailored solver:
  * faster by exploiting **specific structure**, become **[real-time.]{.text-red}**

</v-clicks>
</div>

<div class="center">
  <img src="/ur5_short_sparsity.svg" alt="ur5_pattern" class="w-full center" />

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

* (sequential) quadratic programming
* interior-point methods
* [augmented Lagrangian methods (ALM)]{.font-bold .text-orange}

**All can be implemented with specific attention to OCPs.**

</v-click>

<div v-click class="ns-c-tight">

Our choice is ALM because:

* easy to understand
* simple to implement and pragmatic choice
* versatile (equality constraints, inequality constraints, more...)
* fairly straightforward to **warm-start**

</div>

<!-- [^1]: J. Nocedal and S. J. Wright, Numerical optimization, 2nd ed. in Springer series in operations research. New York: Springer, 2006. -->

<!--
There are a lot of methods. Best reference is Nocedal's book.

ALM is easy to understand and get up and running.
-->

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

---

## Problem statement

**Questions:**

* Can we use a variant of the **augmented Lagrangian method (ALM)** to design a new solver for constrained OCPs?
* Can we **keep it simple**, and **solve multiple problems well**?
* Can we **make it real-time** for model-predictive control (MPC) ?

<hr/>

**Contributions**

* A new, primal-dual ALM algorithm for solving OCP 
* A performant library that can be used for **solving problems offline** *or* for **real-time control**
* A parallel linear solver for OCP

---

## Related work: solvers for OCPs

<div class="ns-c-tight">

* DDP/FDDP[^0] - only for **unconstrained** OCPs

**For constrained OCPs:**

* FATROP[^1]: [interior-point method]{.text-blue}
* CSQP[^jordana]: [sequential quadratic programming (SQP)]{.text-blue}
* HPIPM[^Hpipm]: structure-exploiting [interior-point]{.text-blue} for QPs, used inside of Acados
* ALTRO[^3]/ALTRO-C[^4]: based on ALM, but lacks simplicity and performance

**Performance and robustness are still an issue.**

</div>

[^0]: C. Mastalli et al., ‘Crocoddyl: An Efficient and Versatile Framework for Multi-Contact Optimal Control’, 2020 IEEE International Conference on Robotics and Automation (ICRA), pp. 2536–2542, May 2020, doi: 10.1109/ICRA40945.2020.9196673.
[^1]: L. Vanroye, A. Sathya, J. De Schutter, and W. Decré, ‘FATROP : A Fast Constrained Optimal Control Problem Solver for Robot Trajectory Optimization and Control’, in 2023 IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS), Detroit, MI, USA: IEEE, Oct. 2023. doi: 10.1109/IROS55552.2023.10342336.
[^2]: G. Lantoine and R. P. Russell, ‘A Hybrid Differential Dynamic Programming Algorithm for Constrained Optimal Control Problems. Part 1: Theory’, J Optim Theory Appl, vol. 154, no. 2, pp. 382–417, Aug. 2012, doi: 10.1007/s10957-012-0039-0.
[^3]: T. A. Howell, B. E. Jackson, and Z. Manchester, ‘ALTRO: A Fast Solver for Constrained Trajectory Optimization’, in 2019 IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS), Macau, China: IEEE, Nov. 2019, pp. 7674–7679. doi: 10.1109/IROS40897.2019.8967788.
[^4]: B. E. Jackson, T. Punnoose, D. Neamati, K. Tracy, R. Jitosho, and Z. Manchester, ‘ALTRO-C: A Fast Solver for Conic Model-Predictive Control’, in 2021 IEEE International Conference on Robotics and Automation (ICRA), May 2021, pp. 7357–7364. doi: 10.1109/ICRA48506.2021.9561438.
[^jordana]: A. Jordana, S. Kleff, A. Meduri, J. Carpentier, N. Mansard, and L. Righetti, ‘Stagewise Implementations of Sequential Quadratic Programming for Model-Predictive Control’, Dec. 2023.
[^Hpipm]: G. Frison and M. Diehl, ‘HPIPM: a high-performance quadratic programming framework for model predictive control’, IFAC-PapersOnLine, vol. 53, no. 2, pp. 6563–6569, Jan. 2020.

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
hideInToc: true
---

## Publications

This thesis has led to the following publications:

<div class="!children:text-0.56em">

### Main and related contributions

1. **WJ**, N. Mansard, and J. Carpentier, ‘Implicit Differential Dynamic Programming’, in 2022 International Conference on Robotics and Automation (ICRA), Philadelphia, United States: IEEE, May 2022. doi: 10.1109/ICRA46639.2022.9811647.
2. **WJ**, A. Bambade, N. Mansard, and J. Carpentier, ‘ProxNLP: a primal-dual augmented Lagrangian solver for nonlinear programming in Robotics and beyond’, in 6th Workshop on Legged Robots, Philadelphia, Pennsylvania, United States, May 2022. Accessed: Oct. 10, 2022.
3. **WJ**, A. Bambade, N. Mansard, and J. Carpentier, ‘Constrained Differential Dynamic Programming: A primal-dual augmented Lagrangian approach’, in 2022 IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS), Kyoto, Japan, Oct. 2022.
4. **WJ**, A. Bambade, E. Arlaud, S. El-Kazdadi, N. Mansard, and J. Carpentier, ‘PROXDDP: Proximal Constrained Trajectory Optimization’, 2023, *submitted to IEEE Transactions on Robotics (T-RO), (under revision)*
5. **WJ**, E. Dantec, E. Arlaud, N. Mansard, and J. Carpentier, ‘Parallel and Proximal Constrained Linear-Quadratic Methods for Real-Time Nonlinear MPC’, in Proceedings of Robotics: Science and Systems (RSS), Delft, Netherlands, Jul. 2024. doi: 10.15607/RSS.2024.XX.002.
6. <span class="text-blue">E. Ménager, A. Bilger, **WJ**, J. Carpentier, and C. Duriez, ‘Condensed semi-implicit dynamics for trajectory optimization in soft robotics’, in IEEE International Conference on Soft Robotics (RoboSoft), San Diego (CA), United States: IEEE, Apr. 2024.</span>
7. <span class="text-blue">E. Dantec, **WJ**, and J. Carpentier, ‘From centroidal to whole-body models for legged locomotion: a comparative analysis’, presented at the 2024 IEEE-RAS International Conference on Humanoid Robots, Nancy, France: IEEE, Jul. 2024.</span>

### Side contributions

8. Q. Le Lidec, **WJ**, I. Laptev, C. Schmid, and J. Carpentier, ‘Enforcing the consensus between Trajectory Optimization and Policy Learning for precise robot control’, in 2023 IEEE International Conference on Robotics and Automation (ICRA), May 2023, pp. 946–952.
9. Q. Le Lidec, **WJ**, L. Montaut, I. Laptev, C. Schmid, and J. Carpentier, ‘Contact Models in Robotics: a Comparative Analysis’, IEEE Transactions on Robotics, vol. 40, pp. 3716–3733, Jul. 2024, doi: 10.1109/TRO.2024.3434208.

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

## Introduction to ALM

### Penalty methods

Consider a mathematical program:

$$
\begin{aligned}
  \min_{z\in \mathbb{R}^n}~& f(z) \\
  \mathrm{s.t. }~& g(z) = 0 \\
                 & h(z) \leqslant 0
\end{aligned}
$$

where $f$ is the objective and $g: \mathbb{R}^n \to \mathbb{R}^m$, $h:\mathbb{R}^n \to \mathbb{R}^p$ are the constraints.

<v-click>

**Penalty-based methods:** solve a series of unconstrained minimisations:

1. Solve $\min_z f(z) + \frac{1}{\mu} (\|g(z)\|^2 + \|[h(z)]_+\|^2)$, get solution $z_\mu$
2. Decrease $\mu$
3. Go back to 1., "warm-start" from $z_\mu$.

**Problem:** decrease $\mu$ to satisfy $c=0 \Rightarrow$ **degrade conditioning** (harder to solve step 1.)!

</v-click>


---

### ALM and the prox-point iteration

**Instead...** Rewrite problem as a **saddle-point**:
$$
  \min_{z\in \mathbb{R}^n} \max_{\lambda \in \mathbb{R}^m} f(z) + \lambda^\top c(z).
$$

Use the proximal-point algorithm in the *dual* problem:

1. **"concavify" in $\lambda$** (proximal map):
   $$
     \lambda_\mu^+(\textcolor{red}{\lambda_e}) = \arg_\lambda \min_z \max_\lambda f(z) + \lambda^\top c(z)
     - \frac{\mu}{2} \| \lambda - \textcolor{red}{\lambda_e} \|_2^2
   $$
2. **iterate the proximal mapping $\lambda_\mu^+$ : $\boxed{\lambda^{k+1} = \lambda_\mu^+(\lambda^k)}$**
3. (optionally) decrease $\mu$ and **repeat** from Step 1.

---

Equivalent to the well-known AL method or *method of multipliers*:

1. minimise the **AL function $\mathcal{L}_\mu$**:
   $$
     z^\star_\mu(\lambda_e) \in
     \argmin_z \mathcal{L}_\mu(z, \lambda_e) := f(z) + \lambda^\top c(z) + \frac{1}{2\mu} \| c(z) \|_2^2.
   $$
2. set $\lambda_e \leftarrow \lambda_e + \tfrac{1}{\mu}c(z^\star_\mu(\lambda_e))$
3. (optionally) decrease $\mu$

---

### ALM: computing directions



$$
  \mathcal{L}_\mu(z, \lambda_e) := f(z) + \lambda^\top c(z) + \frac{1}{2\mu} \| c(z) \|_2^2.
$$

$$
  \begin{bmatrix}
    H & J^\top \\ J & -\mu I
  \end{bmatrix}
  \begin{bmatrix} \delta x \\ \delta \lambda \end{bmatrix}
  =
  -\begin{bmatrix}
    \nabla f + J^\top \lambda \\ c + \mu (\lambda_e - \lambda)
  \end{bmatrix}
$$

---

**Caveats.**

* Theory says: decrease $\mu$ to go faster.
* Decreasing $\mu$ makes $\mathcal{K}_\mu \to \mathcal{K}_0$...

---

## ProxDDP: an ALM algorithm for constrained trajectory optimisation

**Idea: do proximal ALM & dynamic programming.**

** TODO FINISH **


<div class="absolute bottom-0 text-0.6em">

**Reference papers**

1. **WJ**, A. Bambade, N. Mansard, and J. Carpentier, ‘Constrained Differential Dynamic Programming: A primal-dual augmented Lagrangian approach’, in 2022 IEEE/RSJ International Conference on Intelligent Robots and Systems
2. **WJ**, N. Mansard, and J. Carpentier, ‘Implicit Differential Dynamic Programming’, in 2022 International Conference on Robotics and Automation (ICRA), Philadelphia, United States
3. **WJ**, A. Bambade, E. Arlaud, S. El-Kazdadi, N. Mansard, and J. Carpentier, ‘PROXDDP: Proximal Constrained Trajectory Optimization’, 2023. _Submitted to Transactions on Robotics (under revision)_

</div>

---
layout: two-cols-title
columns: is-8
---

:: title ::

## A software contribution

:: left ::

<img src="/aligator-github-page.png" alt="ali-github" class="w-140">

:: right ::

**[aligator]{.font-mono} is a C++ OCP solver library for robotics and beyond.**

<div class="ns-c-tight">

* implements our ProxDDP algorithm (and reimplements FDDP as a baseline)
* using Pinocchio for robot modelling
* comes with Python bindings
* suitable for MPC!
* under **active** development: 27k lines of code so far.

</div>

Open-source on GitHub! https://github.com/Simple-Robotics/aligator

---
layout: top-title
---

:: title ::

## Benchmarks

:: content ::

*New in revision of T-RO submission.*

* Compare against other nonlinear solvers: OCP solver ALTRO, and generic solver IPOPT
* Implemented wrappers around ALTRO and IPOPT to solve problems from `aligator`
* Set of benchmarks problems[^bench]

<div v-click>

**Methodology.** Assess multiple configurations of the solvers:

* **ALTRO**: tune subproblem tolerance
* **IPOPT**: Gauss-Newton Hessian approx. vs. LBFGS approx.
* **ProxDDP:** test configurations with: different initial AL penalty $\mu_0 > 0$, linear vs. nonlinear rollout, linesearch parameters...
* *more details in the revised T-RO paper.*

</div>

[^bench]: GitHub: https://github.com/Simple-Robotics/aligator-bench

<style>
  .footnote-item {
    font-size: 11px;
  }
  .footnotes {
    position: absolute;
    bottom: 0;
    margin-right: 4em;
  }
</style>

<!--
Only present 2 benchmark problems
* More info in the revised submission paper (preprint online soon)
-->

---

### SOLO-12 "Yoga" task

A very nonlinear task for a whole-body model, 4 contact phases.

<video controls loop autoplay class="h-10/12 place-self-center">
  <source src="/solo12_lift_paw.mp4" type="video/mp4">
</video>

---

### SOLO-12 "Yoga" task: performance profile

<img src="/bench/solo_yoga_perfprofile_time.svg" alt="solo_times" class="place-self-center" />

<p class="absolute bottom-0 font-italic">
  
  **Performance profile:** no. of problems solved as function of how slower you are w.r.t. the fastest solver on a given problem.
</p>

<!--
ALTRO is not present there because... no instances converged at all.
-->

---

### SOLO-12 "Yoga" task: solve times vs problems solved

<img src="/bench/solo_yoga_solve_times.svg" alt="solo_times" class="place-self-center" />

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

---

### UR10 "ballistic" task: performance profile

<img src="/bench/ur10_ballistic_perfprofile_time.svg" alt="ur10_ballistic" class="place-self-center">

---

### Benchmarks: takeaway message

ProxDDP overall performs well on the benchmark suite.

* Competitive with IPOPT on the test set (in solve time)
* More robust across problems than ALTRO

Benchmarks (including wrappers for ALTRO & IPOPT) available online:  
https://github.com/Simple-Robotics/aligator-bench

---
layout: section
---

# Going parallel for proximal DDP

<hr>

<Toc columns=1 minDepth="2" maxDepth="2" mode="onlyCurrentTree" />

---

## Recap

**So far:**

<v-clicks>

* established a DDP recursion-like algo for solving constrained OCPs
* have to **solve a larger linear system**
* provided an open-source **fast C++ implementations**
* validated on a test bench against other solvers

</v-clicks>

---

## The limitations of Riccati: multicore architectures

DDP-type methods (or any method based on **Riccati**), have a *fatal* flaw:

* inherently **linear in time** $\mathcal{O}(N)$
* no way of exploiting **multicore architectures**! (except evaluating e.g. gradients in parallel)

<v-click>
<span class="text-blue-700 ns-c-tight">

**Questions**

* How to parallelise the Riccati recursion?
* What kind of speedup can we expect?
* What are the tradeoffs?

<figure class="ml-20">
  <img src="/riccati-serial.drawio.svg" alt="serial_riccati" class="w-full"/>
</figure>
</span>
</v-click>

<div class="absolute bottom-0 text-0.7em">
  
  **Reference papers for this section:**

  1. **WJ**, E. Dantec, E. Arlaud, N. Mansard, and J. Carpentier, ‘Parallel and Proximal Constrained Linear-Quadratic Methods for Real-Time Nonlinear MPC’, in *Proceedings of Robotics: Science and Systems*, Delft, Netherlands, Jul. 2024
  2. E. Dantec, **WJ**, and J. Carpentier, ‘From centroidal to whole-body models for legged locomotion: a comparative analysis’, presented at the *2024 IEEE-RAS International Conference on Humanoid Robots*, Nancy, France: IEEE, Jul. 2024

</div>

---

## Our approach

* Focus on the structure-exploiting **linear solver**, not the nonlinear problem
* The main idea: **"Parametrise to parallelise"**

---

### Parametrise

Consider linear-quadratic problem
$$
\begin{aligned}
  \mathcal{E}_0(x_0, \theta) =
  \min_{\bm{x},\bm{u}}~&
  \sum_{t=0}^{N-1} \ell_t(x_t, u_t) + \ell_N(x_N; \theta)  \\
  \mathrm{s.t.}~& A_tx_t + B_tu_t + E_tx_{t+1} + f_t = 0 \\
                & C_tx_t + D_tu_t + d_t = 0
\end{aligned}
$$
where the $\ell_t$ are all quadratic, and **the terminal cost $\ell_N$ is parametric**:
$$
  \ell_N(x, \theta) = \frac{1}{2}
  \begin{bmatrix}x \\ \theta \end{bmatrix}^\top
  \begin{bmatrix}
    Q_N & \Phi_N \\
    \Phi_N^\top & \Gamma_N
  \end{bmatrix}
  \begin{bmatrix}x \\ \theta \end{bmatrix}
  + q_N^\top x + \gamma_N^\top \theta.
$$

<figure class="w-full ml-10">
  <img src="/riccati-parametric.drawio.svg" alt="parametric_riccati" />
</figure>

**Property.** $\mathcal{E}_0$ is quadratic in $(x_0, \theta)$.

---

### Parallelise

**Idea: make the co-state $\lambda$ at each "cut" point into parameters.**

<figure class="w-full ml-10">
  <img src="/riccati-parallel.drawio.svg" alt="parallel_riccati" />
</figure>

---

**The general case:**
<figure class="w-190 ml-30">
  <img src="/riccati-parallel-gen.drawio.svg" alt="parallel_riccati" />
</figure>

<div v-click class="grid grid-cols-2">
<div>

**The consensus problem:** block-sparse problem with *block-tridiagonal pattern:*
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

## Benchmarks

<div class="grid grid-cols-2">

<figure class="grid-col-span-1">
  <img src="/mac-gar-bench-timings.svg" alt="parallel_riccati"
  class="h-120" />
</figure>

<div class="grid-col-span-1 mt-10">

  **Synthetic benchmark.** Timings on an M1 Mac Studio Ultra desktop computer.
  
  * **Problem dimensions:** $n_x=36$, $n_u=12$ (same as SOLO-12 wole-body).
  * Time horizon varies

</div>
</div>

---
layout: two-cols-title
columns: is-4
---

:: title ::
## Deploying multi-phase constrained MPC

:: left ::

**Whole-body jumping on Unitree GO-2:**

* Fixed flight/contact phases
* Constraints: joint position & torque limits, landing $z(t_\text{contact}) = 0$...
* Receding-horizon control, **whole problem rotates (including constraints)**
* Real-time on modern CPU, using the **parallel Riccati recursion (~5ms per iteration)**

:: right ::

<video controls loop autoplay class="w-full place-self-center mb-2">
  <source src="/quadru_jump_edit.mp4" type="video/mp4">
</video>

<div class="grid grid-cols-2 gap-6 w-full">
<p class="text-right">

  **Ewen Dantec**
</p>
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

## Take-home messages

* An appropriate variant of ALM can work quite well for solving OCPs
  * Our solver is seeing use for MPC and solving offline OCPs
* **Implementations (and heuristics) matter**
* Riccati can be revamped: **our parallel solver gives great improvements** on large systems!

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

</v-clicks>

---

### Long term

<v-clicks>

* [Automatic differentiation]{.text-orange .font-bold} for machine/deep learning applications e.g. **policy learning**
  * AL might have the right properties for differentiable optimisation
  * Already done for QPs, including infeasible QPs by Bambade et al.[^qplayer] - maybe nonconvex OCPs work?
  * Parallel LQ solver already introduced the right tools

* [Look at nonsmooth constraints:]{.text-orange .font-bold}
  * Time optimisation
  * Complementarity and vanishing constraints: contact & frictional contact

</v-clicks>

[^qplayer]:  A. Bambade, F. Schramm, A. Taylor, and J. Carpentier, ‘Leveraging augmented-Lagrangian techniques for differentiating over infeasible quadratic programs in machine learning’, presented at the The Twelfth International Conference on Learning Representations (ICLR 2024), Oct. 2023. Available: https://openreview.net/forum?id=YCPDFfmkFr

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

## Backup slides
