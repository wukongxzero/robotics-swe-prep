---

## type: concept domain: Real-time & embedded status: drafted last-reviewed: tags: [embedded, avr, usart, timers, pwm, adc, i2c, kapila-final]

# AVR Peripherals

> [!question] Explain it cold
> 
> - Derive the USART baud register value.
> - How do you get a specific timer overflow period?
> - What sets the I2C clock speed?

---

## Core idea

Every AVR peripheral (serial, timers, PWM, ADC, I2C) is configured by writing control/prescaler registers. Three formulas recur constantly and were drilled for the Kapila final — memorize the derivations, not just the results.

## The three formulas

### USART baud rate

$$\text{UBRR} = \frac{F_{CPU}}{16 \times \text{baud}} - 1$$

- `UBRR` is the baud-rate register. The 16 is the oversampling factor in normal mode (8 in double-speed U2X mode).
- Example: 16 MHz, 9600 baud → UBRR = 16e6/(16×9600) − 1 ≈ 103.

### Timer overflow period

$$t_{overflow} = \frac{(\text{MAX}+1) \times \text{prescaler}}{F_{CPU}}$$

- MAX = 255 (8-bit timer) or 65535 (16-bit). MAX+1 is the number of ticks to overflow.
- Prescaler divides the clock (1, 8, 64, 256, 1024 on AVR).
- Example: 16 MHz, 8-bit, prescaler 256 → (256×256)/16e6 = 4.096 ms per overflow.

### I2C (TWI) bit rate

$$\text{TWBR} = \frac{F_{CPU}/f_{SCL} - 16}{2 \times \text{prescaler}}$$

- TWBR = TWI Bit Rate register; prescaler from TWSR bits (1, 4, 16, 64).
- Example: 16 MHz, 100 kHz SCL, prescaler 1 → (160−16)/2 = 72.

## Key facts

- **Timers/PWM**: AVR timers drive PWM via compare-match (OCRx) — set mode (Fast PWM / Phase-Correct) in TCCRx, duty via OCRx. Timer2 specifically came up in the final.
- **ADC**: 10-bit, configured via ADMUX (reference + channel) and ADCSRA (enable, prescaler, start). Result in ADCH:ADCL.
- **PWM frequency** = timer overflow frequency; **duty** = OCRx / (MAX+1).


## Interview follow-ups

- **Q:** Why the "−1" in UBRR and the "16"?
    - **A:** 16 is the samples-per-bit oversampling in normal mode; the −1 is because the counter counts from 0, so the register holds (divisor − 1).
- **Q:** You need a 1 ms timer tick at 16 MHz — how?
    - **A:** Pick prescaler and compare value: with prescaler 64, 16e6/64 = 250 kHz → 250 ticks = 1 ms, so use CTC mode with OCR = 249.
- **Q:** PWM frequency too low, causing audible whine — what do you change?
    - **A:** Lower the prescaler or use a smaller TOP (Fast PWM with ICR/OCR as TOP) to raise the overflow frequency above ~20 kHz.


## Links

- Related: [[AVR Register Programming]], [[Motor Control]], [[Serial Packet Protocols]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

USART baud register formula? ? UBRR = F_CPU / (16 × baud) − 1. The 16 is oversampling (8 in U2X mode); the −1 because the counter starts at 0.

Timer overflow period formula? ? t = (MAX+1) × prescaler / F_CPU, where MAX is 255 (8-bit) or 65535 (16-bit).

I2C TWBR formula? ? TWBR = (F_CPU/f_SCL − 16) / (2 × prescaler), with prescaler set in TWSR.

At 16 MHz, prescaler 256, 8-bit timer — overflow period? ? (256 × 256) / 16e6 = 4.096 ms.