# ECE 528/L - Homework 2

**Author:** Ryan Fuentes
**Course:** ECE 528/L Robotics and Embedded Systems with Lab, Fall 2026

## Section I: Review Questions

---

### 1. What is a hardware timer? What tasks do embedded systems use them for?

A hardware timer is a peripheral inside the microcontroller that works like a counter driven by a clock signal. It keeps counting up or down on its own, so the CPU doesn't have to spend any time doing it, and it can set a flag or fire an interrupt once it reaches a certain value.

Embedded systems commonly use timers for:
- Accurate delays and timeouts
- Periodic interrupts, like sampling a sensor or running an RTOS scheduler tick
- Generating PWM signals for motors, LEDs, and servos

---

### 2. What is a clock signal used for? What are the primary clock sources on the MSP432P401R?

A clock signal is a steady square wave that keeps all the digital logic in the microcontroller in sync. It decides how fast the CPU runs instructions and it drives the peripherals like timers, UART baud rates, and the ADC, so basically everything is timed off of it.

The MSP432P401R has these clock sources:

| Source | Description |
|---|---|
| **LFXTCLK** | External 32.768 kHz crystal. Very accurate, good for real-time clock work. |
| **HFXTCLK** | External high-frequency crystal, roughly 1 MHz to 48 MHz. Accurate, but it needs an external crystal. |
| **DCOCLK** | Internal digitally controlled oscillator. It can be tuned from 1 MHz to 48 MHz and defaults to 3 MHz. This is the main source for MCLK and SMCLK. |
| **VLOCLK** | Internal very low power oscillator at about 9.4 kHz. It isn't very accurate and is mainly used in low power modes. |
| **REFOCLK** | Internal low power reference oscillator that can be set to 32.768 kHz or 128 kHz. |
| **MODCLK** | Internal low power oscillator at about 25 MHz that some modules like the ADC use. |
| **SYSOSC** | Internal 5 MHz oscillator used for system functions like the flash controller. |

These sources feed the system clocks. MCLK drives the CPU, HSMCLK and SMCLK drive the peripherals, ACLK is for low-frequency peripherals, and BCLK is for the backup and low-speed domain.

---

### 3. What is the max count of a 16-bit timer before it rolls over?

A 16-bit timer has 2^16 = 65,536 different values. It starts at 0, so the highest value it can reach is 2^16 - 1.

**Max count = 65,535 = 0xFFFF.** On the next clock tick it rolls back to 0. In up mode the timer only counts as high as the value in TAxCCR0, so the biggest value you can put there is 0xFFFF.

---

### 4. What is the overflow period of a 24-bit timer counting up at 10 MHz?

- Counts before overflow = 2^24 = 16,777,216
- Clock period = 1 / 10 MHz = 100 ns

Overflow period = 16,777,216 x 100 ns = 1,677,721,600 ns

**Overflow period is about 1.678 seconds.**

---

### 5. Describe the SysTick timer on the MSP432P401R

SysTick is a 24-bit down counter that is built into the ARM Cortex-M4 core. It loads a reload value and counts down to 0 on each clock tick. When it hits 0 it sets a flag called COUNTFLAG and can trigger an interrupt, then it reloads and starts over automatically.

Two example applications:
1. Making accurate delay functions, like a 1 ms or 1 s delay.
2. Making a periodic interrupt for things like an RTOS scheduler tick or sampling a sensor at a fixed rate.

SysTick can use either the processor clock, which is MCLK, or an external reference clock called STCLK from the clock system. I'd check the TRM for the exact STCLK frequency. The source is picked with the CLKSOURCE bit, which is bit 2 of the SysTick Control and Status Register. A 1 selects the processor clock and a 0 selects the external reference clock.

```c
SysTick->CTRL |= 0x04;    // use MCLK as the clock source
SysTick->CTRL &= ~0x04;   // use the external reference clock
```

---

### 6. How many registers does SysTick have and what does each do?

SysTick has **4 registers**:

| Register | Name | Purpose |
|---|---|---|
| **STCSR** or `SysTick->CTRL` | Control and Status | Enables the timer with bit 0, enables the interrupt with bit 1, picks the clock source with bit 2, and has COUNTFLAG at bit 16, which gets set when the counter reaches 0. |
| **STRVR** or `SysTick->LOAD` | Reload Value | Holds the 24-bit value the counter reloads with after it reaches 0. |
| **STCVR** or `SysTick->VAL` | Current Value | Holds the current 24-bit count. Writing any value to it clears it to 0 and also clears COUNTFLAG. |
| **STCALIB** or `SysTick->CALIB` | Calibration Value | Has calibration info, like the reload value for a 10 ms tick. It is mostly read only. |

---

### 7. Describe the Timer_A peripheral and its operating modes

Timer_A is a 16-bit timer on the MSP432P401R with multiple capture/compare registers. That lets it do PWM generation, input capture, output compare, and periodic interrupts. Its clock can come from ACLK, SMCLK, or external pins, and it has dividers to slow the clock down.

The operating modes are set with the MC bits in TAxCTL:

- **Stop mode, MC = 00b:** The timer is halted.
- **Up mode, MC = 01b:** The timer counts from 0 up to the value in TAxCCR0, then goes back to 0 and repeats. CCR0 sets the period.
- **Continuous mode, MC = 10b:** The timer counts from 0 up to 0xFFFF, then rolls over to 0 and repeats.
- **Up/Down mode, MC = 11b:** The timer counts from 0 up to TAxCCR0 and then back down to 0, then repeats. One full period is 2 x CCR0 counts.

---

### 8. Purpose of the Timer_A registers

- **TAxCTL:** The main control register. It selects the clock source with TASSEL, the input divider with ID, and the mode with MC. It also has TACLR to clear the timer, plus the interrupt enable TAIE and the interrupt flag TAIFG for timer overflow.
- **TAxCCTLn:** The control register for each capture/compare channel. It chooses between capture and compare mode, sets the output mode with OUTMOD, and sets the capture edge and input. It also has the interrupt enable CCIE and the flag CCIFG.
- **TAxR:** Holds the current 16-bit count of the timer.
- **TAxCCRn:** In compare mode it holds the value the timer is compared against, like the period in CCR0 or the duty cycle in CCR1 through CCR4. In capture mode it holds the timer value that was captured on an edge.
- **TAxIV:** The interrupt vector register. It tells you which interrupt is pending with the highest priority, and reading it clears that flag.
- **TAxEX0:** The expansion register. Its TAIDEX bits add an extra clock divider from 1 to 8 on top of the ID divider in TAxCTL.

---

### 9. What is PWM? Give example applications

PWM, or Pulse Width Modulation, is a square wave with a fixed period where the duty cycle can be changed. The duty cycle is the percent of the period that the signal is high. Changing it changes the average power sent to a load without changing the voltage level.

Duty cycle = high time / period x 100%

Example applications:
1. DC motor speed control, like in Lab 1
2. Servo motor position control, where the pulse width sets the angle
3. LED brightness dimming
4. Others include switching power supplies, making tones on a buzzer or speaker, and heater control

---

### 10. How many Timer_A modules are there and how many PWM outputs can each provide?

The MSP432P401R has **4 Timer_A modules**: Timer_A0, Timer_A1, Timer_A2, and Timer_A3.

CCR0 is normally used to set the period, so the PWM outputs come from the other capture/compare channels:
- Timer_A0, A1, and A2 each have 5 capture/compare registers, CCR0 through CCR4, so each one gives **4 PWM outputs** that share the same period.
- Timer_A3 has 2 capture/compare registers, so it gives **1 PWM output**.

I'd double check the exact number of CCRs per module in the device datasheet.

---

### 11. Initialize P2.4 as a PWM output driven by Timer_A0 CCR1 (TA0.1)

I assumed SMCLK is 12 MHz, same as the Timer_A2_PWM.c driver from Lab 1.

**Steps:**

1. Set P2.4 to its primary module function. That means setting Bit 4 of SEL0, clearing Bit 4 of SEL1, and setting Bit 4 of DIR so the pin is an output. The mask for bit 4 is `0x10`.
2. Put the period in `CCR[0]`. The timer clock is 12 MHz / 2 = 6 MHz, and in up/down mode a full period takes 2 x CCR0 ticks, so:
   - 20 ms = (2 x CCR0) / 6 MHz
   - CCR0 = (20 ms x 6 MHz) / 2 = **60000**
3. Clear `EX0` so the expansion divider is divide by 1.
4. Set the output mode of `CCTL[1]` to Toggle/Reset by making OUTMOD 010b, which is `0x0040`.
5. Put the duty cycle in `CCR[1]`. The duty cycle is CCR1 / CCR0, so for 50% CCR1 = 60000 x 0.5 = **30000**.
6. Finally, start the timer by setting the bits in `CTL`. TASSEL = 10b for SMCLK is `0x0200`, ID = 01b for divide by 2 is `0x0040`, and MC = 11b for up/down mode is `0x0030`. Together that is `0x0270`.

**Code:**

```c
#include "msp.h"
#include <stdint.h>

void Timer_A0_PWM_Init(uint16_t period_constant, uint16_t duty_cycle_1)
{
    // duty cycle can't be bigger than the period
    if (duty_cycle_1 >= period_constant) return;

    // P2.4 uses primary module function (TA0.1)
    // set bit 4 of SEL0, clear bit 4 of SEL1
    P2->SEL0 |= 0x10;
    P2->SEL1 &= ~0x10;

    // P2.4 is an output
    P2->DIR |= 0x10;

    // period = (2 * 60000) / (12 MHz / 2) = 20 ms
    TIMER_A0->CCR[0] = period_constant;

    // divide by 1 on the expansion register
    TIMER_A0->EX0 = 0x0000;

    // toggle/reset for CCR[1], OUTMOD = 010b
    TIMER_A0->CCTL[1] |= 0x0040;

    // duty cycle = duty_cycle_1 / period_constant
    TIMER_A0->CCR[1] = duty_cycle_1;

    // SMCLK (TASSEL = 10b), divide by 2 (ID = 01b), up/down (MC = 11b)
    TIMER_A0->CTL |= 0x0270;
}

int main(void)
{
    // 60000 -> 20 ms period, 30000 -> 50% duty
    Timer_A0_PWM_Init(60000, 30000);

    while (1)
    {
        // PWM runs on its own
    }
}
```

---

### 12. Configure Timer_A0 for a periodic interrupt every 2 ms with priority level 2

I assumed SMCLK is 12 MHz again.

**Steps:**

1. Stop the timer while configuring by clearing the MC bits in `CTL` with `~0x0030`.
2. In `CTL`, pick SMCLK with TASSEL = 10b, which is `0x0200`. The prescaler of 1 means ID = 00b, so there is nothing to set there.
3. Set `EX0` to `0x0000` so TAIDEX = 000b and the clock is divided by 1.
4. Calculate the period. The timer clock is 12 MHz / 1 / 1 = 12 MHz, and 2 ms x 12 MHz = 24000 ticks. The timer counts from 0, so CCR0 = 24000 - 1 = **23999**.
5. Enable the CCR0 interrupt with the CCIE bit in `CCTL[0]`, which is `0x0010`.
6. Set the NVIC priority of interrupt 8 to 2. Interrupt 8 is in IPR2 and the priority lives in the top 3 bits of its byte, so priority 2 is `2 << 5 = 0x40`.
7. Enable interrupt 8 in the NVIC by setting Bit 8 of `ISER[0]`, which is `0x00000100`.
8. Set TACLR, which is `0x0004`, and up mode, which is MC = 01b or `0x0010`, in `CTL` to start the timer. Together that is `0x0014`.
9. Write the ISR `TA0_0_IRQHandler` and clear the CCIFG flag inside it.

**Code:**

```c
#include "msp.h"
#include <stdint.h>

// task that runs every time the interrupt happens
void (*Timer_A0_Task)(void);

void Timer_A0_Interrupt_Init(void(*task)(void), uint16_t period)
{
    // save the task function
    Timer_A0_Task = task;

    // stop timer A0 (clear MC bits)
    TIMER_A0->CTL &= ~0x0030;

    // SMCLK as clock source (TASSEL = 10b), prescaler of 1 (ID = 00b)
    TIMER_A0->CTL |= 0x0200;

    // enable the CCR0 interrupt (CCIE bit)
    TIMER_A0->CCTL[0] |= 0x0010;

    // timer starts at 0, so period - 1
    // 2 ms * 12 MHz = 24000 -> CCR0 = 23999
    TIMER_A0->CCR[0] = (period - 1);

    // TAIDEX = 000b (divide by 1)
    TIMER_A0->EX0 = 0x0000;

    // priority 2 for interrupt 8 (IPR2), top 3 bits of the byte -> 2 << 5 = 0x40
    NVIC->IP[2] = (NVIC->IP[2] & 0xFFFFFF00) | 0x00000040;

    // enable interrupt 8 in the NVIC (bit 8 of ISER)
    NVIC->ISER[0] |= 0x00000100;

    // set TACLR and start in up mode (MC = 01b)
    TIMER_A0->CTL |= 0x0014;
}

void TA0_0_IRQHandler(void)
{
    // clear the CCIFG flag
    TIMER_A0->CCTL[0] &= ~0x0001;

    // run the task
    (*Timer_A0_Task)();
}

// example task, runs every 2 ms
volatile uint32_t tick_count = 0;
void my_task(void)
{
    tick_count++;
}

int main(void)
{
    // period of 24000 ticks = 2 ms at 12 MHz
    Timer_A0_Interrupt_Init(&my_task, 24000);

    // turn on interrupts
    __enable_irq();

    while (1)
    {
        // ISR does the work
    }
}
```
