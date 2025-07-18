---
layout: post
title:  "STM32 #1 Getting Started with Embedded"
date:   2025-03-14 00:00:00 +0000
categories: STM32
---

Some of the stuff here may be wrong, and please feel free to let me know as I'm learning, and this is my account of it!

If you aren't fussed about the inner workings of the devices and just want to get some sensor outputs then I suggest the Arduino ecosystem along with the Arduino IDE. For more in-depth interests I hope you find this amusing.

This post will be a birds eye view of some of the key components of embedded systems from my best understanding, as many tutorials online follow one of the specific paths, often with little explanation of the context. Whilst I had some background in electronics I found the landscape vast when looking at where to begin with the STM32, so each section here fills out a question I had.


# Embedded systems

There are a few classes of embedded systems, but here we will focus on real-time embedded systems. These systems are designed to perform specific tasks within strict time constraints - meaning they must respond to inputs or events within a pre-defined time frame. Examples of real-time embedded systems include smartwatches, automotive control systems, and industrial robots.

They typically consists of these key components:

- **Processor (CPU)**: The brain of the system, responsible for executing instructions.

- **Memory**: Used for storing data, and code instructions for the system. This may be both volatile (RAM), and non-volatile memory (e.g. flash)

- **Input/Output (IO) units**: Interfaces that allow the system to communicate with the outside world such as; sensors, displays, buttons, and communication modules (like bluetooth and wifi).

- **Power management**: Embedded systems are often designed to be power efficient, as they are often run on small batteries constrained by weight, space, and cost.

- **Peripherals**: Additional components on the system that increase functionality such as; sensors, actuators, and communication interfaces.

# Development Boards, MCUs, SoCs, and Processing Cores.

A **processing core** does the main number crunching, for us this is the thing when I mention the [ARM Cortex](https://en.wikipedia.org/wiki/ARM_architecture_family) architecture. ARM doesn't make chips... yet, they are mostly a designer who licenses out IP to manufacturers.

A **Micro Controller Unit (MCU)** is a processor with a single integrated circuit with a processor core, memory, interfaces, counters and a few other things. They are typically dedicated to small control systems, hence the name.

A **System on Chip (SoC)** is a bit more of a wavey term, it is sort of anything more complicated than an MCU, maybe something with multiple MCUs on one chip! It might include modules for digital signal processing, extra memory, extra processing cores, and maybe sensors like accelerometers. SoCs are becoming more and more common when talking about embedded systems, so the terms regularly appear and are often used interchangeably!

Manufacturers make it very easy to build and experiment with their MCU/SoC offerings by creating **development boards**. These boards come with everything needed to get a system running around their chip. Dev boards make it very easy to get started by serving as a sandbox for your ideas - allowing you to prototype, test, and modify an embedded system without having to design everything from scratch and deal with manufacturing and assembling a PCB!

Some examples of these are:

- [Arduino Uno](https://docs.arduino.cc/hardware/uno-rev3/) - This is a development board for the ATMega 328P MCU and a common introduction to micro controllers. It's not an STM32 chip though, it's a [AVR](https://en.wikipedia.org/wiki/AVR_microcontrollers) architecture made by ATMega

- [Raspberry Pi Pico](https://www.raspberrypi.com/products/raspberry-pi-pico/) - This is a development board for the RP2040 microcontroller chip designed by the Raspberry Pi foundation and based on the [ARM Cortex](https://en.wikipedia.org/wiki/ARM_architecture_family) architecture.

- [Espressif ESP32-WROOM](https://www.espressif.com/en/products/socs/esp32) - This is a development board for the Espressif ESP32 chip which packs a punch for the price with WiFi and Bluetooth at a very low cost. Again, not a STM32 chip, is based on Espressifs flavour of the [Xtensa LXn](https://www.cadence.com/en_US/home/tools/silicon-solutions/compute-ip/tensilica-xtensa-controllers-and-extensible-processors/xtensa-lx-processor-platform.html) architecture 

- [ST Nucleo M-Bed L152RE](https://os.mbed.com/platforms/ST-Nucleo-L152RE/) - This IS an STM32 development board and uses the [ARM Cortex](https://en.wikipedia.org/wiki/ARM_architecture_family) architecture. It also comes with the ST-LINK/V2-1 debugger/programmer hardware attached. The ARM architecture is widely used in embedded IoT applications including some Laptops and Phones. **I will initially be using this board**.

- [Black Pill](https://stm32-base.org/boards/STM32F103C8T6-Black-Pill) - This is based on a STM32F1 chip and is a great option if you are strapped for cash. You can pick these up from Aliexpress very cheaply, you will also need the ST-Link v2 USB adapter to be able to flash code and debug on it. If this interests you also check out it's more famous predecessor - the Blue Pill.

# STM32

To be clear; ST Microelectronics make chips and sell them. ARM "design" processing cores/SoCs and sell the intellectual properties to companies like ST Microelectronics (for now). So there are many flavours of ARM from different manufacturers.

I'm interested in the STM32 chips because they are widely available, offer cheap development boards, have reasonably good platforms for learning on, and the designs can be scaled up to professional level applications. It seems like a solid approach to learning the Arm Cortex architecture.

Here is a list of their most relevant IoT focused chip families based on the Cortex-M series architecture:

- STM32**F0** - Arm Cortex M0
- STM32**F1** - Arm Cortex M3
- STM32**F7** - Arm Cortex M7
- STM32**H5** - Arm Cortex M33
- STM32**N6** - Arm Cortex M55

The M33 and M55 are much newer chips and are what I will aim to use in future and will be replacing the M3, M4, and M5 systems likely. There are many variants within the lists above, these are the codes appended after the STM32XX code, they typically refer to things like the amount of flash, clock speed, and additional modules/capabilities such as floating point units and dsp modules.

Here is a [rundown from STMicroelectronics](https://www.st.com/en/microcontrollers-microprocessors/stm32-high-performance-mcus.html) on their offerings.

# The Docs

I asked a question about getting started in hacker news and was given some help, here is a distilled version of the comment~~s~~ I liked, including my additional tangents, thoughts and research. This comment, from who I'll call "The Alchemist" gave some gold: Manuals. There are different levels of manuals for embedded documentation, here are some of them:

(You can find them by poking around the [ST site for the L152RE](https://www.st.com/en/microcontrollers-microprocessors/stm32l152re.html#documentation))

- **Development Board Datasheet**
    
    - External pin outs, on board peripherals, and general spec of the board. 
    - I am using a [ST Nucleo L152RE (MB1136)](https://www.st.com/en/evaluation-tools/nucleo-l152re.html).


- **Micro Controller Unit Product Specification Sheet**

    - Pin outs, on chip peripherals, and general specs of the MCU.
    -  The ST Nucleo L152RE uses a [Cortex M3 MCU (STM32L152RE)](https://www.st.com/en/microcontrollers-microprocessors/stm32l152re.html), find the product specification there.


- **Reference Manual**
    
    - Provides information for application level software. This is a beast of a document (>900 pages) giving details on the MCU family, the alchemist advises using this as your guide to programming and peripheral capabilities. 
    - Manual for the board I am using [RM0038 Reference Manual - Direct download](https://www.st.com/resource/en/reference_manual/cd00240193-stm32l100xx-stm32l151xx-stm32l152xx-and-stm32l162xx-advanced-arm-based-32-bit-mcus-stmicroelectronics.pdf), here is a link to the [preceding documentation page](https://www.st.com/en/microcontrollers-microprocessors/stm32l151-152/documentation.html) if direct download links are insulting.


- **Programming Manual**

    - Provides information for application and system-level software
developers - It gives a full description of the processor programming model, instruction set and core peripherals
    - [Programming Manual PM0056 - Direct download](https://www.st.com/resource/en/programming_manual/pm0056-stm32f10xxx20xxx21xxxl1xxxx-cortexm3-programming-manual-stmicroelectronics.pdf).


- **Application Notes**

    - A whole bunch of documents going into specific areas of interest
    - There are 38 application notes for the [L152RE on the ST site](https://www.st.com/en/microcontrollers-microprocessors/stm32l152re.html#documentation), some examples include; Digital signal processing, clock configuration, and getting started with MCUs in the STM32CubeIDE.


I'm sure if you are smart enough you can sit there like a stoic absorbing all these documents and then never ask for help again. 

_"You could leave life right now. Let that determine what you do and say and think"_ - Marcus Auralius

And so we push forward, but these will come in handy as I build motivation to read them...

# Languages 

So embedded is typically C right? Well C++ seems to be growing which makes sense because you can compile C with a C++ compiler and memory is getting bigger on chips these days. There is also the 'new' kid on the block - Rust. I'm going to stick with C... for now, but Rust is very interesting and potentially the future, although the advice right now seems to be that the Libraries for Rust are not quite there yet, so if you are keen with data sheets and building things from scratch; maybe that's the way to go. Some people talk about micro Python and Java but I think I just threw up a bit in my mouth so need to go wash that out.

Whilst I'm cleaning out the sink here's a list of embedded languages from [geekstogeeks](https://www.geeksforgeeks.org/embedded-systems-programming-languages/).

# Toolchains
A toolchain is a set of software development tools that are integrated to build software for a specific architecture. Toolchains help automate and streamline converting source code into executable programs or in our case firmware. These often have tools for compiling, assembling, linking, and debugging code, and it is tailored for a specific environment such as embedded ARM systems.

The main parts of a toolchain:

- **Compiler**: Translate source code (C, C++) into machine code or intermediate code. Some popular examples are; GCC (GNU Compiler Collection), and Clang.

- **Assembler**: Converts assembly language code into machine code for a specific architecture, for example; GNU Assembler (binutils)

- **Linker**: It's in the name, it links object files from the compiler/assembler together into a single executable or firmware image. E.g: GNU Linker (ld).

- **Debugger**: Help you debug code allowing you to step through execution, inspect variables, and find errors! E.g. GDB (GNU Debugger).

- **Libraries**: Libraries for the specific architectures may often be bundled with a tool chain, along with some standard functionalities like math. E.g. Newlib.

- **Build System**: This automates compiling and linking code, typically using a script or via tools like Make and CMake.

## Arm Toolchains

There are two main streams of toolchains for Arm from my understanding: The free open-source one, and the not free, not open-source ones.

- [ARM GNU Toolchain](https://developer.arm.com/Tools%20and%20Software/GNU%20Toolchain) - This is the free open-source one
- ARMCC: The proprietary one does not seem clearly labelled to me, it has a proprietary arm compiler with additional support for Fortran and Assembly.

Arm seem to be going through a rebrand with their offerings and in April 2025 they will release everything under the naming [Arm Toolchain for Embedded](https://developer.arm.com/Tools%20and%20Software/Arm%20Toolchain%20for%20Embedded) which will consist of the following offerings:

- **Arm Toolchain** - "Source and build scripts in a Github repo..."
- **Arm Toolchain for Embedded** - "A pre-built and tested, free to use, 100% open source toolchain supported only by the open source community".
- **Arm Toolchain for Embedded Professional** - "Functionally identical to Arm Toolchain for Embedded but with additional features for professional developers...".
- **Arm Toolchain for Embedded FuSa** - "A safety-qualified toolchain for development of safety-related projects.". Expected 2026.

It is not clear to me if the Arm Embedded Toolchain will replace the Arm GNU Toolchain, or if it is something else. For now we will stick with the free open-source ARM GNU Toolchain.

Within these are different configurations depending on the context you are interested in, such as compiling for bare-metal embedded, or for a linux kernel.

General naming conventions for the tool chains are: arch-vendor-os-eabi

- **arch:** Architecture (ARM in our case!).
- **vendor:** The toolchain supplier.
- **os:** The target operating system (e.g. Linux on ARM).
- **eabi:** Embedded Application Binary Interface. (A good explanation of ABI can be found [here](https://stackoverflow.com/questions/8060174/what-are-the-purposes-of-the-arm-abi-and-eabi).

It's a little jumbled around for the toolchains we are interested in as you'll see below but I am interested in the bare-metal open source tool chain:

**gcc-arm-none-eabi**: GNU based GCC, targets the ARM architecture, has no vendor, no target os (bare-metal), and complies with the ARM embedded application binary interface (EABI). Open-source and widely used for bare-metal programming for hobbyists and industry.

I sometimes came across information referring to the now discontinued toolchain for arm embedded systems from the very similarly named [GNU Arm Embedded Toolchain](https://developer.arm.com/downloads/-/gnu-rm) (last release 2021) with file download naming conventions such as:

- gcc-arm-none-eabi-...

Whereas, the current (Q1 2025) [ARM GNU toolchain](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads) is named with a prepended part to them, and a removed gcc before the architecture identifier, such as:
- arm-gnu-toolchain-...x86_64_-arm-none-eabi.tar.xz

Just a little thing that sent me down a rabbit hole.

For a guide on installing the toolchain from arm themselves check [here](https://learn.arm.com/install-guides/gcc/arm-gnu/). I found the apt package for Ubuntu to be about 1 year out of date, but still went with it because I was too lazy to get the latests version from the Arm website, the guide shows you both ways on Linux, Mac, and Windows.

# IDEs
You could do everything in [Neovim](https://neovim.io/) if you wanted, but it might be easier whilst learning to use a system with pre-made automation tools to make the process a little smoother. The main IDEs that are mentioned for STM32 development are listed below:

- [Arm Keil MDK v6](https://www.keil.arm.com/) - This is the official IDE suite from ARM, there is a [free non-commercial license](https://www.keil.arm.com/keil-mdk/#mdk-v6-editions) for it but it does not support all Arm chips or features. It does include support for some RTOS such as CMSIS library and FreeRTOS, along with providing some of the proprietary arm toolchains such as the Arm compiler and a debugger. Also has a [VSCode extension](https://marketplace.visualstudio.com/items?itemName=Arm.keil-studio-pack).

- [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html) - The official IDE supplied by ST Microelectronics. It is based on Eclipse and uses the GNU arm tool chains, along with supporting the official ST Link debugger hardware. There is a bunch of support and packages with this. It seems like a great option to low the barrier to getting started with STM32 development.

- [PlatformIO IDE](https://platformio.org/) - Free and Open-Source! They supply a standalone IDE and also an [extension for vscode](https://platformio.org/install/ide?install=vscode). This seems fully featured enough to try so I will be using it for my initial progress.

- [Arm Mbed](https://os.mbed.com/) - Free and Open-source development platform - IDE and OS. However, this project is being [binned in 2026](https://os.mbed.com/blog/entry/Important-Update-on-Mbed/).

# Libraries
There are a bunch of libraries that can make life easier but it is always a balance between this and not having a clue what is actually going on at the bare-metal, along with having to check licenses of anything you grab. There are some main free libries of interest:

- [Common Microcontroller Software Interface Standard (CMSIS)](https://www.arm.com/technologies/cmsis) is a [Hardware Abstraction Layer (HAL)](https://www.embedded.com/do-you-need-your-own-hardware-abstraction-layer-hal/) provided by Arm for free, under Apache 2.0 license, with the goal of creating some standardisation between vendors and arm chips to decouple the hardware from the higher level logic.

- ST Microelectronics also provides their own [HAL](https://www.st.com/en/embedded-software/stm32cube-mcu-mpu-packages.html) along with some additional libraries.

# The fuzzy line between Bare-Metal and Real Time Operating Systems (RTOS)

"Bare-metal" is a pure form of interacting with the hardware, you are running instructions exactly as you ask, per clock cycle, in the order that you have told it - your code is running synchronously. You may only use bare-metal when you have too little resources to use an OS, or the task is so simple that the effort of setting up an OS is not worth it, or you just don't like the offerings out there for your application. Maybe you make your own "OS".

An RTOS handles task scheduling, may have some form of memory management, and some hardware abstraction layer libraries - it's like bare metal but you have a library handling some of the complex and core functionality for you. In simple applications you might allocate all the memory at the beginning, but if you need to be more dynamic then an RTOS can manage this. You probably want things like task scheduling and memory management to have been figured out by smart mathematicians rather than tackling that yourself for complex projects. An RTOS can also provide higher levels of abstraction, which can be useful depending on your needs. 

An RTOS can be treated as a library in most cases or a full blown OS in some, the way you use them generally is not very different than for bare metal from an implementation point of view.

Lastly an RTOS is quite different from the operating system you will find on a laptop or phone, which is known as a general purpose operating system (GPOS). The main point here is in the name, a RTOS considers the timing of tasks to be the top priority which is important in systems where you want to read sensors and do things with that in a timely manner. I found this to be a palatable guide on [RTOS](https://www.spiceworks.com/tech/hardware/articles/what-is-rtos/).

## Some RTOS offerings

[ChibiOS](https://www.chibios.org/dokuwiki/doku.php) was recommended to me by a friend at a local Hackerspace. It's a "Complete development environment for embedded applications" including a real time operating system (RTOS), hardware abstraction layer (HAL), peripheral drivers, support files, and tools. It's also open source, nice. Lastly, it has an IDE if you want to use it called [ChibiStudio](https://www.chibios.org/dokuwiki/doku.php?id=chibios:products:chibistudio:start) (Disclaimer: I have not tried it). ChibiOS seems like a fairly reasonable RTOS to learn the basics with.

Here are some notes I made when I tried to compile some code with ChibiOS on Ubuntu:

- gcc-arm-none-eabi (Compiler-Link-Library: Tool Chain) (Via apt-get install)
- Clone [ChibiOS github](https://github.com/chibios)
- Find demo example in repo
- Run "make" command to compile code from example dir, use this dir as a template

Here are some notes I made when I tried to flash to hardware:
- Run make flash or find a way to flash to the board - using STMicro software for for their specific dev board to flash compile firmware to the board.
- [Open Source version of STLink Tools](https://github.com/stlink-org/stlink#installation)

Some alternatives to ChibiOS:

- [FreeRTOS](https://www.freertos.org/) - Open Source, less abstracted than ChibiOS, commonly used in industry, I will try this out.

- [Mbed Iot OS](https://os.mbed.com/) - Open Source, designed for IoT incl. a full open source development platform and IDE, will be depreciated by Arm in 2026.

- [RTEMS](https://www.rtems.org/) - Open source, used by [ESA for safety-critical space applications](https://www.esa.int/Enabling_Support/Space_Engineering_Technology/Software_Systems_Engineering/RTEMS). This is probably not starter territory, if that needed to be said.

# Some Starting Resources:

- [STM Base Project](https://stm32-base.org/): They have a nice [Introduction to STM32](https://stm32-base.org/guides/getting-started) page with some introduction on the STM32 chip series, IDEs, and platforms you can use. It is not comprehensive, but I found it a useful start. They also have some getting started guides which I found useful - Tried a blinky example from them using VSCode, PlatformIO and a hardware abstraction layer library for the board. The most difficult bug I encountered was a faulty USB cable, that tiny 10cm imposter cable stole a good 1 hour of my life. So if it builds but doesn't upload, try another cable.

- [ST Microelectronics Wiki](https://wiki.st.com/stm32mcu/index.php?title=STM32StepByStep:Getting_started_with_STM32_:_STM32_step_by_step&oldid=10323):  I guess this is the official starting area for learning STM32, from the manufacturer resources themselves.


# Summary
So there is the underlying architecture (e.g. Arm Cortex M0), the manufacturers who make and implement devices (e.g. ST with the STM32F0 series), development boards that house a system on chip with GPIO pins broken out, power circuitary, bootloaders, and all the caps/resistors/diodes placed so that you can get started with a SoC (e.g. STnucleo, or the Bluepill / Blackpill).

Then we have the documentation to understand it all such as Dev Board data sheets, MCU Product Specification sheet, Reference Manual, Programming Manual, and Application notes.

Then we have the "toolchains" which help you develop and get code running on the SoCs such as the; Compiler, Assembler, Linker, Debugger, build system, and libraries (E.g. CMake and ARM GNU Toolchain).

IDEs /Editiors for writing code and automating toolchain use (e.g. PlatformIO in VSCode, or neovim for the raw approach)

Libraries such as CMSIS, manufacturers flavoured HAL (e.g. ST HAL for their chips).

Then real time operating system which are much like advanced libraries that handle a lot of the difficult timing, interup, and low level things (e.g. FreeRTOS)

So now, lets try a blinky!


<script src="https://utteranc.es/client.js"
        repo="skoopsy/skoopsy.github.io"
        issue-term="pathname"
        label="blog-embedded1"
        theme="preferred-color-scheme"
        crossorigin="anonymous"
        async>
</script>

Copywrite © 2025 Skoopsy
