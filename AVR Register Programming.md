---

## type: concept domain: Real-time & embedded status: drafted last-reviewed: tags: [embedded, avr, arduino, registers, kapila-final]

# AVR Register Programming


---

## Core idea

On AVR (ATmega328/2560 — Arduino UNO/Mega), GPIO is controlled by three registers per port: **DDRx** sets direction, **PORTx** sets output level (or pull-up), **PINx** reads input. Direct register manipulation is faster and deterministic vs the Arduino HAL.

## Key facts & formulas

- **DDRx** (Data Direction Register): bit = 1 → output, 0 → input. `DDRB |= (1<<PB5)` makes PB5 an output.
- **PORTx**: when output, sets HIGH/LOW. When input, 1 enables the internal pull-up.
- **PINx**: read the live pin state. Writing 1 to a PINx bit _toggles_ the corresponding PORTx bit (AVR trick).
- **Bit manipulation idioms** (the core skill):
    - Set bit: `PORTB |= (1<<PB5)`
    - Clear bit: `PORTB &= ~(1<<PB5)`
    - Toggle bit: `PORTB ^= (1<<PB5)`
    - Test bit: `if (PINB & (1<<PB0))`
- **Why registers over digitalWrite**: `digitalWrite()` does pin-mapping lookups, interrupt-disable, and is ~50× slower. Register writes are single instructions — necessary for bit-banged protocols and tight timing.
- Same pattern generalizes to every peripheral: configure by setting bits in control registers (see [[AVR Peripherals]]).


## Links

- Related: [[AVR Peripherals]], [[Serial Packet Protocols]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

