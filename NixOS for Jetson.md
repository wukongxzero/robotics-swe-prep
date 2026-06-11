---

## type: concept domain: Compute & acceleration status: drafted last-reviewed: tags: [compute, nixos, reproducibility, embedded]

# NixOS for Jetson

> [!question] Explain it cold
> 
> - What makes Nix different from apt/pip/Docker for reproducibility?
> - What does "declarative, reproducible system" actually mean?
> - Why would you run NixOS on an embedded robot board?

---

## Core idea

Nix is a purely-functional package manager (and NixOS, a Linux distro built on it) where the _entire system configuration_ is declared in code and builds are reproducible by construction — same inputs always produce the same output, with no hidden global state. For an embedded robot, that means the board's whole software environment is version-controlled and reproducible bit-for-bit.

## Key facts & formulas

- **Declarative**: you describe the desired system state (packages, services, configs) in a Nix expression; Nix realizes it. No imperative "install this, then that" drift.
- **Reproducible/hermetic**: every package is built in isolation with pinned inputs and stored by content hash in `/nix/store`. Two machines with the same config get identical environments — no "works on mine."
- **Atomic upgrades & rollback**: system generations — upgrade is atomic, and you can roll back to a previous generation if something breaks. Valuable on a robot you can't easily re-flash in the field.
- **vs Docker**: Docker reproduces a _container image_; Nix reproduces the _build itself_ (and can build the image). They compose — Nix for hermetic builds, Docker for deployment/isolation. ([[Docker for ROS]])
- **The cost**: steep learning curve, the Nix language, and ARM/Jetson + proprietary NVIDIA drivers (CUDA) are genuinely painful to get working under Nix's purity model.

## Where I've used it

- **WALL-E V3 / FENCE-BOT**: ran a **NixOS Jetson (nixjetson)** for reproducible embedded builds alongside the Docker path — using `arduino-cli` and integrating [[AVR Peripherals|TCA9548A/AS5600 encoders]]. The reproducibility was the draw: a declarative robot environment vs hand-installed drift.

## Interview follow-ups

- **Q:** What does Nix give you that Docker doesn't?
    - **A:** Nix reproduces the _build_ hermetically — pinned inputs, content-addressed store, no hidden state — and declares the whole system, with atomic rollback. Docker reproduces an image; they compose (Nix can build the image). Nix is reproducibility at the build/system level.
- **Q:** Why care about reproducibility on a robot?
    - **A:** A field robot you can't casually re-flash benefits from a version-controlled, declarative environment with atomic rollback — if an update breaks, you revert to a known-good generation deterministically.
- **Q:** What's hard about NixOS on a Jetson?
    - **A:** ARM plus proprietary NVIDIA/CUDA drivers fight Nix's purity model, and the learning curve/language is steep. It's a real investment for the reproducibility payoff.

## Gotchas / what trips me up

- Proprietary CUDA drivers vs Nix purity — the main friction on Jetson.
- Steep Nix-language learning curve; easy to underestimate.

## Links

- Related: [[Docker for ROS]], [[Jetson Orin Setup]], [[AVR Peripherals]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What makes Nix reproducible where apt/pip aren't? ? Purely-functional: every package builds in isolation from pinned inputs, stored by content hash, with no hidden global state — same inputs always yield the same output, and the whole system is declared in code.

Nix vs Docker for reproducibility? ? Docker reproduces a container image; Nix reproduces the build itself (hermetic, pinned) and declares the whole system with atomic rollback. They compose — Nix can build the image.

Why is NixOS hard on a Jetson? ? Proprietary NVIDIA/CUDA drivers and ARM fight Nix's purity model, plus a steep language/learning curve.