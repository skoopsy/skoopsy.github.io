---
layout: post
title:  "STM32 #4: The interupt counter on Arm Cortex M with Libopencm3 & SysTick "
date:   2025-06-30 23:03:00 +0000
categories: STM32
---

In this post we will progress from using the assembler code __asm__("nop") to using a interrupt based delay via the libopencm3 library on the Bluepill STM32F1 board. See [STM32: From Template to Blinky - Building Bluepill Firmware with libopencm3](https://skoopsy.dev/stm32/2025/06/29/STM32-3-blinky-bluepill.html) for how to get a blinky running with libopencm3 on a bluepill using make. However, this code should work for any of the chips supported by libopencm3.

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

Libopencm3 has a systick module. This accesses a 24-bit timer which is built into every Cortex M core (from ARM themselves, not STM32), it is typically used for periodic interrupts, system time keeping, and delays.

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

The [libopencm3 docs for systick](https://libopencm3.org/docs/latest/stm32f1/html/group__CM3__systick__file.html#ga2604630453d0b6b35601375d0ee7e4a0)

So lets mkae a slighly more professional version of our blinky which uses a timer rather than raw clock cycles.

```c
#include <libopencm3/stm32/rcc.h>
#include <libopencm3/stm32/gpio.h>
#include <libopencm3/cm3/systick.h> // Notice it is not stm32 specific

#include <stdint.h> // Standard library for delaring int32

#define LED_PORT GPIOB // set the port in a more flexible fashion
#define LED_PIN GPIO2 // same with the pin
```

Now lets create a 32bit usinged int to hold a millisecond value we will count from system startup, about 49 days worth of counting before resetting. It's important to set this as voltaile as it will be modified by sus_tick_handler(), not inside main(). Without volatile declaration the compiler might opimise out reads/writes thinking the value is unused:

```c
static volatile uint32_t system_millis = 0;
```

Now lets write a small handler for the interups from systick to increment the system_millis value:

```c
void sys_tick_handler(void){
    system_millis++;
}
```

Let's create a sleep function, this is a "busy-waiting" approach and so blocks the CPU from doing anything else whilst the delay occurs, although ISRs still run so it can respond to interrupts.

```
void msleep(uint32_t ms) {
    uint32_t start = system_millis;
    while ((system_millis - start) < ms);
}
```

Now we need to setup SysTick to interupt every 1ms, in our case we have a 72Mhz system clock

```
void systick_setup(void) {
    systick_set_clocksource(STK_CSR_CLKSOURCE_AHB);
    systick_set_reload(72000 - 1) // 72,000 ticks for 1ms at 72 MHz
    systick_clear();
    systick_interrupt_enable();
    systick_counter_enable();
}
```

Enable the peripheral clocks and configure GPIO pins

```
void gpio_setup(void) {
    rcc_periph_clock_enable(RCC_GPIOB);
    gpio_set_mode(LED_PORT,
                  GPIO_MODE_OUTPUT_2_MHZ,
                  GPIO_CNF_OUTPUT_PUSHPULL,
                  LED_PIN);
}
```

Make sure the system clock is set to 72 MHz

```
void clock_setup(void) {
    rcc_clock_setup_pll(&rcc_hse_configs[RCC_CLOCK_HSE8_72MHZ]);
}

```

Our main loop is now very clean an maintainable:

```
int main(void) {
    clock_setup();
    gpio_setup();
    systick_setup();

    while (1) {
        gpio_toggle(LED_PORT, LED_PIN);
        msleep(500);
    }
}
```

Now make sure to go back to the top of the file and declare your function protoypes so that the compiler does not compain:

```
void systick_setup(void);
void gpio_setup(void);
void clock_setup(void);
void msleep(uint32_t ms);
void sys_tick_handler(void);
```
Alternatively you could delcare your functions as static and it would also chill out.


Save and quit the file, run make, and flash that bad boi using an ST-LINK V2 with:
```
st-flash write blinky.bin 0x8000000
```

A nice slow blinking appears, and the beast becomes calm.

You now have a fairly approach using thin layers from Libopencm3 for precise control of your hardware to make a silly led blink. The timing should be fairly precise with this method, but we have implemented a blocking function so the CPU is held up whilst waiting on the delay. In the next post we'll cover a Non-blocking version.

Incase you messed up; here's the full main.c file:
```c
#include <libopencm3/stm32/rcc.h>
#include <libopencm3/stm32/gpio.h>
#include <libopencm3/cm3/systick.h>

#include <stdint.h>

// Function prototypes
void systick_setup(void);
void gpio_setup(void);
void clock_setup(void);
void msleep(uint32_t ms);
void sys_tick_handler(void);

// Define LED port and pin
#define LED_PORT GPIOB
#define LED_PIN GPIO2

// Set a varaible to hold the millisecond count
static volatile uint32_t system_millis = 0;

void systickhandler(void);

// A mini handler to increment the system_millis counter
void sys_tick_handler(void) {
	system_millis++;
}

// Simple sleep function in milliseconds
void msleep(uint32_t ms) {
	uint32_t start = system_millis;
	while ((system_millis - start) < ms);
}

// Setup SysTick to interrupt every 1 ms
void systick_setup(void) {
	systick_set_clocksource(STK_CSR_CLKSOURCE_AHB);
	systick_set_reload(72000 -1); // 72,000 ticker for 1ms at 72 MHz
	systick_clear();
	systick_interrupt_enable();
	systick_counter_enable();
}

// Enable peripheral clocks and initialise GPIO pin
void gpio_setup(void) {
	rcc_periph_clock_enable(RCC_GPIOB);
	gpio_set_mode(LED_PORT,
		      GPIO_MODE_OUTPUT_2_MHZ,
		      GPIO_CNF_OUTPUT_PUSHPULL,
		      LED_PIN);
}

// Set system clock to 72 Mhz
void clock_setup(void) {
	rcc_clock_setup_pll(&rcc_hse_configs[RCC_CLOCK_HSE8_72MHZ]);	
}

// Main
int main(void) {
	clock_setup();
	gpio_setup();
	systick_setup();

	while (1) {
		gpio_toggle(LED_PORT, LED_PIN);
		msleep(500); // 500ms delay
	}
}

```
<script src="https://utteranc.es/client.js"
        repo="skoopsy/skoopsy.github.io"
        issue-term="pathname"
        label="blog-embedded1"
        theme="preferred-color-scheme"
        crossorigin="anonymous"
        async>
</script>

Copywrite © 2025 Skoopsy
