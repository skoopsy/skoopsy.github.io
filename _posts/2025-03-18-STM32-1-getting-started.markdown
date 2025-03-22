---
layout: post
title:  "STM32 #1 Getting Started with Embedded"
date:   2025-03-14 00:00:00 +0000
categories: STM32
---

**Disclaimer: This is an account of my learning journey with embedded electronics and the STM32 - There will likely be mistakes, inefficiencies and bad approaches!.**

When focusing on things more powerful than the AVR arduino system (such as STM32) I can only find a few short tutorials on this subject, or a very complicated mix of tutorials that do not seem structured for a beginner. The alternatives are video courses (which are painfully slow and lacking IMO), or reading books which are often outdated. So here I will log my journey trying to learn embedded electronics with the STM32 platform. My end goal is to build a smart watch, but will set much more incremental and achievable goals along the path, who knows if I'll get there but there will be a bunch of fun trying. Some of the stuff here may be wrong, and please feel free to let me know!

Maybe you've done some arduino projects, a C++ course, or you've spent some time around a hackerspace and got inspired by the awesome things being built around you with these things called a Micro Controller Unit (MCU)! It's not my first rodeo, but I'm also not a seasonsed firmware or electrical engineer (full disclosure: I do have a PhD in materials and printed electronics, but that was much more to do with moving electrons and atoms around, rather than flipping bits - Which is either a million miles away or right next door, depending on your background maybe). If you just want to quickly make a system which logs some sensor data somewhere and you aren't too fussed about the inner workings then I suggest the Arduino ecosystem along with the Arduino IDE. For more indepth interests I hope you find this amusing.

I asked a question about getting started in hacker news and was given some help, here is a distilled verson of the comment~~s~~ I liked, including my additional tangents and thoughts:

# Embedded systems

There are a few classes of embedded systems, but here we will focus on real-time embedded systems. These sysems are designed to perform specific tasks within strict time constraints - meaning they must respond to inputs or events within a pre-defined time frame. Examples of real-time embedded systems include smartwatches, automotive control systems, and industrial robots.

A real-time embedded system typically consists of these key components:

- Processor (CPU): The brain of the system, responsible for executing instructions.

- Memory: Used for storing data, and code instructions for the system. This may be both volatile (RAM), and non-volitile memory (e.g. flash)

- Input/Output (IO) units: Interfaces that allow the system to communicate with the outside world such as; sensors, displays, buttons, and communication modules (like bluetooth and wifi).

- Power management: Embedded systems are often designed to be power efficient, as they are often run on small batteries constrained by weight, space, and cost.

- Peripherals: Additional comonents on the system that increase functionality such as; sensors, actuators, and communication interfaces.

# Development Boards, MCUs and SoCs

Manufacturers make it very easy to build and experiment with their embedded system offerings by creating development boards. These boards come with everything needed to get a system running based of a System on Chip (SoC) that the manufactuer offers. A System on Chip is a single chip that integrates the processor, memory, I/O interfaces, and any other essential components all in one neat package. SoCs simplify development and make it easy to attach sensors, memory, and communication peripherals. 

A development board is build around a SoC and makes it very easy to get started by serving as a sandbox for your ideas - allowing you to prototype, test, and modify an embedded system without having to design everything from scratch and deal with manufacturing and assembling a PCB!

Some examples of these are:

- [Arduino Uno](https://docs.arduino.cc/hardware/uno-rev3/) - This is a development board for the ATMega 328P MCU and a common introduction to micro controllers. It's not an STM32 chip though, it's a [AVR](https://en.wikipedia.org/wiki/AVR_microcontrollers) architecture made by ATMega

- [Raspberry Pi Pico](https://www.raspberrypi.com/products/raspberry-pi-pico/) - This is a development board for the RP2040 microcontroller chip designed by the Raspberry Pi foundation and based on the [ARM Cortex](https://en.wikipedia.org/wiki/ARM_architecture_family) architecture.

- [Espressif ESP32-WROOM](https://www.espressif.com/en/products/socs/esp32) - This is a development board for the Espressif ESP32 chip which packs a punch for the price with WiFi and Bluetooth at a very low cost. Again, not a STM32 chip, is based on Espressifs flavour of the [Xtensa LXn](https://www.cadence.com/en_US/home/tools/silicon-solutions/compute-ip/tensilica-xtensa-controllers-and-extensible-processors/xtensa-lx-processor-platform.html) architecture 

- [ST Nucleo M-Bed L152RE](https://os.mbed.com/platforms/ST-Nucleo-L152RE/) - This IS an STM32 development board and uses the [ARM Cortex architecture](https://en.wikipedia.org/wiki/ARM_architecture_family). It also comes with the ST-LINK/V2-1 debugger/programmer hardware attached. The ARM architecture is widely used in embedded IoT applications including some Laptops and Phones. I will initially be using this board.
here are many codes and types, here is an overview them:

# STM32
To be clear ST Microelectronics make chips and sell them. ARM "design" chip architectures and sell the intellectual properties to companies like ST Microelectronics. So there are many flavours of ARM from different manaufacturers.

I'm interested in the STM32 chips because they are widely available, offer cheap development boards, have reasonably good platforms for learning on, and the designs can be scaled up to professional level applications. It seems like a solid approach to learning the Arm Cortex Architecture.

Here is a rundown on their main chip families:

- STM32F0 - Arm Cortex M0
- STM32F1 - Arm Cortex M3s
- STM32F3 - Arm Cortex M4
- STM32F7 - Arm Cortex M7
- STM32H5 - Arm Cortex M33
- STM32N6 - Arm Cortex M55

The M33 and M55 are much newer chips and are what I will be aiming to use in future projects and will proably replace the M0, M3, and M4 systems in the future. There are many varients within the lists above, these are the codes appended at the end of the STM32 code, they typically refer to things like the amount of onboard flash, clock speed, and additional capabilities such as floating point units and dsp modules.

Here is a [rundown from STMicroelectronics](https://www.st.com/en/microcontrollers-microprocessors/stm32-high-performance-mcus.html) on their offerings.

# The Docs

This comment, from who I'll call "The Alchemist" gave more introductory gold: Manuals. There are different levels of manuals for embedded documentation, here are some of them:

(You can find them by poking around the [ST site for the L152RE](https://www.st.com/en/microcontrollers-microprocessors/stm32l152re.html#documentation))

- Development Board Datasheet
    
    - External pin outs, on board peripherals, and general spec of the board. 
    - I am using a [ST Nucleo L152RE (MB1136)](https://www.st.com/en/evaluation-tools/nucleo-l152re.html).


- Micro Controller Unit Product Specification Sheet

    - Pin outs, on chip peripherals, and general specs of the MCU.
    -  The ST Nucleo L152RE uses a [Cortex M3 MCU (STM32L152RE)](https://www.st.com/en/microcontrollers-microprocessors/stm32l152re.html), find the product specification there.


- Reference Manual
    
    - Provides information for application level software. This is a beast of a document (>900 pages) giving details on the MCU family, the alchemist advises using this as your guide to programming and pheripheral capabilities. 
    - Manual for the board I am using [RM0038 Reference Manual - Direct download](https://www.st.com/resource/en/reference_manual/cd00240193-stm32l100xx-stm32l151xx-stm32l152xx-and-stm32l162xx-advanced-arm-based-32-bit-mcus-stmicroelectronics.pdf), here is a link to the [preceeding documentation page](https://www.st.com/en/microcontrollers-microprocessors/stm32l151-152/documentation.html) if direct download links are insulting.


- Programming Manual

    - Provides information for application and system-level software
developers - It gives a full description of the processor programming model, instruction set and core peripherals
    - [Programming Manual PM0056 - Direct download](https://www.st.com/resource/en/programming_manual/pm0056-stm32f10xxx20xxx21xxxl1xxxx-cortexm3-programming-manual-stmicroelectronics.pdf).


- Application Notes

    - A whole bunch of documents going into specifc areas of interest
    - There are 38 application notes for the [L152RE on the ST site](https://www.st.com/en/microcontrollers-microprocessors/stm32l152re.html#documentation), some examples include; Digital signal processing, clock configuration, and getting started with MCUs in the STM32CubeIDE.


I'm sure if you are smart enough you can sit there like a stoic absorbing all these documents and then never ask for help again. 

_"You could leave life right now. Let that determine what you do and say and think"_ - Marcus Auralius

And so we push forward, but these will come in handy as I build motivation to read them...

# Some Starting Resources:

- [STM Base Project](https://stm32-base.org/): They have a nice [Introduction to STM32](https://stm32-base.org/guides/getting-started) page with some introduction on the STM32 chip series, IDEs, and platforms you can use. It is not comprehensive, but I found it a useful start. They also have some getting started guides which I found useful - Tried a blinky example from them using VSCode, PlatformIO and a hardware abstration layer library for the board. The most difficult bug I encountered was a faulty USB cable, that tiny 10cm imposter cable stole a good 1 hour of my life. So if it builds but doesn't upload, try another cable.

- [ST Microelectronics Wiki](https://wiki.st.com/stm32mcu/index.php?title=STM32StepByStep:Getting_started_with_STM32_:_STM32_step_by_step&oldid=10323):  I guess this is the official starting area for learning STM32, from the manufacturer resources themselves.

# Languages 

So embedded is typically C right? Well C++ seems to be growing which makes sense because you can compile C with a C++ compiler and memory is getting bigger on chips these days. There is also the 'new' kid on the block - Rust. I'm going to stick with C... for now, but Rust is very interesting and potentially the future, although the advice right seems to be that the Libraries for Rust are not quite there yet, so if you are keen with data sheets and building things from scratch maybe that's the way to go. Some people talk about micro Python and Java but I think I just threw up a bit in my mouth so need to go wash that out.

Here's a list of embedded languages from [geekstogeeks](https://www.geeksforgeeks.org/embedded-systems-programming-languages/) from people who are probably much more worth listening to than me.

# The fuzzy line between Bare Metal and Real Time Operating Systems (RTOS)

"Bare metal" is a pure form of interacting with the hardware, you are running instructions exactly as you ask per clock cycle in the order that you have told it - your code is running synchronously. You may use bare metal when you have too little resources to use an OS, or the task is so simple that the effort of setting up an OS is not worth it. 

An RTOS handles task schedualing, may have some form of memory management, and some hardware abstraction layer libraries - it's like baremetal but you have a library handling some of the complex and core functionality for you. In simple applications you might allocate all the memory at the beginning, but if you need to be more dynamic then an RTOS can manage this. You probably want things like task schedualing and memory management to have been figured out by smart mathematicians rather than tackling that yourself for complex projects. An RTOS can also provide higher levels of abstraction, which can be useful depending on your needs. 

An RTOS can be treated as a library, the way you use them is not very different than for bare metal from an implementation point of view.

Lastly an RTOS is quite different from the operating system you will find on a laptop or phone, which is known as a general purpose operating system (GPOS). The main point here is in the name, a RTOS considers the timing of tasks to be the top priority which is important in systems where you want to read sensors and do things with that in a timely manner. I found this to be a palletable guide on [RTOS](https://www.spiceworks.com/tech/hardware/articles/what-is-rtos/).

## Some RTOS offerings

[ChibiOS](https://www.chibios.org/dokuwiki/doku.php) was recommended to me by a friend at a local hackerspace. It's a "Complete development environment for embedded applications" including a real time operating system (RTOS), hardware abstraction layer (HAL), peripheral drivers, support files, and tools. It's also open source, nice. Lastly, it has an IDE if you want to use it called [ChibiStudio](https://www.chibios.org/dokuwiki/doku.php?id=chibios:products:chibistudio:start) (Disclaimer: I have not tried it). ChibiOS seems like a fairly reasonable RTOS to learn the basics with.

Here are some notes I made when I tried to compile some code with ChibiOS on Ubuntu:

- gcc-arm-none-eabi (Compiler-Link-Library: Tool Chain) (Via apt-get install)
- Clone [ChibiOS github](https://github.com/chibios)
- Find demo example in repo
- Run "make" command to compile code from example dir, use this dir as a template

Here are some notes I made when I tried to flash to hardware:
- Run make flash or find a way to flash to the board - using stmicro software for for their specific dev board to flash compile firmware to the board.
- [Open Source version of STLink Tools](https://github.com/stlink-org/stlink#installation)

Some alternatives to ChibiOS:

- [FreeRTOS](https://www.freertos.org/) - Open Source, less abstracted than ChibiOS, commonly used in industry.
- [Mbed Iot OS](https://os.mbed.com/) - Open Source, designed for IoT incl. a full open source development platform and IDE.
- [RTEMS](https://www.rtems.org/) - Open source, used by [ESA for safety-critical space applications](https://www.esa.int/Enabling_Support/Space_Engineering_Technology/Software_Systems_Engineering/RTEMS). This is probably not starter territory, if that needed to be said.

# Conclusion
This seems to be the basic landscape for getting started with embedded STM32 systems. From here I will get all the dependencies installed, play around with some IDEs or use VIM, get a blinky going on the nucleo board, and then begin expanding on that.

<script src="https://utteranc.es/client.js"
        repo="skoopsy/skoopsy.github.io"
        issue-term="pathname"
        label="blog-embedded1"
        theme="preferred-color-scheme"
        crossorigin="anonymous"
        async>
</script>

Copywrite © 2025 Skoopsy
