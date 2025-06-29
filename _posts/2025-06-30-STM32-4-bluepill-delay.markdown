---
layout: post
title:  "STM32 #4: The interupt counter on Bluepill with Libopencm3 SysTick "
date:   2025-06-29 00:00:00 +0000
categories: STM32
---

In this post we will progress from using the assembler code __asm__("nop") to using a interupt based delay via the libopencm3 library on the Bluepill STM32F1 board. See [STM32: From Template to Blinky - Building Bluepill Firmware with libopencm3](https://skoopsy.dev/stm32/2025/06/29/STM32-3-blinky-bluepill.html) for how to get a blinky running with libopencm3 on a bluepill using make. 

The main.c code from the last blinky is as follows:

```c
int main(void) {

    // Enable the clock for GPIO port B via RCC (Reset & Clock control)
    // This is defined in the libopencm3/stm32/rcc.h file
    rcc_perip_clock_enable(RCC_GPIOB)

    // Set the PB2 pin as an output pin, in push-pull mode, using the 2 MHz clock
    // This is defined in the libopencm3/stm32/gpio.h file
    gpio_set_mode(GPIOB, // Which GPIO port to target
                  GPIO_MODE_OUTPUT_2_MHZ, // The output singla clock (alternatives: 10 & 50 Mhz)
                  GPIO_CNF_OUTPUT_PUSHPULL, // Sets the pin as a push-pull output
                  GPIO2 // The pin to set on the GPIO port.
                  )

    // Blink loop
    while(1) {

        // Toggle the logic level of the pin, high becomes low and vice versa
        // Defined in libopencm3/stm32/gpio.h
        gpio_toggle(GPIOB, GPIO2);
        for (int i = 0; i < 500000; i++) {__asm__("nop");}
        }
    }
```

# SysTick
Rather than using a the assembler delay, we can use a interup based timer delay to tick away in the background until it needs to fire, allowing the processing core to do other things in the mean time like check sensors, or write memory, or something.

Libopencm3 has a systick module. This accesses a 24-bit timer which is built into every Cortex M core (from ARM themselves, not STM32), it is typically used for periodic interrupts, system time keeping, and delays like the msleep(), and usleep() functions.

It counts downwards, when it hits zero, it can generate an interrupt and reload automatically.

SysTick has a small set of registers:
- STK_CTRL - control & status
- STK_LOAD - Reload tick period
- STK_VAL - Current value
- STK_CALIB - Calibration, this is optional and chip specific.

Clock sources for SysTick:
- STK_CSR_CLKSOURCE_AHB - CPU Clock
- STK_CSR_CLKSOURCE_AHB_DIV8 - CPU Clock / 8.

Openlibcm3 provides a HAL in libopencm3/includes/libopencm3/cm3/systick.h which contains the following function prototypes to set systick hardware:

- void systick_set_reload(uint32_t value);
- bool systick_set_frequency(uint32_t freq, uint32_t ahb);
- uint32_t systick_get_reload(void);
- uint32_t systick_get_value(void);
- void systick_set_clocksource(uint8_t clocksource);
- void systick_interrupt_enable(void);
- void systick_interrupt_disable(void);
- void systick_counter_enable(void);
- void systick_counter_disable(void);
- uint8_t systick_get_countflag(void);
- void systick_clear(void);
- uint32_t systick_get_calib(void);




<script src="https://utteranc.es/client.js"
        repo="skoopsy/skoopsy.github.io"
        issue-term="pathname"
        label="blog-embedded1"
        theme="preferred-color-scheme"
        crossorigin="anonymous"
        async>
</script>

Copywrite © 2025 Skoopsy
