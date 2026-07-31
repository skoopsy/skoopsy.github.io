---
layout: post
title:  "STM32 #18: Application Storage Layer using FatFs"
date:   2026-07-30 10:00:00 +0000
categories: STM32
---

In this post I'm going to add my own storage API on top of the FatFs layer that was tested in the last post. This will give the firmware a simple way to record events to a normal file on the SD card, which is a small but important step towards a useful data logger.

I plan to give the rest of the application one clear entry point for storage, rather than peppering FatFs calls all over the place. I want other parts of the code to be able to call things like `storage_log_event("Initialisation successful")` or, later, `storage_write_sensor_sample(&sample)` without knowing about `f_mount()`, `f_open()`, `f_write()`, file paths, or FatFs objects.

This post is really about stopping FatFs from leaking into the rest of the firmware.

Note: I do need to be careful here, SD card writes can block. FatFs may need to update file data, directory entries, FAT tables etc. The SD card itself can be busy internally writing. In the future the architecture might follow something like:

```
Sensor read -> sample to RAM buffer -> main loop calls storage_process() -> storage layer write chunks to SD card.
```

Files for the storage layer:

```
src/app/app_storage.c
include/app/app_storage.h
```

# Storage API

The requirements for this will be to:

1. Mount the filesystem once
2. Create a log file
3. Append log messages
4. Keep FatFs specifics out of main.c

Here is the app_storage.h:

```c
#pragma once

#include <stdint.h>

// Storage error codes
typedef enum {
        STORAGE_OK = 0,
        STORAGE_ERR_NOT_READY,
        STORAGE_ERR_PARAM,
        STORAGE_ERR_MOUNT,
        STORAGE_ERR_OPEN,
        STORAGE_ERR_WRITE,
        STORAGE_ERR_PARTIAL_WRITE,
        STORAGE_ERR_READ,
        STORAGE_ERR_CLOSE,
} storage_status_t;

// Public storage API
storage_status_t storage_init(void);
storage_status_t storage_log_event(const char *message);
```

The log file to be created will be called: ```0:/EVENT.LOG```.

I'm sticking with the 8.3 style filenames for FAT32 for now so that I don't have to add in long file name support making the FatFs memory burden larger.

# Storage implementation

I am going to reuse the tests developed in the last post to create the functions for app_storage.c so for a more detailed explanation of how I'm using the FatFs API check out my [last post.](https://skoopsy.dev/stm32/2026/07/29/STM32-18-using-fatfs-diskio-for-reading-and-writing-microsd.html)

Let's set up app_storage.c:
```c
#include "app/app_storage.h"
#include "bsp/bsp_usart.h"

#include "ff.h"
#include <stdbool.h>

// Drive and File path macros:
#define STORAGE_DRIVE_PATH "0:"
#define STORAGE_PATH_EVENT_LOG "0:/EVENT.LOG"

static FATFS storage_fs;                 // FatFs filesystem object
static bool storage_initialised = false; // Set flag after mounting
```

I'm keeping the filesystem object ```storage_fs``` static so that it is private to the module, but will stay for the lifetime of the program so that it can be used by any of the app_storage functions as and when required.

First we will build the initialisation function, which mounts the FatFs volume:

```c
storage_status_t storage_init(void) {
        FRESULT fr; // FatFs return status object
        
        // Mount the drive
        fr = f_mount(&storage_fs,
                     STORAGE_DRIVE_PATH,
                     1); // Mount drive immediately

        // Basic error handling
        if (fr != FR_OK) { 
                storage_initialised = false;
                return STORAGE_ERR_MOUNT;
        }
    
        // Success
        storage_initialised = true;
        return STORAGE_OK;
}
```

Then we can write a function to write event logs which is basically our write test from the last post but I'm using different f_open() modes this time - I want to open a file and if the file doesn't exist already then create it, set it to allow writing to the file, AND make sure to set the read/write pointer to the end of the file, so that new logs don't overwrite old logs. Here is the [list of f_open mode flags available]
(https://elm-chan.org/fsw/ff/doc/open.html):

| Flags                 | Meaning |
|-----------------------|---------|
| FA_READ	        | Specifies read access to the file. Data can be read from the file. |
| FA_WRITE	        | Specifies write access to the file. Data can be written to the file. Combine with FA_READ for read-write access. |
| FA_OPEN_EXISTING      | Opens the file. The function fails if the file is not existing. (Default) |
| FA_CREATE_ALWAYS      | Creates a new file. If the file is existing, the file is truncated and overwritten. |
| FA_CREATE_NEW         | Creates a new file. The function fails if the file is existing. |
| FA_OPEN_ALWAYS        | Opens the file. If it is not exist, a new file is created. |
| FA_OPEN_APPEND	| Same as FA_OPEN_ALWAYS except the read/write pointer is set end of the file. |

Oh and would you look at that there is are modes exactly for my needs: ```FA_OPEN_APPEND``` and ```FA_WRITE```, how wonderful.

Here is the function to write logs:

```c
#include <string.h>

storage_status_t storage_log_event(const char *message) {
        FRESULT fr;             //FatFs function return status object
        FIL event_file;         // FatFs file object
        UINT bytes_written = 0u;
        UINT bytes_to_write = 0u;

        // Check if the drive has been mounted:
        if (!storage_initialised) { return STORAGE_ERR_NOT_READY; }

        // Check if there is actually a message to write:
        if (message == NULL) { return STORAGE_ERR_PARAM; }

        // Open EVENT.LOG:
        fr = f_open(&event_file,
                    STORAGE_PATH_EVENT_LOG,
                    FA_WRITE | FA_OPEN_APPEND);
        if (fr != FR_OK) { return STORAGE_ERR_OPEN; }

        // Write message to EVENT.LOG
        bytes_to_write = (UINT)strlen(message);
        fr = f_write(&event_file,
                     message,
                     bytes_to_write,
                     &bytes_written);
        if (fr != FR_OK) {
                f_close(&event_file);
                return STORAGE_ERR_WRITE;
        }

        // You can also do a check here to see if it fully wrote the message or ran out of space and truncated the message by comparing bytes_written:
        if (bytes_to_write != bytes_written) {
                f_close(&event_file);
                return STORAGE_ERR_PARTIAL_WRITE;
        }
        
        // Write a newline character so that the next log starts on a new line:
        static const char newline[] = "\r\n";
        bytes_to_write = sizeof(newline) -1u; // -1u means it will truncate the C string null character '\0' at the end
        fr = f_write(&event_file,
                     newline,
                     bytes_to_write,
                     &bytes_written);
        if (fr != FR_OK) {
                f_close(&event_file);
                return STORAGE_ERR_WRITE;
        }
        if (bytes_to_write != bytes_written) {
                f_close(&event_file);
                return STORAGE_ERR_PARTIAL_WRITE;
        }

        // Close event log:
        fr = f_close(&event_file);
        if (fr != FR_OK) { return STORAGE_ERR_CLOSE; }

        // Success:
        return STORAGE_OK;
}
```

As you can see creating and appending to files with FatFs is very simple! For occasional event logs, this is fine. The code opens the log file, appends the message, appends a newline, and closes the file again. That is simple and robust enough for low rate event logging. I would not use this exact pattern for streaming sensor data though. For that I would rather buffer samples in RAM and write larger chunks to the SD card.


Now into our main file we will have a go at using the API, and it is now blissfully simple to write to the log files:

```c
#include "app/app_storage.h"

int main (void) {

        // ...other initialisation code like clocks, SPI, etc...

        storage_init();
        storage_log_event("An event that needs to be logged to EVENT.LOG!");

        // rest of code..
}
```

That is it. Easy peasy. Now you can try adding things like a real time clock readout to the beginning of the message, but that can be done within the app_storage.c file now privately!

Lastly a little trick, that I realised whilst thinking about the status error code flow: later I can add a helper function to convert a `storage_status_t` value into a printable string:

```c
const char *storage_status_string(storage_status_t status);
```

Then I can print the status over UART or include it in a log:

```c
uart_write_string(storage_status_string(status));
```


<script src="https://utteranc.es/client.js"
        repo="skoopsy/skoopsy.github.io"
        issue-term="pathname"
        label="blog-embedded18"
        theme="preferred-color-scheme"
        crossorigin="anonymous"
        async>
</script>

Copyright © 2026 David O'Connor
