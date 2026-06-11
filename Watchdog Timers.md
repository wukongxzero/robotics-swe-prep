---

## type: concept domain: Real-time & embedded status: drafted last-reviewed: tags: [embedded, watchdog, safety, debugging, differentiator]

# Watchdog Timers

> [!question] Explain it cold
> 
> - What is a watchdog timer and what failure does it catch?
> - Plain vs _windowed_ WDT — what extra failure does windowed catch?
> - Why does the watchdog's _clock source_ matter for safety?
> - What's the difference between a blunt reset and graceful degradation?

---

## Core idea

A watchdog timer (WDT) is an independent timer that resets (or interrupts) the system if application code fails to periodically "kick" it. It catches hangs, infinite loops, and deadlocks that code can't self-detect — because wedged code can't notice it's wedged. The watchdog _family_ ranges from a one-bit MCU timer up to multi-stage, independently-clocked, externally-monitored safety systems.

## Key facts & formulas

### The basic mechanism

- WDT counts down from a timeout value; healthy code "kicks"/"feeds" it before expiry. Reaching zero → reset (or NMI/interrupt first).
- **Independence is the whole point**: it's separate from the code it watches, so it still fires when the main loop is dead. Self-monitoring by the broken thing doesn't work.
- **On AVR**: WDTCSR (timeout prescaler bits), `wdt_enable()`, `wdt_reset()` to kick; a timed write sequence is required to change settings (prevents accidental misconfig).
- **Kick placement rule**: kick from the main loop _after confirming forward progress_ — never from an ISR or tight inner loop, which keeps feeding the dog while the outer loop is hung.

### Watchdog family (the part I was missing)

- **Plain (timeout) WDT** — catches _too slow_: kick didn't arrive in time. Misses _too fast_.
- **Windowed WDT (e.g. STM32 WWDG)** — you must kick inside a window: not before it opens, not after it closes. Catches _too fast_ too — kicking early signals a runaway loop or stuck ISR feeding the dog off-cadence. Enforces the "kick only on real progress" discipline _in hardware_ — you literally can't paper over a hang by kicking too often.
- **Independent clock source (e.g. STM32 IWDG on its own internal RC oscillator)** — the watchdog runs off a _separate_ clock from the core. If the main clock tree fails or the PLL unlocks, a watchdog sharing that clock dies with it and can't fire. Independent clocking is what makes it a true backstop.
- **Multi-stage watchdog** — expiry first fires an NMI/interrupt (capture state, log the reset cause, attempt a safe shutdown / move to a fail-safe state), _then_ a hard reset on the second stage if recovery doesn't happen. Buys you graceful degradation instead of a blunt reboot.
- **Software / task watchdog** — in an RTOS, a supervisor task each monitored task must check in with. Catches one hung task without nuking the whole system; lets you restart just that task. The link to [[Safety-Critical Architecture]].
- **External / PMIC watchdog IC** — a discrete chip (or one inside the power-management IC) _outside_ the MCU entirely. Catches the case where the whole MCU — including its internal WDT — is wedged or browning out. Belt-and-suspenders for safety-critical designs; often required for certification.

### Diagnostics (the half I underweighted)

- **Reset-cause register**: after a reset the MCU latches the source — **MCUSR** on AVR (WDRF bit = watchdog reset flag), RCC reset flags on STM32. Read it on boot to _know_ it was the dog vs a brownout vs a power cycle.
- **Boot-loop handling**: read and clear the flag on boot; if you see repeated WDT resets, fall to a safe mode / log instead of resetting forever. (Also: clear MCUSR and disable the WDT early in startup on AVR, or a short timeout can trap you in a reset loop before `setup()` finishes.)

## Where I've used it

- **WALL-E V3 (hands-on war story)**: extended debugging on **WDT timeout resets** tied to **I²C bus failures** — when the [[AVR Peripherals|TCA9548A mux / AS5600 encoder]] I²C transaction hung, the main loop stalled and the WDT reset the board. The reset _was the symptom pointing at the real bug_: a blocking I²C read with no timeout. Two-part fix: add a timeout/recovery to the I²C path so it can't block, keep the WDT as the backstop. Reading MCUSR/WDRF on boot is how you'd confirm the dog (not a brownout) caused it.
- **Surgical-robotics framing (design-level)**: the windowed + independent-clock + multi-stage pattern is exactly what a safety-critical actuator controller wants — a hung control loop must reach a _defined fail-safe state_ (motors disabled, brakes engaged), not just reboot mid-motion. _(If I ran a windowed/external watchdog at Articulus, anchor it here with specifics.)_

## Interview follow-ups

- **Q:** Plain WDT vs windowed WDT — what does windowed add?
    - **A:** Plain catches "too slow" (missed kick). Windowed also catches "too fast" — you must kick inside a window, so a runaway loop or stuck ISR feeding the dog early trips a fault. It enforces correct kick cadence in hardware.
- **Q:** Why does the watchdog's clock source matter?
    - **A:** If it shares the core's clock and that clock fails (PLL unlock, oscillator dead), the watchdog fails with it — exactly when you need it. An independent oscillator (STM32 IWDG) keeps it firing through a clock fault.
- **Q:** Your board keeps resetting every few seconds — diagnose it.
    - **A:** Read the reset-cause register (MCUSR/WDRF). If it's the watchdog, find the hang: instrument around blocking peripheral reads (I²C/SPI with no timeout are classic), check kick placement. The reset means the loop never completed forward progress.
- **Q:** Why not just make the timeout long to stop the resets?
    - **A:** That hides the real bug and weakens the guarantee. Fix the hang; size the timeout to the loop's worst-case execution time, not to paper over it.
- **Q:** Why kick from the main loop, not an ISR?
    - **A:** An ISR fires even when the main loop is wedged — kicking there feeds the dog while the system is hung. Kick only after confirming real forward progress. A windowed WDT enforces this in hardware.
- **Q:** Reset is blunt — how do you degrade gracefully instead?
    - **A:** Multi-stage watchdog: first expiry fires an NMI to capture state and command a fail-safe state (disable actuators, log), then hard-reset only if that doesn't complete. In an RTOS, a task watchdog restarts the one hung task instead of the whole system.

## Gotchas / what trips me up

- Calling all watchdogs the same — plain vs windowed vs independent-clock vs external IC are different guarantees.
- Kicking in the wrong place masks the hang it's meant to catch (windowed WDT defends against this).
- Treating WDT resets as noise instead of a signal — they point at a blocking/hung path.
- Forgetting to read/clear the reset-cause flag → you never learn it was the dog, and risk a boot loop.
- Watchdog sharing the failing clock — looks safe, isn't.

## Links

- Related: [[AVR Peripherals]], [[Real-Time Determinism]], [[Safety-Critical Architecture]], [[Serial Packet Protocols]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does a watchdog timer catch that code can't self-detect? ? Hangs, deadlocks, and infinite loops — wedged code can't notice it's wedged, so an independent timer resets the system if not periodically kicked.

Plain WDT vs windowed WDT — what extra failure does windowed catch? ? Plain catches "too slow" (missed kick). Windowed also catches "too fast": you must kick inside a window, so feeding the dog early (runaway loop / stuck ISR) trips a fault.

Why does the watchdog's clock source matter for safety? ? If it shares the core clock and that clock fails (PLL unlock, dead oscillator), the watchdog dies with it. An independent oscillator (e.g. STM32 IWDG) keeps it firing through a clock fault.

What does a multi-stage watchdog give you over a plain reset? ? First expiry fires an NMI/interrupt to capture state and command a fail-safe state (disable actuators, log); hard reset only on the second stage — graceful degradation instead of a blunt reboot.

What is an external / PMIC watchdog for? ? A watchdog chip outside the MCU that catches the case where the whole MCU — including its internal WDT — is wedged or browning out. Belt-and-suspenders for safety-critical / certified designs.

After a reset, how do you know the watchdog caused it? ? Read the latched reset-cause register — MCUSR (WDRF bit) on AVR, RCC reset flags on STM32. Read and clear it on boot to distinguish WDT reset from brownout/power-cycle and to break boot loops.

Why kick the WDT from the main loop, not an ISR? ? An ISR keeps firing even when the main loop is hung, feeding the dog while the system is wedged. Kick only after confirming forward progress; a windowed WDT enforces this in hardware.