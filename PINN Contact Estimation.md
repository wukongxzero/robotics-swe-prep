---

## type: concept domain: Perception & ML status: drafted last-reviewed: tags: [ml, pinn, physics-ml, force-sensing, surgical]

# PINN Contact Estimation

> [!question] Explain it cold
> 
> - What is a PINN and what goes in its loss function?
> - How does a PINN differ from an LNN?
> - Why frame contact-force estimation as a surgical force-sensing analogy?

---

## Core idea

A Physics-Informed Neural Network (PINN) embeds a known **physical law (a PDE/ODE) directly into the training loss** — the network is penalized both for mismatching data _and_ for violating the governing equation. This lets it learn from sparse data, respect physics, and even infer unmeasured quantities (like contact force) that are hard to sense directly.

## Key facts & formulas

- **Composite loss**: $$\mathcal L = \mathcal L_{data} + \lambda,\mathcal L_{physics}$$
    - $\mathcal L_{data}$ — fit to measured points (supervised).
    - $\mathcal L_{physics}$ — **residual of the governing equation** evaluated via autodiff at collocation points (e.g. plug the network's output into $M\ddot q + C\dot q + g - \tau - J^TF = 0$ and penalize the residual). The network's derivatives are taken by autodiff and required to satisfy the physics.
- **Why it helps**: the physics term regularizes — fewer data points needed, solutions stay physical, and you can _infer_ terms (force $F$) that appear in the equation but weren't directly measured.
- **PINN vs [[Lagrangian Neural Networks|LNN]]**: an LNN bakes structure into the _architecture_ (learns $L$, derives dynamics so physics holds exactly by construction). A PINN bakes physics into the _loss_ (soft constraint — penalized, not guaranteed). PINN is more flexible (any PDE, including dissipative/forced), LNN is stricter (conservation guaranteed). Different knob: architecture vs objective.
- **Collocation points**: where the physics residual is enforced — can be unlabeled, which is why PINNs need less labeled data.

## Where I've used it

- **FENCE-BOT**: wrote the PINN scaffolding — `pinn_contact.py`, `train_pinn.py`, `predict_contact.py` — for **contact force estimation**, framed as the analogy for **surgical force sensing**: estimating tool–tissue interaction force when you can't (or don't want to) put a direct force sensor at the tip. The contact force enters the dynamics as $J^TF$ ([[Jacobian]], [[Contact Modeling]]), so a PINN constrained by the equation of motion can infer $F$ from motion data.
- This is the **surgical differentiator tie-in** for the ML branch — force sensing is a core surgical-robotics problem ([[Force Feedback & Haptics]]).

## Interview follow-ups

- **Q:** What's in a PINN's loss?
    - **A:** A data term (fit measurements) plus a physics-residual term — the governing PDE/ODE evaluated via autodiff at collocation points and penalized when violated. The physics term regularizes and lets it infer unmeasured quantities.
- **Q:** PINN vs LNN?
    - **A:** LNN enforces physics in the architecture (learns the Lagrangian, conservation holds exactly); PINN enforces it as a soft loss penalty (flexible, handles forced/dissipative systems, but only approximately satisfied). Architecture-constraint vs objective-constraint.
- **Q:** Why use a PINN for contact force?
    - **A:** Contact force is hard to measure directly at a tool tip but appears in the dynamics as $J^TF$. A PINN constrained by the equation of motion can infer the force from observable motion, with the physics term compensating for sparse data — directly analogous to surgical tool–tissue force sensing.

## Gotchas / what trips me up

- Calling PINN physics a hard constraint — it's a _soft_ loss penalty (weighted by λ), unlike the LNN's architectural guarantee.
- Loss-balancing (the λ between data and physics terms) is finicky and a real training challenge.

## Links

- Related: [[Lagrangian Neural Networks]], [[Contact Modeling]], [[Jacobian]], [[Force Feedback & Haptics]], [[Robot Dynamics Formulations]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What goes in a PINN's loss function? ? A data term (fit measurements) + a physics-residual term: the governing PDE/ODE evaluated via autodiff at collocation points and penalized when violated. The physics term regularizes and enables inferring unmeasured quantities.

PINN vs LNN — where does the physics live? ? LNN: in the architecture (learns the Lagrangian, conservation holds exactly by construction). PINN: in the loss (soft penalty — flexible, handles forced/dissipative systems, only approximately satisfied).

Why use a PINN to estimate contact force? ? Contact force is hard to measure at the tool tip but enters the dynamics as JᵀF. A PINN constrained by the equation of motion infers F from observable motion — analogous to surgical tool–tissue force sensing.