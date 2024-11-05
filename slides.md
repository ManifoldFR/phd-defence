---
# You can also start simply with 'default'
theme: neversink
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
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
color: light
hideInToC: true
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
color: light
---

# Introduction

<Toc columns=1 minDepth="1" maxDepth="2" mode="onlyCurrentTree" />

---
layout: default
color: light
---

## From industrial robots... to mobile and fully autonomous robots

<figure class="w-full">
  <img src="/robot-timeline.drawio.png">
</figure>

Each stage required either more refined control schemes or perception methods.

* mobile vacuums: required advances in planning and mapping;
* legged robots: required advances in real-time **control**.

<div class="absolute bottom-0">

  Sources: KUKA: [Wikipedia](https://commons.wikimedia.org/wiki/File:KUKA_Industrial_Robots_IR.jpg); Dyson 360 robot: [dyson.com.au](https://edition.cnn.com/cnn-underscored/reviews/dyson-360-vis-nav-robot-vacuum)
</div>

---
layout: default
---

## Robot control: the problem

<div class="grid grid-cols-2 gap-4">
  <div>
    The basic problem goes like this:
    <img src="/control-loop.drawio.svg" alt="control law" class="h-80 place-self-center"></img>
  </div>
  <div class="mt-10">

  * Sensor data: **noisy**
  * Control law: **known transformation** from sensor data
  * Robot: **somewhat known**
  * World: **imperfect**, **high variability**, **physics** (contact, friction...)

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

  * Models map from sensors, to **state**, to **control**

  <img src="/optctrl.drawio.svg" alt="control law" class="w-full place-self-center"></img>

  <div v-click>

  Solve and extract $\textcolor{green}{u_0}$ from:
  $$
  \begin{aligned}
    \min_{\bm{x}, \textcolor{green}{\bm{u}} }%
    &\sum_{t=0}^{N-1} \ell_t(x_t, \textcolor{green}{u_t}) + \ell_N(x_N) \\
    \mathrm{s.t.}~%
    &f_t(x_t, \textcolor{green}{u_t}, x_{t+1}) = 0 \ \text{(dynamics)} \\
    &g_t(x_t, \textcolor{green}{u_t} ) \leq 0 \ \text{(path constraints)} \\
    &g_N(x_N) \leq 0 \ \text{(term. constraint)}
  \end{aligned}
  $$
  </div>

  </div>
  <div>

  ## Reinforcement learning

  * **Learn** the mapping from sensors to **control**
  * Still use **models** in the simulator to learn!

  <img src="/deeprl.drawio.svg" alt="control law" class="w-full place-self-center"></img>

  <div v-click>

  $$
    \min_{\textcolor{red}{\theta}\in\Theta} \mathbb{E}_{u_t\sim \pi_{\textcolor{red}{\theta}}(x_t)}\left[
        \sum_{t=0}^{N-1} \ell_t(x_t, u_t) + \ell_N(x_N) \right]
  $$
  </div>

  </div>
</div>

---
layout: top-title
hideInToc: true
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

---

## What is the optimal control problem (OCP) ?

It is a **mathematical optimization model** for driving a system according to a **performance objective**:
$$
\begin{aligned}
  \underset{\bm{x}, \bm{u}}{\operatorname{\mathrm{minimize}}}~%
  &J(\bm{x}, \bm{u}) = \sum_{t=0}^{N-1} \ell_t(x_t, u_t) + \ell_N(x_N) \\
  \mathrm{s.t.}~%
  &f_t(x_t, u_t, x_{t+1}) = 0 \\
  &g_t(x_t, u_t) \leq 0 \\
  &g_N(x_N) \leq 0.
\end{aligned}
$$

<div v-click>

* **Unknowns:** System states $\bm{x} = (x_0,\ldots,x_N)$ / Control inputs $\bm{u} = (u_0,...,u_{N-1})$
* $J(\bm{x}, \bm{u})$ the **cost function** (e.g. distance to target, magnitude of controls $u$...)
* $g_t\leq 0$ and $g_N\leq 0$ are the **path constraints** for the system (obstacles, velocity limits, ...)
* $f_t(x_t, u_t, x_{t+1}) = 0$ defines _discrete dynamics_ $(x_t,u_t) \mapsto x_{t+1}$ (e.g. **robot dynamics**)

</div>

---

It is a **nonlinear program** with _lots_ of variables but with a **specific structure**.



---

## Why make tailored solvers?

* Need for **large-scale** nonlinear solvers
* A generic solver like IPOPT:
  * has a lot of heuristics to perform well on generic problems
  * handles generic sparsity patterns
* A tailored solver:
  * **less heuristics** (maybe)
  * faster by exploiting **specific structure**, perhaps **real-time**

---

## Proposals

**Question:** Can we a variant of the **augmented Lagrangian method (ALM)** to design a new solver for constrained MPC?

* ALM is simple to understand and implement, a pragmatic choice
* not a new idea for numerical OC

**Contribution** A new ALM-based 



---

## Publications

This thesis has led to the following publications:

<div>

### Main contributions

1. **WJ**, N. Mansard, and J. Carpentier, ‘Implicit Differential Dynamic Programming’, in 2022 International Conference on Robotics and Automation (ICRA), Philadelphia, United States: IEEE, May 2022. doi: 10.1109/ICRA46639.2022.9811647.
2. **WJ**, A. Bambade, N. Mansard, and J. Carpentier, ‘ProxNLP: a primal-dual augmented Lagrangian solver for nonlinear programming in Robotics and beyond’, in 6th Workshop on Legged Robots, Philadelphia, Pennsylvania, United States, May 2022. Accessed: Oct. 10, 2022.
3. **WJ**, A. Bambade, N. Mansard, and J. Carpentier, ‘Constrained Differential Dynamic Programming: A primal-dual augmented Lagrangian approach’, in 2022 IEEE/RSJ International Conference on Intelligent Robots and Systems, Kyoto, Japan, Oct. 2022.
4. **WJ**, A. Bambade, E. Arlaud, S. El-Kazdadi, N. Mansard, and J. Carpentier, ‘PROXDDP: Proximal Constrained Trajectory Optimization’, 2023.
5. **WJ**, E. Dantec, E. Arlaud, N. Mansard, and J. Carpentier, ‘Parallel and Proximal Constrained Linear-Quadratic Methods for Real-Time Nonlinear MPC’, in Proceedings of Robotics: Science and Systems, Delft, Netherlands, Jul. 2024. doi: 10.15607/RSS.2024.XX.002.

### Related publications

6. E. Ménager, A. Bilger, **WJ**, J. Carpentier, and C. Duriez, ‘Condensed semi-implicit dynamics for trajectory optimization in soft robotics’, in IEEE International Conference on Soft Robotics (RoboSoft), San Diego (CA), United States: IEEE, Apr. 2024.
7. E. Dantec, **WJ**, and J. Carpentier, ‘From centroidal to whole-body models for legged locomotion: a comparative analysis’, presented at the 2024 IEEE-RAS International Conference on Humanoid Robots, Nancy, France: IEEE, Jul. 2024.

### Other publications

8. . Q. Le Lidec, **WJ**, I. Laptev, C. Schmid, and J. Carpentier, ‘Enforcing the consensus between Trajectory Optimization and Policy Learning for precise robot control’, in 2023 IEEE International Conference on Robotics and Automation (ICRA), May 2023, pp. 946–952.
9.  Q. Le Lidec, **WJ**, L. Montaut, I. Laptev, C. Schmid, and J. Carpentier, ‘Contact Models in Robotics: a Comparative Analysis’, IEEE Transactions on Robotics, vol. 40, pp. 3716–3733, Jul. 2024, doi: 10.1109/TRO.2024.3434208.

</div>

<style>
h3 {
  font-weight: bold;
}
</style>

---
layout: section
---

# Augmented Lagrangians

<Toc columns=1 minDepth="1" maxDepth="2" mode="onlyCurrentTree" />

---

## Introduction to ALM

---

## ProxDDP: an ALM algorithm for constrained trajectory optimization


<div class="absolute bottom-0">
Reference papers:

1. **WJ**, A. Bambade, N. Mansard, and J. Carpentier, ‘Constrained Differential Dynamic Programming: A primal-dual augmented Lagrangian approach’, in 2022 IEEE/RSJ International Conference on Intelligent Robots and Systems
2. **WJ**, N. Mansard, and J. Carpentier, ‘Implicit Differential Dynamic Programming’, in 2022 International Conference on Robotics and Automation (ICRA), Philadelphia, United States
3. **WJ**, A. Bambade, E. Arlaud, S. El-Kazdadi, N. Mansard, and J. Carpentier, ‘PROXDDP: Proximal Constrained Trajectory Optimization’, 2023. _Submitted to Transactions on Robotics (under revision)_

</div>

---
layout: two-cols-title
columns: is-8
---

:: title ::

## A software [contribution]{.text-red}

:: left ::

<img src="/aligator-github-page.png" class="w-140">

:: right ::

**[aligator]{.font-mono} is a C++17 OCP solver library for robotics and beyond.**

* implements our ProxDDP algorithm
* provides a modelling interface (using Pinocchio for robot dynamics)
* comes with Python bindings
* suitable for MPC!

---
layout: top-title
---

:: title ::

## Benchmarks

:: content ::

* Compare against other nonlinear solvers ALTRO (tailored for OCP) and IPOPT (generic NLP solver)
  * Implemented wrappers around ALTRO and IPOPT to solve problems from `aligator`
* Set of three benchmark problems

<div v-click>
Assess multiple configurations of the solvers:

* **ALTRO**/**IPOPT**: 2 configs each
* **ProxDDP:** test configurations with... 
  * ...different initial AL penalty $\mu_0 > 0$
  * ...linear vs. nonlinear rollout
  * ...nonmonotone linesearch parameters

</div>
---

### SOLO-12 "Yoga" task

A very nonlinear task for a whole-body model, 4 contact phases.

<video controls loop autoplay class="h-4/5">
  <source src="/solo12_lift_paw.mp4" type="video/mp4">
</video>

---

### SOLO-12 "Yoga": performance profile

<div>

![solo_times](/bench/solo_yoga_perfprofile_time.svg)

</div>

---

### UR10 "ballistic" task

* Underactuated system
* Hard constraint on projectile final's position

<video controls loop autoplay class="h-90">
  <source src="/ur10_mug_throw.mp4" type="video/mp4">
</video>

---

### UR10 "ballistic" task: performance profile

<div>

  ![ur10_ballistic_times](/bench/ur10_ballistic_perfprofile_time.svg)

</div>

---
layout: top-title
---

::title::

### Benchmarks: takeaway message

::content::

### Lack of a suite of standard benchmarks for robotics:

* nonlinear prog. has [CUTEr](https://en.wikipedia.org/wiki/CUTEr) <sup>1</sup> (Fortran) test set, Maros-Meszaros for QPs -- efforts at standardization <sup>2</sup>
  * IPOPT was validated on ~954 problems from CUTEr
* In numerical OC, **everyone reimplements and tests their own problems, with their own solver's API.**
* Create a standard form for complex OCPs?
  * Symbolic language like AMPL? Julia's JuMP? CasADi? 
  * Pinocchio = right modelling tool for rigid-body quantities

<div class="absolute bottom-0">
  <p>[1] CUTEr: https://en.wikipedia.org/wiki/CUTEr</p>
  <p>[2] qpenchmark by S. Caron: https://github.com/qpsolvers/qpbenchmark</p>
</div>

---

# Going parallel for proximal DDP



<div class="absolute bottom-0">

  1. **WJ**, E. Dantec, E. Arlaud, N. Mansard, and J. Carpentier, ‘Parallel and Proximal Constrained Linear-Quadratic Methods for Real-Time Nonlinear MPC’, in _Proceedings of Robotics: Science and Systems_, Delft, Netherlands, Jul. 2024
  2. E. Dantec, **WJ**, and J. Carpentier, ‘From centroidal to whole-body models for legged locomotion: a comparative analysis’, presented at the _2024 IEEE-RAS International Conference on Humanoid Robots_, Nancy, France: IEEE, Jul. 2024
</div>


---
layout: two-cols-title
columns: is-4
---

:: title ::
### Deploying constrained MPC

:: left ::

#### Whole-body jumping on Unitree GO-2

- Constraints: joint position & torque limits, landing $z(t_\text{contact}) = 0$
  - usually all thrown in cost function !
- Real-time on consumer-grade CPU, using the parallel Riccati recursion (~5ms per iteration)

:: right ::

<video controls loop autoplay class="w-full">
  <source src="/quadru_jump_edit.mp4" type="video/mp4">
</video>

---

# Conclusion


