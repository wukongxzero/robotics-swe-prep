---

## type: concept domain: Perception & ML status: drafted last-reviewed: tags: [ml, pinn, physics-ml, force-sensing, surgical]

# PINN Contact Estimation


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


## Links

- Related: [[Lagrangian Neural Networks]], [[Contact Modeling]], [[Jacobian]], [[Force Feedback & Haptics]], [[Robot Dynamics Formulations]]
- Parent: [[00 Knowledge Map]]

---

