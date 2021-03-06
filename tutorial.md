---
layout: page
---

# Tutorial

<div class="callout">
  This tutorial is still incomplete. Please contribute by clicking on the `Edit this page on GitHub` link at the bottom of this page.
</div>

{% include toc.md %}

## Registers

All registers are in undefined state at startup.
The "Reset" column in the datasheets is misleading (and only emulated by the original IDE which generates code to set all registers to 0).
You have to initialize all registers yourself.
> Source: <https://www.eevblog.com/forum/blog/eevblog-1144-padauk-programmer-reverse-engineering/msg3129442/#msg3129442>

## Clock Sources and Setup

<div class="callout">
  TODO: This section is still missing and should explain the following terms:

  _sdcc_external_startup, IHRC, ILRC, external crystal, SYSCLK, maximum possible clk frequency, factory calibrated values, easy pdk calibration, ...
</div>

## Digital I/O

Digital I/O is organized in ports of (at most) 8 pins. The ports are named `A`, `B`, and `C`.
I/O pins are controlled by the following registers (replace `x` by `A`, `B`, or `C`):

- `PxC` **Control Register**: Controls whether the pin is used as an input (`0`) or output (`1`).
- `Px` **Data Register**: Sets the output low (`0`) or (`high`). Has no effect on inputs.
- `PxPH` **Pull-up ("pull-high") Register**:
  Enables a pull-up resistor. The resistor is also active when the pin is used as an output and driven high (!)
- `PxPL` **Pull-down ("pull-low") Register**:
  Enables a pull-down resistor (only very few ports and pins have pull-down resistors).
  It is unknown whether the resistor is also active when the pin is used as an output and driven high (!)
- `PxDIER` **Digital Input Enable Register**: For pins to work as digital inputs, this register needs to be set to `1`.
  Otherwise the input signal is cut off from the digital circuitry (including the external interrupt hardware + wake up functionality).
  This is recommended when using the pin as an analog input to the ADC or comparator, and required when using the pin as input of an external crystal.

The following table provides an overview of how the registers work together to control an I/O pin (this example uses PB.0).

| PBC.0 | PB.0 | PBPH.0 | PBPL.0 | PBDIER.0 | Result                                                               |
| ----- | ---- | ------ | ------ | -------- | -------------------------------------------------------------------- |
| 0     | x    | x      | x      | 0        | Digital I/O disabled (can only use pin for analog input and crystal) |
| 0     | x    | 0      | 0      | 1        | Input without pull resistors                                         |
| 0     | x    | 1      | 0      | 1        | Input with pull-up resistor                                          |
| 0     | x    | 0      | 1      | 1        | Input with pull-down resistor                                        |
| 0     | x    | 1      | 1      | 1        | *not explicitly mentioned in datasheet*                              |
| 1     | 0    | x      | x      | x        | Output low wihout pull resistors                                     |
| 1     | 1    | 0      | 0      | x        | Output high wihout pull resistors                                    |
| 1     | 1    | 1      | 0      | x        | Output high with pull-up resistor                                    |
| 1     | 1    | 1      | 1      | x        | *not explicitly mentioned in datasheet*                              |
| 1     | 1    | 0      | 1      | x        | *not explicitly mentioned in datasheet*                              |

### Maximum Current

The maximum current an I/O pin can drive and sink varies by pin and µC.
Take a look at the [interactive pinout diagram of an individual µC](/chips/PFS173) and enable "Maximum Sink/Drive Current" to learn more.

<div class="callout">
  TODO: Many µCs can increase the maximum current on two of their pins.

  -> How do I enable that?
  -> Is there any downside to enabling that?
</div>


## Interrupts

Padauk µCs provide up to 8 interrupt sources.
Interrupts are configured with the following registers:

- `INTEGS`: Interrupt Edge Select Register
- `INTRQ`:  Interrupt Request Register
- `INTEN`:  Interrupt Enable Register

Whenever an interrupt occurs, the corresponding bit in the `INTRQ` register is set to `1` by hardware.
The µC never sets the bit back to `0`. This has to be done by your program.

If you want to trigger an interrupt service routine (ISR) when an interrupt occurs,
then you have to individually enable it for each interrupt source by setting the corresponding bit to `1` in the `INTEN` register.
In addition, you need to enable interrupts globally by calling `__engint()`.

All interrupts share a single ISR, which is denoted by adding the `__interrupt(0)` attribute to the function definition.
Inside the ISR, you have to check which interrupt(s) triggered it by checking the corresponding bits in the `INTRQ` register.
It is possible for multiple interrupts to occur at the same time

If you handle multiple interrupts in your ISR **and** you disable an interrupt conditionally by setting its corresponding bit in `INTEN` to `0` at some point in your program, you also have to check the `INTEN` register when the ISR is called. This is because the bit in `INTRQ` is still set even if the interrupt is disabled in `INTEN`.
The alternative is to disable **all** interrupts whenever you want to disable a single interrupt by calling `__disgint()`.

The following example shows how to use the two external interrupts on a PFS173:

```c
#include <stdint.h>
#include <pdk/device.h>

volatile uint16_t counter;

void interrupt(void) __interrupt(0)
{
  if (
    INTEN & INTEN_PA0  // Is the interrupt enabled?
    &&
    INTRQ & INTRQ_PA0  // Did the interrupt occur?
  ) {
    // Reset interrupt request bit
    INTRQ &= ~INTRQ_PA0;

    // Handle interrupt
    counter++;
  }

  if (
    INTEN & INTEN_PB0  // Is the interrupt enabled?
    &&
    INTRQ & INTRQ_PB0  // Did the interrupt occur?
  ) {
    // Reset interrupt request bit
    INTRQ &= ~INTRQ_PB0;

    // Handle interrupt
    // ...
  }
}

void main(void)
{
  INTEN = INTEN_PA0 | INTEN_PB0;
  INTEGS = INTEGS_PA0_BOTH | INTEGS_PB0_BOTH;

  // Further initialization code

  // Enable interrupts.
  INTRQ = 0;
  __engint();

  for(;;) {
    // Make sure to temporarily disable the interrupt when
    // reading the variable from the main program, since access
    // to 16 bit variables is not atomic.
    uint16_t i;

    INTEN &= ~INTEN_PA0;
    // The ISR is now no longer called when an interrupt at PA0 occurrs.
    // However, the INTRQ_PA0 bit in INTRQ will still be set if an interrupt at PA0 occurrs.

    // -> Let's say at this moment the signal at PA0 changes
    // -> INTRQ & INTRQ_PA0 is set to 1 by hardware
    // -> The ISR is _not_ called, because INTEN & INTEN_PA0 is not set.

    i = counter;

    // -> Let's say at this moment the signal at PB0 changes
    // -> INTRQ & INTRQ_PB0 is set to 1 by hardware
    // -> The ISR is called, because INTEN & INTEN_PB0 is set AND interrupts are globally enabled.
    //
    // Important: The INTRQ_PA0 bit is still set to 1!
    // If we don't check INTEN & INTEN_PA0 in the ISR, the code for PA0 would be triggered even
    // though we disabled the interrupt.

    // Re-enable the PA0 interrupt now that we have read the counter variable.
    INTEN |= INTEN_PA0;

    // ...
  }
}

unsigned char _sdcc_external_startup(void)
{
  // ...
}
```