# PDHLC

## Posterior-Dispersion-Aware Hierarchical Latent Control for Humanoid Whole-Body Tracking

**Whole-body motion tracking on Unitree G1 through hierarchical motion context and separately regulated residual corrections.**

[Paper](docs/paper.pdf) · [Motivation](#motivation) · [Method](#method-architecture) · [Results](#experimental-results) · [Training](#training-procedure) · [Deployment](#deployment-interface)

PDHLC separates **motion-conditioned correction generation** from **reference- and feedback-conditioned amplitude regulation**. A hierarchical motion prior provides global motion context, temporally aligned local details, and a posterior-dispersion signal. A residual policy generates a candidate joint-space correction, while a separate scalar gate regulates its strength using reference dispersion, estimated physical feedback, and inertial cues.

### Highlights

| Setting | Reported result |
| :--- | :--- |
| Robot and control interface | Unitree G1; 29 actuated joints; 50 Hz joint-position control |
| H2 controller comparison | **92.00 ± 0.17%** full-reference completion under a 10 s deadline |
| Matched cue-routing comparison | **91.68%** completion for PDHLC vs. **89.10%** for direct cue input |
| Physical robot evaluation | **86/100** completed trials across five motion categories |
| Squat/stand hardware trials | **19/20** completed trials |

These results come from different evaluation protocols. Their definitions, denominators, and source tables are given in [Experimental Results](#experimental-results).

## Motivation

![Motivation for separating the joint-space correction pattern from its amplitude](images/motivation.png)

*Figure 1 from the paper. Motion context helps determine the candidate correction pattern; reference and execution cues regulate its magnitude.*

Whole-body tracking must preserve the intended motion while responding to contact changes, actuation errors, model mismatch, and disturbances. Similar tracking deviations can require different joint corrections during walking and squatting. Even for the same candidate correction, the appropriate magnitude can vary with execution conditions.

PDHLC addresses these two requirements through separate pathways:

- **Correction generation:** the current reference, recent proprioceptive history, and hierarchical motion latents determine a candidate joint-space correction.
- **Amplitude regulation:** reference-posterior dispersion, estimated physical feedback, and inertial inputs determine a shared scalar gate.

For a fixed candidate during deterministic execution, this gate attenuates its magnitude while preserving its signs and relative contributions across joints. The nominal tracking command bypasses the gate.

## Method Architecture

![PDHLC architecture with a hierarchical motion prior, nominal and residual policies, a physical-state estimator, and an amplitude gate](images/method_architecture.png)

*Figure 2 from the paper. Global and local reference latents condition tracking and residual generation. The additional reference and feedback cues enter the amplitude gate.*

### 1. Hierarchical motion prior

A reference window is decomposed into a low-frequency component and a temporally aligned local residual:

- A **global Gaussian latent** represents coarse motion progression.
- **Local vector-quantized codes** represent short-window residual details.
- A shared decoder learns motion and contact reconstruction, with additional penalties for reference-contact slip, contact-prediction error, and projected center-of-mass support violations.

The global encoder also provides the mean posterior variance:

$$
u_t = \frac{1}{d_G}\sum_{j=1}^{d_G}\sigma_{G,j}^{2}.
$$

This posterior dispersion is a reference-derived regulation cue. During deterministic evaluation, the global latent is replaced by its posterior mean; reference descriptors and dispersion values are retrieved from a fixed cache.

### 2. Motion-conditioned residual generation

Both the nominal tracker and residual policy receive the current reference, a five-frame proprioceptive history, and global/local motion latents. The residual policy produces a bounded candidate:

$$
\Delta a_t^{\mathrm{cand}}
= \Delta a_{\max}\odot\tanh\!\left(\pi_r(s'_t,z_G,z_{L,t})\right).
$$

Here, $s'_t$ combines the current reference with recent robot observations. The residual policy therefore retains execution feedback through its proprioceptive history.

### 3. Feedback-conditioned amplitude regulation

A physical-state estimator uses observation history and inertial inputs to estimate center-of-mass velocity, support position/ZMP, execution contacts, and normalized contact loads. These estimates are combined with posterior dispersion and an IMU descriptor:
$$
\alpha_t = \operatorname{sigmoid}\!\left(
g_\eta(u_t,\hat y_t^{\mathrm{phys}},p_t^{\mathrm{IMU}})
\right),
\qquad 0 < \alpha_t < 1.
$$


The deterministic joint-position command is
$$
a_t^{\mathrm{final}}
= a_t^{\mathrm{base}} + \alpha_t\Delta a_t^{\mathrm{cand}}.
$$

The gate modulates only the residual. The per-joint residual limit is $\Delta a_{\max,j}=0.10\,\mathrm{rad}$, giving a deterministic bound of $|\Delta a_{t,j}|\leq 0.10\alpha_t\,\mathrm{rad}$.

**Training distinction.** In Stage IV, the gate modifies the pre-squash Gaussian mean, while the exploration standard deviation is parameterized separately. Stochastic training retains a residual support bounded by ±0.10 rad; the tighter gate-scaled bound above describes zero-noise execution.

*Source: paper Sections III-B–III-D, Eqs. (15), (24)–(26), (31), and (35).*

## Experimental Results

All values below are reported in the [accompanying paper](docs/paper.pdf). Completion means finishing the full prescribed reference under the stated protocol.

### Simulation: controller comparison

The main comparison on the **held-out H2 motion split** uses **five seeds and 1,000 rollouts per seed**. Values are mean ± standard deviation across seeds. References have mixed durations and a **10 s full-reference deadline**; references longer than 10 s remain in the denominator as failures.

| Controller | Completion ↑ (%) | Position error $E_{\mathrm{pos}}$ ↓ (mm) | Velocity error $E_{\mathrm{vel}}$ ↓ (m/s) | Stance-foot slip ↓ (m/s) |
| :--- | ---: | ---: | ---: | ---: |
| VMS — external baseline | 88.20 ± 0.26 | 65.74 ± 1.62 | 0.231 ± 0.108 | 0.067 ± 0.004 |
| GMT — external baseline | 85.80 ± 0.34 | 69.32 ± 1.85 | 0.252 ± 0.009 | 0.076 ± 0.053 |
| Frozen nominal tracker | 80.80 ± 0.42 | 83.12 ± 2.14 | 0.307 ± 0.012 | 0.097 ± 0.007 |
| Ungated residual | 84.60 ± 0.36 | 72.41 ± 1.76 | 0.278 ± 0.034 | 0.084 ± 0.023 |
| Physics-only gate | 89.42 ± 0.31 | 62.71 ± 1.51 | 0.229 ± 0.023 | 0.058 ± 0.009 |
| Reference-only gate | 88.62 ± 0.22 | 64.17 ± 1.43 | 0.235 ± 0.010 | 0.064 ± 0.013 |
| **PDHLC — combined gate** | **92.00 ± 0.17** | **57.88 ± 1.20** | **0.205 ± 0.009** | **0.043 ± 0.002** |

PDHLC improves completion by **2.58 percentage points** over physics-only gating and **7.40 percentage points** over ungated residual control. Tracking errors are measured over each controller's observed frames up to completion or termination.

*Source: Table II(A), p. 8. VMS and GMT are contextual external baselines; the remaining rows are internal controller variants.*

![Simulation examples for Kungfu, squatting and standing, hip swing, and 45-degree leftward walking](images/simulation_examples.png)

*Figure 3 from the paper. Representative simulated motion-tracking sequences.*

### Matched comparison: scalar gating vs. direct cue input

This experiment matches upstream features, action bounds, reward, model capacity, and Stage-IV training budgets. Each of the **1,158 H2 references** is evaluated once per controller for each of **five matched training-seed pairs**.

| Evaluation subset | Direct-input residual | PDHLC |
| :--- | ---: | ---: |
| All references | 5,159 / 5,790 (**89.10%**) | 5,308 / 5,790 (**91.68%**) |
| Deadline-eligible references, duration ≤ 10 s | 5,159 / 5,540 (**93.12%**) | 5,308 / 5,540 (**95.81%**) |

The additional cues can change both correction pattern and magnitude when supplied directly to the residual policy. PDHLC routes those cues to a separate scalar gate. This comparison uses a separate evaluation block from the controller comparison above.

*Source: Table II(B), pp. 8–9.*

### Perturbation robustness in simulation

The paper evaluates command delays, dynamics scaling, torso pushes, friction changes, and torque limits using **five seeds × 200 rollouts per seed**. Selected conditions are shown below.

| Perturbation | PDHLC completion (%) | Ungated completion (%) | Difference (percentage points) |
| :--- | ---: | ---: | ---: |
| 80 ms command delay | 70.6 | 58.7 | +11.9 |
| Dynamics scaling: mass 1.15, inertia 1.2, Kd 1.0 | 83.6 | 70.4 | +13.2 |
| 15 N torso push | 65.2 | 53.4 | +11.8 |
| Friction coefficient 0.2 | 68.4 | 55.6 | +12.8 |
| Torque limit scaled to 0.3 | 63.8 | 50.9 | +12.9 |

The push acts along world +x from 0.50 to 0.70 s. These are simulation perturbation results; physical disturbance-recovery testing is identified as future work in the paper.

*Source: Table V and Section IV-D, pp. 9–10.*

### Real-world Unitree G1 experiments

![Unitree G1 hardware execution sequences from public-dataset and self-collected motion references](images/hardware_examples.png)

*Figure 7 from the paper. Public-dataset references are shown in the upper group and self-collected references in the lower group, with frames in temporal order.*

**Set C — motion categories.** The physical G1 completes **86 of 100 trials**, with 20 trials per category.

| Motion category | Completed / trials | Completion (%) | Mean computation latency (ms) | p95 computation latency (ms) |
| :--- | ---: | ---: | ---: | ---: |
| Directional walking | 18 / 20 | 90.0 | 13.3 | 15.5 |
| Squat/stand | 19 / 20 | 95.0 | 13.8 | 16.8 |
| Hip-swing adjustment | 18 / 20 | 90.0 | 13.7 | 16.3 |
| Kungfu-style motion | 16 / 20 | 80.0 | 14.6 | 17.0 |
| Uneven-terrain walking | 15 / 20 | 75.0 | 14.0 | 16.9 |
| **All categories** | **86 / 100** | **86.0** | — | — |

**Set A — shared-clip controller comparison.** Four controllers are evaluated on ten shared clips, with 40 trials per controller.

| Controller | Completed / trials | Completion (%) | Trial-balanced body-point error ↓ (mm) |
| :--- | ---: | ---: | ---: |
| Nominal tracker | 29 / 40 | 72.5 | 78.9 |
| Ungated residual | 33 / 40 | 82.5 | 70.4 |
| Physics-only gate | 35 / 40 | 87.5 | 64.9 |
| **PDHLC** | **36 / 40** | **90.0** | **60.8** |

Body-point error uses 15 points after root-translation and yaw alignment. A separate repeated-G1 evaluation reports **45/50** completed trials for PDHLC versus **43/50** for physics-only gating, with trial-balanced body-point errors of **62.3 mm** and **64.1 mm**, respectively.

Latency is measured from input-packet availability to command submission, excluding sensing, communication, and actuator response. Set C p95 values are **15.5–17.0 ms**; the separate repeated-G1 evaluation reports **17.4 ms**, within the **20 ms** control period at that percentile under the tested configuration.

*Source: Table VI and Section IV-E, pp. 10–11. The category, shared-clip, and repeated-G1 results are separate trial sets.*

## Reference Data

Reference motions come from **AMASS**, **LAFAN1**, and additional sources. The preparation pipeline retargets motions to the 29-DoF G1 using GMT, resamples them at 50 Hz, annotates contacts, and filters them using a pretrained tracker.

| Split | Purpose | Sequences |
| :--- | :--- | ---: |
| Training | Prior and controller learning | 9,272 |
| H1 | Model selection | 1,158 |
| H2 | Held-out evaluation | 1,158 |
| **Total** | **46.2 hours of prepared motion** | **11,588** |

The controller accesses prescribed reference frames ahead of the current time. This lookahead does not require future robot measurements.

*Source: Section III-A, p. 3.*

## Training Procedure

| Stage | Component | Learning objective | Reported budget |
| :--- | :--- | :--- | :--- |
| I | Hierarchical motion prior | Motion/contact reconstruction, KL regularization, VQ objectives, and contact-related losses | 20 million samples; 20,000 updates |
| II | Nominal tracking policy | PPO with the tracking reward | 492 million environment steps |
| III | Physical-state estimator | Supervised CoM, support/ZMP, contact, and load estimation | 10 million samples; 10,000 updates |
| IV | Residual policy, scalar gate, and exploration parameters | Joint PPO optimization | 492 million environment steps; 5,000 updates |

Stage IV freezes the motion prior, nominal tracker, and physical-state estimator. Stages II and IV use the same tracking reward. Models are selected on H1 and evaluated on H2.

### Reported implementation settings

| Item | Setting |
| :--- | :--- |
| Simulation environments | 4,096 IsaacGym environments |
| Global latent / local latent | 64 / 32 dimensions |
| VQ codebook | 512 entries |
| Reference preview | 60 intervals; 61 frames; 1.2 s at 50 Hz |
| Local encoding window | 17 frames; half-window of 8 |
| Coarse motion operator | 6 Hz low-pass cutoff; stride 4; linear interpolation |
| Proprioceptive history | 5 observations, each 93-dimensional |
| Prior, estimator, residual hidden layers | [512, 256, 128] |
| Gate hidden layers | [128, 64] |
| Nominal actor / critic hidden layers | [512, 256, 128] each; ELU |
| Optimizer | Adam; learning rate 3 × 10⁻⁴ |
| Batch / epochs | 8,192 / 5 |
| Stage-IV PPO | Clip 0.20; discount 0.99; GAE parameter 0.95; rollout length 24 |
| Domain randomization | Mass ±15%; friction 0.6–1.2; motor scale 0.9–1.1; delay 0–2 steps |

*Source: Table I, p. 6, and Section IV-A, p. 7.*

## Deployment Interface

The controller sends **29 joint-position commands at 50 Hz**. The current-step deployment packet contains **218 values**, with temporal context, reference-cache entries, and inferred physical feedback supplied through auxiliary paths.

| Current-step packet block | Width |
| :--- | ---: |
| Reference joint positions and velocities | 58 |
| Reference torso orientation descriptor | 6 |
| IMU angular velocity | 3 |
| Joint position offsets from the default configuration | 29 |
| Joint velocities | 29 |
| Previous normalized action | 29 |
| Lower and upper soft-limit margins | 58 |
| Inertial descriptor | 3 |
| IMU acceleration | 3 |
| **Total** | **218** |

Additional inputs include the **5 × 93 proprioceptive history**, cached global/local latents and posterior dispersion, and estimated physical feedback. The 218-dimensional packet is therefore one part of the complete controller input.

The reported normalized residual scale is **0.40 policy units**, mapped through a **0.25 rad** actuator scale to **0.10 rad**. The paper reports low-level PD gains of **Kp/Kd = 60/2.5**.

*Source: Sections III-A and III-C; Table I, pp. 3–6.*

## Scope and Interpretation

- The posterior-dispersion signal is the reference encoder's mean posterior variance. The paper evaluates its usefulness for residual regulation; it is not presented as a calibrated failure probability.
- Pattern preservation applies to a fixed candidate correction under deterministic scalar gating. The candidate itself changes with the reference and robot state.
- Contact-related losses regularize the learned motion representation. Residual bounds and empirical tracking performance do not constitute a general closed-loop stability guarantee.
- The main simulation completion metric includes a fixed deadline. It should be interpreted together with reference duration and the deadline-eligible results.
- Hardware computation latency excludes sensing, communication, and actuator response. Hardware push-recovery performance is not established by the simulation perturbation tests.

## Project Resources

| Resource | Location |
| :--- | :--- |
| Manuscript | [docs/paper.pdf](docs/paper.pdf) |
| Motivation figure | [images/motivation.png](images/motivation.png) |
| Method architecture | [images/method_architecture.png](images/method_architecture.png) |
| Simulation examples | [images/simulation_examples.png](images/simulation_examples.png) |
| Hardware examples | [images/hardware_examples.png](images/hardware_examples.png) |
| Project repository named in the manuscript | [ZX123445/PDHLC](https://github.com/ZX123445/PDHLC) |

This README summarizes the accompanying manuscript. Installation commands, executable training/evaluation entry points, checkpoint links, and license information are not specified in that manuscript.







##Real Robot


https://github.com/user-attachments/assets/f36123d6-a1ba-4ae9-a890-0cf2d5042285



https://github.com/user-attachments/assets/1097b05f-34b5-40e8-9a98-f0ad992eee90





