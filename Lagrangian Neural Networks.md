---

## type: concept domain: Perception & ML status: drafted last-reviewed: tags: [ml, lnn, physics-ml, fence-bot, deep]

# Lagrangian Neural Networks

> [!star] You debugged this — strong "I found subtle ML bugs" story Fixing the LNN's structure-violating bugs is a sharp story: it shows you understand _both_ the physics and the ML, and can find errors that break the math invariants silently. Most candidates can't.

> [!question] Explain it cold
> 
> - What is an LNN and what does it preserve that a vanilla net doesn't?
> - What were the three bugs you fixed, and why did each break the physics?
> - Why did free-flight converge but contact underfit?

---

## Core idea

A Lagrangian Neural Network learns a system's **Lagrangian** $L = T - V$ with a neural net, then derives the dynamics through the **Euler–Lagrange equations** rather than predicting accelerations directly. By baking the physics structure into the architecture, it conserves energy and obeys the equations of motion _by construction_ — generalizing far better than a black-box net that just regresses next-state. (Sibling: Hamiltonian NNs, which learn $H = T+V$.)

## Key facts & formulas

- **Structure**: net outputs scalar $L(q,\dot q)$; the predicted acceleration comes from the Euler–Lagrange identity $$\ddot q = (\nabla_{\dot q}\nabla_{\dot q}^\top L)^{-1}\left[\nabla_q L - (\nabla_q\nabla_{\dot q}^\top L),\dot q\right]$$ — note it requires the **Hessian** of $L$ w.r.t. $\dot q$ (and it must be invertible). This is why second-derivative correctness matters so much. (See [[Robot Dynamics Formulations]] for the physics being preserved.)
- **Why it generalizes**: it can't predict energy-violating motion because the energy structure is hard-wired — unlike a vanilla net that can fit training data while violating conservation laws off-distribution.

## The three bugs I fixed (FENCE-BOT, Jordan's LNN)

1. **StandardScaler broke Euler–Lagrange consistency.** Normalizing inputs/outputs with a scaler applied an affine transform that the Euler–Lagrange derivation doesn't account for — the learned $L$ no longer corresponded to the _physical_ coordinates, so the derived dynamics were inconsistent. Fix: keep the physics in true coordinates (or fold scaling into the derivation consistently).
2. **Discrete `is_colliding` input corrupted the Hessians.** Feeding a discrete/boolean collision flag into the network meant the autodiff Hessian (∇²L) was taken w.r.t. a non-smooth input — garbage second derivatives, since the function isn't differentiable across the discrete jump. Fix: don't route discrete contact state through the smooth Lagrangian path.
3. **Per-sample training loop.** Training one sample at a time (vs batched) — inefficient and noisy gradient estimates. Fix: proper batching.

## Why free-flight worked but contact didn't

- **Free-flight LNN converged** to MSE ~1.12 — smooth, conservative dynamics, exactly what the LNN structure assumes.
- **Contact LNN underfit** — only **28 collision samples**, and contact dynamics are **non-smooth/discontinuous** ([[Contact Modeling]]), violating the smoothness the Euler–Lagrange + Hessian machinery relies on. Sparse data + non-smooth physics = the LNN's worst case. This isn't a bug, it's the structural limitation.


## Interview follow-ups

- **Q:** What does an LNN preserve that a normal network doesn't?
    - **A:** It learns the Lagrangian and derives dynamics via Euler–Lagrange, so energy conservation and the equations of motion hold by construction — it can't predict physically impossible, energy-violating trajectories the way a black-box regressor can off-distribution.
- **Q:** You normalized inputs with StandardScaler and the model broke — why?
    - **A:** The Euler–Lagrange derivation assumes the network's coordinates are the physical ones. An affine scaling makes the learned Lagrangian correspond to scaled coordinates, so the derived dynamics are inconsistent with the real system. The physics structure and the preprocessing fought each other.
- **Q:** Why did a discrete collision flag corrupt training?
    - **A:** The LNN takes second derivatives (Hessian of L) via autodiff; differentiating w.r.t. a discrete/non-smooth input gives meaningless second derivatives. Discrete contact state shouldn't flow through the smooth Lagrangian path.
- **Q:** Why did contact underfit but free-flight converge?
    - **A:** Free-flight is smooth and conservative — ideal for the LNN. Contact is non-smooth/discontinuous and I had only 28 samples; the smoothness assumption behind Euler–Lagrange and the Hessian breaks down, so it underfits. Structural limit, not a code bug.


## Links

- Related: [[Robot Dynamics Formulations]], [[Contact Modeling]], [[PINN Contact Estimation]], [[SymPy & scipy]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does a Lagrangian Neural Network preserve, and how? ? Energy conservation and the equations of motion — it learns the Lagrangian L=T−V and derives dynamics via Euler–Lagrange, so physics structure holds by construction (unlike a black-box next-state regressor).

Why did StandardScaler break the LNN? ? Euler–Lagrange assumes the net's coordinates are the physical ones; an affine scaling makes the learned Lagrangian correspond to scaled coordinates, so the derived dynamics become inconsistent with the real system.

Why did a discrete collision flag corrupt the LNN's training? ? The LNN takes the Hessian of L via autodiff; differentiating w.r.t. a discrete/non-smooth input gives meaningless second derivatives. Discrete contact state must not flow through the smooth Lagrangian path.

Why did the contact LNN underfit while free-flight converged (MSE ~1.12)? ? Free-flight is smooth/conservative — ideal for the LNN. Contact is non-smooth/discontinuous with only 28 samples, violating the smoothness Euler–Lagrange and the Hessian rely on. A structural limit, not a bug.