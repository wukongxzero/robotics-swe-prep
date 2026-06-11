---

## type: concept domain: Real-time & embedded status: drafted last-reviewed: tags: [embedded, avr, arduino, registers, kapila-final]

# AVR Register Programming

> [!question] Explain it cold
> 
> - What do DDRx, PORTx, and PINx do?
> - How do you set one pin without disturbing the others?
> - Why bypass `digitalWrite()` and go to registers?

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

## Where I've used it

- **WALL-E V3**: Arduino UNO ran the 200 Hz gimbal loop and Arduino Mega ran tank motor control. Register-level control matters when you're hitting 200 Hz and bit-banging the [[Serial Packet Protocols|TankStatus protocol]] — the HAL overhead is real at that rate.
- **Kapila final (ROB-GY 6103)**: drilled DDRx/PORTx, USART, Timer2, I2C/TWI, ADC, PWM at register level.

## Interview follow-ups

- **Q:** Set pin 13 (PB5) high without touching other pins on port B?
    - **A:** `PORTB |= (1<<PB5);` — OR-assign so other bits are preserved. Never `PORTB = 0x20` (clobbers the rest).
- **Q:** Why does `PORTB = 0b00100000` risk a bug?
    - **A:** It overwrites the entire port, clearing every other pin. Always use read-modify-write (`|=`, `&= ~`).
- **Q:** What's the fastest way to toggle a pin?
    - **A:** `PORTB ^= (1<<PB5)`, or the AVR trick of writing 1 to the PINB bit, which toggles PORTB in one instruction.

## Gotchas / what trips me up

- Assignment (`=`) vs read-modify-write (`|=`, `&=~`) — assignment clobbers neighbors.
- Forgetting that writing PINx toggles PORTx (looks like a no-op read, isn't).

## Links

- Related: [[AVR Peripherals]], [[Serial Packet Protocols]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

DDRx, PORTx, PINx — what does each control? ? DDRx = direction (1 out / 0 in); PORTx = output level (or pull-up enable when input); PINx = read live pin state.

Set, clear, and toggle bit n in a register — the three idioms? ? Set: REG |= (1<<n). Clear: REG &= ~(1<<n). Toggle: REG ^= (1<<n).

Why use register writes over digitalWrite() on a tight loop? ? digitalWrite does pin lookups and interrupt masking (~50× slower); register writes are single instructions — needed for deterministic timing and bit-banging.

What does writing a 1 to a PINx bit do on AVR? ? It toggles the corresponding PORTx output bit — a one-instruction toggle trick.