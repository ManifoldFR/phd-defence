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
color: light
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
color: light
---

### Table of contents

<Toc columns=1 minDepth="1" maxDepth="1" mode="all" />

---
layout: section
color: light
---

# Introduction


---
layout: default
color: light
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
layout: top-title
hideInToc: true
---

:: title ::

## Robot control: two ways

:: content ::

<div class="grid grid-cols-2">
  <div>

  ## Optimal control

  * Model-based mapping from sensors, to **state**, to **control**
  * For $u_t$: solve an **optimal control problem** (OCP).

  <img src="/optctrl.drawio.svg" alt="control law" class="w-full place-self-center" />

  </div>
  <div>

  ## Reinforcement learning

  <div v-click>

  * **Learn** the mapping from sensors to **control**
  * (Mostly) still use **models** in the simulator to learn (but not at "inference" time)

  <img src="/deeprl.drawio.svg" alt="control law" class="w-full place-self-center"></img>
  </div>

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
  &g_t(x_t, u_t) \leqslant 0 \\
  &g_N(x_N) \leqslant 0.
\end{aligned}
$$

<v-clicks>

* **Unknowns:** System states $\bm{x} = (x_0,\ldots,x_N)$ / Control inputs $\bm{u} = (u_0,...,u_{N-1})$
* $J(\bm{x}, \bm{u})$ the **cost function** (e.g. distance to target, magnitude of controls $u$...)
* $f_t = 0$ defines *discrete-time dynamics* $(x_t,u_t) \mapsto x_{t+1}$ (e.g. **robot dynamics**)
* $g_t\leqslant 0$ and $g_N\leqslant 0$ are the **path constraints** for the system (obstacles, velocity limits, ...).  
  If there are no $g_t, g_N$, we say the OCP is **unconstrained**.

</v-clicks>

---

## Why make tailored solvers?

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
hideInToc: true
---

## Why make tailored solvers? (II)

**Large-scale** and **structured** problem means:

<v-clicks>

* Need for **large-scale** nonlinear solvers. Those exist! But...
* A *generic* solver like IPOPT handles *generic* sparsity patterns
* A tailored solver:
  * faster by exploiting **specific structure**, perhaps become... **[real-time.]{.text-red}**

</v-clicks>

---

## Strategies for nonlinear constraints

The OCP is a *nonlinear problem* with *nonlinear constraints*.
**How to deal with this?**

<v-click>
Literature for strategies on this goes back years. Main families of methods are:

* (sequential) quadratic programming
* interior-point methods
* [augmented Lagrangian methods (ALM)]{.font-bold .text-orange}

All can be implemented with specific attention to OCPs.

</v-click>

<div v-click>
Our choice is ALM because:

* easy to understand
* simple to implement and pragmatic choice
* versatile (equality constraints, inequality constraints, more...)

</div>

---

## Problem statement

**Questions:**

* Can we use a variant of the **augmented Lagrangian method (ALM)** to design a new solver for constrained OCPs?
* Can we **keep it simple**, and solve multiple problems well?
* Can we make it run in **real-time** for model-predictive control (MPC) ?

---

## Contributions

* A new, primal-dual ALM algorithm for solving OCP 
* A parallel linear solver for OCP
* A performant library that can be used for **solving problems offline** *or* for **real-time control**

---

## Related work: solvers for OCPs

<div class="ns-c-tight">

* DDP[^mayneDDP] and variants such as FDDP[^0] - only for **unconstrained** OCPs
* Hybrid DDP (2012)[^2]: based on ALM, from authors in the aerospace community
* ALTRO[^3][^4]: based on ALM, for robotics, handles inequality & conic constraints
* FATROP[^1]: uses an interior-point method
* CSQP[^jordana]: structure-exploiting sequential quadratic programming (SQP)

</div>

[^mayneDDP]: D. Q. Mayne, ‘A Second-order Gradient Method for Determining Optimal Trajectories of Non-linear Discrete-time Systems’, International Journal of Control, vol. 3, no. 1, pp. 85–95, Jan. 1966, doi: 10.1080/00207176608921369.
[^0]: C. Mastalli et al., ‘Crocoddyl: An Efficient and Versatile Framework for Multi-Contact Optimal Control’, 2020 IEEE International Conference on Robotics and Automation (ICRA), pp. 2536–2542, May 2020, doi: 10.1109/ICRA40945.2020.9196673.
[^1]: L. Vanroye, A. Sathya, J. De Schutter, and W. Decré, ‘FATROP : A Fast Constrained Optimal Control Problem Solver for Robot Trajectory Optimization and Control’, in 2023 IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS), Detroit, MI, USA: IEEE, Oct. 2023. doi: 10.1109/IROS55552.2023.10342336.
[^2]: G. Lantoine and R. P. Russell, ‘A Hybrid Differential Dynamic Programming Algorithm for Constrained Optimal Control Problems. Part 1: Theory’, J Optim Theory Appl, vol. 154, no. 2, pp. 382–417, Aug. 2012, doi: 10.1007/s10957-012-0039-0.
[^3]: T. A. Howell, B. E. Jackson, and Z. Manchester, ‘ALTRO: A Fast Solver for Constrained Trajectory Optimization’, in 2019 IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS), Macau, China: IEEE, Nov. 2019, pp. 7674–7679. doi: 10.1109/IROS40897.2019.8967788.
[^4]: B. E. Jackson, T. Punnoose, D. Neamati, K. Tracy, R. Jitosho, and Z. Manchester, ‘ALTRO-C: A Fast Solver for Conic Model-Predictive Control’, in 2021 IEEE International Conference on Robotics and Automation (ICRA), May 2021, pp. 7357–7364. doi: 10.1109/ICRA48506.2021.9561438.
[^jordana]: A. Jordana, S. Kleff, A. Meduri, J. Carpentier, N. Mansard, and L. Righetti, ‘Stagewise Implementations of Sequential Quadratic Programming for Model-Predictive Control’, Dec. 2023.

<style>
  .footnote-item {
    font-size: 9px;
  }
  .footnote-item p {
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
---

# Proximal algorithms for Nonlinear Programming and Optimal Control
<hr>

<Toc columns=1 minDepth="2" maxDepth="2" mode="onlyCurrentTree" />

---

## Introduction to ALM

Consider a mathematical program:

$$
\begin{aligned}
  \min_z~& J(z) \\
  \mathrm{s.t. }~& c(z) = 0
\end{aligned}
$$

**Penalty-based methods** consist in dualizing it as a series of unconstrained minimizations:

1. Solve $\min_z J(z) + \mu c(z)$, get solution $z_\mu$
2. Increase $\mu$
3. Go back to 1., "warm-start" from $z_\mu$.


---

## ProxDDP: an ALM algorithm for constrained trajectory optimization


<div class="absolute bottom-0 text-0.6em">

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

**[aligator]{.font-mono} is a C++ OCP solver library for robotics and beyond.**

<div class="ns-c-tight">

* implements our ProxDDP algorithm (and reimplements FDDP as a baseline)
* using Pinocchio for robot modelling
* comes with Python bindings
* suitable for MPC!
* under **active** development: 27kloc so far.

</div>

Open-source on GitHub! https://github.com/Simple-Robotics/aligator

---
layout: top-title
---

:: title ::

## Benchmarks

:: content ::

*New in revision of T-RO submission.*

* Compare against other nonlinear solvers ALTRO (tailored for OCP) and IPOPT (generic NLP solver)
  * Implemented wrappers around ALTRO and IPOPT to solve problems from `aligator`
* Set of three benchmarks problems[^bench]

<div v-click>
Assess multiple configurations of the solvers:

* **ALTRO**/**IPOPT**: 2 configs each
* **ProxDDP:** test configurations with... 
  * different initial AL penalty $\mu_0 > 0$
  * linear vs. nonlinear rollout
  * linesearch parameters

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

---

### SOLO-12 "Yoga" task

A very nonlinear task for a whole-body model, 4 contact phases.

<video controls loop autoplay class="h-11/12 place-self-center">
  <source src="/solo12_lift_paw.mp4" type="video/mp4">
</video>

---

### SOLO-12 "Yoga" task: solve times vs problems solved

<img src="/bench/solo_yoga_solve_times.svg" alt="solo_times" class="place-self-center" />

---

### SOLO-12 "Yoga" task: performance profile

<img src="/bench/solo_yoga_perfprofile_time.svg" alt="solo_times" class="place-self-center" />

<p class="absolute bottom-0 font-italic">
  
  **Performance profile:** no. of problems solved as function of how slower you are w.r.t. the fastest solver on a given problem.
</p> 

---

### UR10 "ballistic" task

<div class="ns-c-tight">

* Underactuated system
* Hard constraint on projectile final's position

</div>

<video controls loop autoplay class="h-11/12 place-self-center">
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

---

### Benchmarks: on standardisation

Lack of a suite of standard benchmarks for robotics:

* nonlinear prog. has [CUTEr](https://en.wikipedia.org/wiki/CUTEr) <sup>1</sup> (Fortran) test set, Maros-Meszaros for QPs -- efforts at standardization <sup>2</sup>
  * solvers like IPOPT are validated on CUTEr
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

DDP-type methods (or any method) based on the **Riccati** algorithm, have a *fatal* flaw:

* inherently **linear in time** $\mathcal{O}(N)$
* no way of exploiting **multicore architectures** !


<div class="absolute bottom-0 text-0.7em">
  Reference papers for this section:

  1. **WJ**, E. Dantec, E. Arlaud, N. Mansard, and J. Carpentier, ‘Parallel and Proximal Constrained Linear-Quadratic Methods for Real-Time Nonlinear MPC’, in _Proceedings of Robotics: Science and Systems_, Delft, Netherlands, Jul. 2024
  2. E. Dantec, **WJ**, and J. Carpentier, ‘From centroidal to whole-body models for legged locomotion: a comparative analysis’, presented at the _2024 IEEE-RAS International Conference on Humanoid Robots_, Nancy, France: IEEE, Jul. 2024
</div>


---
layout: two-cols-title
columns: is-4
---

:: title ::
### Deploying multi-phase constrained MPC

:: left ::

**Whole-body jumping on Unitree GO-2:**

* Fixed flight/contact phases
* Constraints: joint position & torque limits, landing $z(t_\text{contact}) = 0$...
* Receding-horizon control, **whole problem rotates, including the constraints**
* Real-time on modern CPU, using the **parallel Riccati recursion (~5ms per iteration)**

:: right ::

<video controls loop autoplay class="w-full place-self-center">
  <source src="/quadru_jump_edit.mp4" type="video/mp4">
</video>

<div class="grid grid-cols-2 gap-6 w-fit center">
<img src="/trombi/ewen.jpg" alt="ewen" class="h-36"/>
<p>

  **Ewen Dantec**
</p>
</div>

---

# Randomized smoothing

....

---

# Conclusion and perspectives

---

## A roadmap

---
layout: cover
hideInToc: true
---

# Thank you!

---
hideInToc: true
---

## Backup slides
