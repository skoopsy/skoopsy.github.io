---
layout: post
title:  "STM32 #5: Non-Blocking delay with Libopencm3"
date:   2025-06-30 23:00:00 +0000
categories: STM32
---
So the last post we used Systick to create a delay in the code, and a custom function which made the main loop hang around in a while loop until the delay was over. This would be considered a blocking function as it stops main from doing anything else while it waits for the delay, but we have loads of processing power, so lets do some other shit inbetween toggling the led!

The last example goes through the code in detail, so I will only be discussing the improvements here:

```c
#include <libopencm3/stm32/rcc.h>
#include <libopencm3/stm32/gpio.h>
#include <libopencm3/cm3/systick.h>

#include <stdint.h>

// Define LED port and pin
#define LED_PORT GPIOB
#define LED_PIN GPIO2

// Set a varaible to hold the millisecond count
static volatile uint32_t system_millis = 0;

// Function prototypes
void systick_setup(void);
void gpio_setup(void);
void clock_setup(void);
void sys_tick_handler(void);
void systickhandler(void);

// A mini handler to increment the system_millis counter
void sys_tick_handler(void) {
	system_millis++;
}

// Setup SysTick to interrupt every 1 ms
void systick_setup(void) {
	systick_set_clocksource(STK_CSR_CLKSOURCE_AHB);
	systick_set_reload(72000); // 72,000 ticker for 1ms at 72 MHz
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
```

So all that code is the same, you'll notice I have removed the msleep() function, we are going to build a function that combines the delay and the toggling of the LED in a non-blocking manner

first at the top of the file after decalring the system_millis variable let us delare a new struct that can hold information on the led blink:

```c
// define some variables
typedef struct {
    uint32_t last_toggle;
    uint32_t interval_ms;
    uint32_t port;
    uint32_t pin;
} blink_t;
```

Now add the following function to perform the blink using keys from the struct

```
// Toggle the led blink frequency
void blink_ticker(blink_t *b, uint32_t now) {
	if ((now - b->last_toggle) >= b->interval_ms) {
        gpio_toggle(b->port, b->pin);
        b->last_toggle = now;
    }
}

```
and add the ```void blink_ticker(blink_t *b, uint32_t now);``` to the prototypes at the top.

Now the main loop will look like:

```c
// Main
int main(void) {
	clock_setup();
	gpio_setup();
	systick_setup();
    
    blink_t led_pb2 = {
        .last_toggle = 0,
        .interval_ms = 1000,
        .port = GPIOB,
        .pin = GPIO2
    };

	while (1) {
	    blink_ticker(&led_pb2, system_millis);

        // Free to do other stuff here now!
         
    };
}
```

Save and quit, then run ```st-flash write blinky.bin 0x8000000``` via the ST-LINK V2 plugged into the blue pill and you should now have an even slower blinky, that is somuch more capable.




<script src="https://utteranc.es/client.js"
        repo="skoopsy/skoopsy.github.io"
        issue-term="pathname"
        label="blog-embedded1"
        theme="preferred-color-scheme"
        crossorigin="anonymous"
        async>
</script>

Copywrite © 2025 Skoopsy
