---

## type: concept domain: Compute & acceleration status: drafted last-reviewed: tags: [compute, nixos, reproducibility, embedded]

# NixOS for Jetson


---

## Core idea

Nix is a purely-functional package manager (and NixOS, a Linux distro built on it) where the _entire system configuration_ is declared in code and builds are reproducible by construction — same inputs always produce the same output, with no hidden global state. For an embedded robot, that means the board's whole software environment is version-controlled and reproducible bit-for-bit.

## Key facts & formulas

- **Declarative**: you describe the desired system state (packages, services, configs) in a Nix expression; Nix realizes it. No imperative "install this, then that" drift.
- **Reproducible/hermetic**: every package is built in isolation with pinned inputs and stored by content hash in `/nix/store`. Two machines with the same config get identical environments — no "works on mine."
- **Atomic upgrades & rollback**: system generations — upgrade is atomic, and you can roll back to a previous generation if something breaks. Valuable on a robot you can't easily re-flash in the field.
- **vs Docker**: Docker reproduces a _container image_; Nix reproduces the _build itself_ (and can build the image). They compose — Nix for hermetic builds, Docker for deployment/isolation. ([[Docker for ROS]])
- **The cost**: steep learning curve, the Nix language, and ARM/Jetson + proprietary NVIDIA drivers (CUDA) are genuinely painful to get working under Nix's purity model.


## Links

- Related: [[Docker for ROS]], [[Jetson Orin Setup]], [[AVR Peripherals]]
- Parent: [[00 Knowledge Map]]

---

