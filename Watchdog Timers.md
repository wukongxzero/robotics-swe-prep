---

## type: concept domain: Real-time & embedded status: drafted last-reviewed: tags: [embedded, watchdog, safety, debugging, differentiator]

# Watchdog Timers


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


## Links

- Related: [[AVR Peripherals]], [[Real-Time Determinism]], [[Safety-Critical Architecture]], [[Serial Packet Protocols]]
- Parent: [[00 Knowledge Map]]

---

