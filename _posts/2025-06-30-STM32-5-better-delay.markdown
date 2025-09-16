---
layout: post
title:  "STM32 #5: Non-Blocking delay blinky with Bluepill and Libopencm3 using SysTick"
date:   2025-06-30 20:00:00 +0000
categories: STM32
---
Board: WeActStudio BluePill Plus STM32 F103CBT8 (Arm Cortex M3)

The [last post](https://skoopsy.dev/stm32/2025/06/29/STM32-4-bluepill-delay.html) employed Systick to create a timer, and then used that timer to create a delay in the code. The delay function developed, msleep(), caused the main loop to hang around in a while loop, until the delay was over. This is considered a blocking function - it stops main from doing anything else while it waits for the delay. There is a lot of processing power on the F103CBT8 and you might want to do some UART communication, sensor reading, or other stuff, so lets make it better.

These lines of code are mostly unchanged from the previous post, minus the removal of the ```msleep()``` function and some reshuffling:

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

// A mini handler to increment the system_millis counter
void sys_tick_handler(void) {
	system_millis++;
}

// Setup SysTick to interrupt every 1 ms
void systick_setup(void) {
	systick_set_clocksource(STK_CSR_CLKSOURCE_AHB);
	systick_set_reload(72000 - 1); // 72,000 ticker for 1ms at 72 MHz
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

We are going to build a function that implements the delay, and the toggling of the LED, in a non-blocking manner, packaged neatly and specifically for blinking the led.

At the top of the file, after declaring the ```system_millis``` variable, lets create some organisation by delaring a new struct that can hold information on the led blink parameters:

```c
// define some variables for led blink
typedef struct {
    uint32_t last_toggle;
    uint32_t interval_ms;
    uint32_t port;
    uint32_t pin;
} blink_t;
```

Now add the following function to perform the blink using keys from the struct:

- First we setup the function with a pointer ```b``` to a ```blink_t``` structure, passing it's address, and setup a variable to hold the current counter value.
- Access the ```last_toggle``` key in the structure pointed to by ```b``` by using the ->.
- Calculate how much time has passed since last toggle
- Check if the elapsed time is greater than, or equal to, the blinking interval set
- If it is: toggle the led.
- Update ```last_toggle``` to the current time to restart the count.

```c
// Toggle the led blink frequency
void blink_ticker(blink_t *b, uint32_t now) {
	if ((now - b->last_toggle) >= b->interval_ms) {
        gpio_toggle(b->port, b->pin);
        b->last_toggle = now;
    }
}

```
and add ```void blink_ticker(blink_t *b, uint32_t now);``` to the prototypes at the top.

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

Save, quit, then run ```st-flash write blinky.bin 0x8000000``` via the ST-LINK V2 plugged into the bluepill, and you should now have an even slower blinky!

What if you wanted to add another LED and have it blinking to a different beat? Well thats easy, you can define a second blink_t structure and run the blink_ticker function with that structure in the main loop!

Incase you messed up, or my explanation was poor... here's the ```main.c``` file in full:
```c
#include <libopencm3/stm32/rcc.h>
#include <libopencm3/stm32/gpio.h>
#include <libopencm3/cm3/systick.h>

#include <stdint.h>

// Define LED port and pin
#define LED_PORT GPIOB
#define LED_PIN GPIO2

// Set a variable to hold the millisecond count
static volatile uint32_t system_millis = 0;

// structure for the blink_ticker
typedef struct {
	uint32_t last_toggle;
	uint32_t interval_ms;
	uint32_t port;
	uint32_t pin;
} blink_t;

// Function prototypes
void systick_setup(void);
void gpio_setup(void);
void clock_setup(void);
void sys_tick_handler(void);
void blink_ticker(blink_t *b, uint32_t now);
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


// Toggle led blink timing
void blink_ticker(blink_t *b, uint32_t now) {
	if ((now - b->last_toggle) >= b->interval_ms) {
		gpio_toggle(b->port, b->pin);
		b->last_toggle = now;
	}
}

// Main
int main(void) {
	clock_setup();
	gpio_setup();
	systick_setup();

	// blink led setup
	blink_t led_pb2 = {
		.last_toggle = 0,
		.interval_ms = 1000,
		.port = GPIOB,
		.pin = GPIO2
	};

	while (1) {
		blink_ticker(&led_pb2, system_millis);

		// do other stuff here!
	};
}
```


# Interrupt based approach

The code above is a bit of a cop out though isn't it, it's a sort of "wait until timestamp, and check later" approach. It can be impacted by other things happening in the loop. So we could try a [Interrupt Service Routine](https://en.wikipedia.org/wiki/Interrupt) (ISR) which is ran outside of the main loop, freeing it up to do more useful things.

We will achieve this by letting the SysTick interrupt handle the LED toggling directly, every N milliseconds. The interrupt is hardware driven, it doesn't rely on software polling, so it will be less disruptive to our main loop.

First, modify the ```sys_tick_handler``` to increment a slightly more appropriately named variable, then create a conditonal statement to toggle the LED if the timer is at the blink interval:

```c
static volatile uint32_t blink_timer = 0;

// blink led setup structure
blink_t led_pb2 = {
    .interval_ms = 2000,
    .port = GPIOB,
    .pin = GPIO2
};

void sys_tick_handler(void) {
    blink_timer++;
    if (blink_timer >= led_pb2.interval_ms) {
        gpio_toggle(led_pb2.port, led_pb2.pin);
        blink_timer = 0;
    }
}
```
I have tweaked the blink_t structure slightly, by removing the now defunct last_toggle so you will have to go back and modify the declaration to match.

The blink_timer variable will be incremented everytime the Systick interrupt fires, which is based on the ```systick_setup()``` function defined before:

```c
// Setup SysTick to interrupt every 1 ms
void systick_setup(void) {
	systick_set_clocksource(STK_CSR_CLKSOURCE_AHB);
	systick_set_reload(72000 - 1); // 72,000 ticker for 1ms at 72 MHz
	systick_clear(); // Worth clearing here as it will be randomly set on reset of device.
	systick_interrupt_enable();
	systick_counter_enable();
}
```

The main loop simply loses the old delay logic:

```c
// Main
int main(void) {
	clock_setup();
	gpio_setup();
	systick_setup();

	while (1) {
		// do other timing critical stuff here!
	};
```

Now ```st-flash write blinky.bin 0x8000000``` with the ST-LINK V2 and there should be a 2 Hz blink.

Here is the full code, as the explanation of this one required a bit more chopping and changing:

```c

#include <libopencm3/stm32/rcc.h>
#include <libopencm3/stm32/gpio.h>
#include <libopencm3/cm3/systick.h>

#include <stdint.h>

static volatile uint32_t blink_timer = 0;

// structure for the blink_ticker
typedef struct {
	uint32_t interval_ms;
	uint32_t port;
	uint32_t pin;
} blink_t;

// Function prototypes
void systick_setup(void);
void gpio_setup(void);
void clock_setup(void);
void sys_tick_handler(void);

// blink led setup structure
const blink_t led_pb2 = {.interval_ms = 1000,
    			 .port = GPIOB,
    			 .pin = GPIO2
			 };

// New systick handler directly toggling LED
void sys_tick_handler(void) {
	blink_timer++;
	if (blink_timer >= led_pb2.interval_ms) {
		gpio_toggle(led_pb2.port, led_pb2.pin);
		blink_timer = 0;
	}
}

// Setup SysTick to interrupt every 1 ms
void systick_setup(void) {
	systick_set_clocksource(STK_CSR_CLKSOURCE_AHB);
	systick_set_reload(72000 - 1); // 72,000 ticker for 1ms at 72 MHz
	systick_clear(); // Worth clearing as unknown state at device reset
	systick_interrupt_enable();
	systick_counter_enable();
}

// Enable peripheral clocks and initialise GPIO pin
void gpio_setup(void) {
	rcc_periph_clock_enable(RCC_GPIOB);
	gpio_set_mode(led_pb2.port,
		      GPIO_MODE_OUTPUT_2_MHZ,
		      GPIO_CNF_OUTPUT_PUSHPULL,
		      led_pb2.pin
		      );
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
		// Do whatevery you likee!
	}
}

```

# A very low power approach (WFI)
Considering that we aren't doing anything in the main loop, and using a ISR to toggle the led, we could save some power by making the device go sleep until an interrupt wakes it.

Depending on the compiler you are using; you can access this functionality directly via the assembly command wfi (wait for interrupt) which is built into most ARM Cortex M chips. This halts the CPU clock between interrupts, reducing the pwoer consumption of the chip.

All we have to do is add it to the while loop in our main loop, here is the new main loop:

```c
// Main
int main(void) {
	clock_setup();
	gpio_setup();
	systick_setup();
	
	while (1) {
		__asm("wfi");
	}
}
```
Save, quit, then ```st-flash write blinky.bin 0x8000000``` and your done... 

Sorry about that one, it's a bit anticlimactic isn't it, nothing externally happens, but you hope a little less power is being used momentarily between systick interrupts. You could put a multimeter on the 3.3v line and you might see a difference if you slow the systick interrupt timer down, as it will sleep between each 1ms interrupt before waking up to incremenet the blink timer.

<script src="https://utteranc.es/client.js"
        repo="skoopsy/skoopsy.github.io"
        issue-term="pathname"
        label="blog-embedded1"
        theme="preferred-color-scheme"
        crossorigin="anonymous"
        async>
</script>

Copyright © 2025 Skoopsy
