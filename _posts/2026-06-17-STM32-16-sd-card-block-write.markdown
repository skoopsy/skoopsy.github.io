---
layout: post
title:  "STM32 #15: Writing raw blocks on a microSD with SPI"
date:   2026-06-16 19:40:00:00 +0000
categories: STM32
---

The [last post](https://skoopsy.dev/stm32/2026/06/16/STM32-15-sd-card-block-read.html) finished the raw block read path. Using `CMD17`, I read block 0, followed the MBR to the FAT32 boot sector, found the root directory, decoded the HELLO.TXT directory entry, and finally read the actual file contents from the data sector. That proved the read side of the driver. But there was one important cheat in that test: the file itself was still written by macOS.

So this time I want to build and test the other half of our low level SD driver: writing a raw 512 byte block from the STM32 using `CMD24`.

One warning before going further: this is still not filesystem aware writing. I am not asking FatFs to create a file, allocate a cluster, update a directory entry, or modify the FAT tables. I am just writing 512 bytes directly to a block address. If I write to the wrong block, I can corrupt the card. So this test belongs on a disposable card, or at least a card whose contents I do not care about.

The goals for this post are simple:

- write 512 bytes to a known test block
- read the same block back with CMD17
- compare the transmitted and received buffers

If that works, then the driver can provide the two primitives FatFs will need:

```c
sd_status_t sd_card_cmd17_read_block(uint32_t block_index, uint8_t *buffer);
sd_status_t sd_card_cmd24_write_block(uint32_t block_index, const uint8_t *buffer);
```

At that point, `disk_read()` and `disk_write()` should mostly become translation wrappers between FatFs and our lower level SD card driver.

# CMD24

For the SDHC/SDXC card I'm using, the argument passed to CMD24 is a block number. So if I want to write block 100,000, I send 100,000 as the command argument. For older SDSC cards, the argument is a byte address, so the block index would need to be multiplied by 512 via the helper function developed in the last post.

The CMD24 command byte is: `0x40 | 24 = 0x58`. So the command packet for writing to block 100000 should look like 58 [address bytes] FF. The FF at the end is a dummy CRC byte. In SPI mode after initialisation, CRC checking is normally disabled unless explicitly enabled.

# The write sequence

1. Select the card
2. Send CMD24 with the block address
3. Wait for an R1 response
4. Check that R1 is 0x00
5. Send the 0xFE data token
6. Send 512 bytes of block data
7. Send two dummy CRC bytes
8. Read the data response token
9. Wait while the card is busy programming the flash
10. Deselect the card

# New macros to add

These are the extra definitions I need for the single block write:

```c
#define SD_CMD24 24u

#define SD_BLOCK_SIZE 512u // in last post
#define SD_DATA_TOKEN 0xFEu // in last post
#define SD_CRC_SIZE 2u // in last post

#define SD_DATA_RESPONSE_MASK 0x1Fu 
#define SD_DATA_ACCEPTED 0x05u 
#define SD_WRITE_BUSY_ATTEMPTS 10000u
```

The write data token is the same used for a single block read payload. This time, for the write the host will send the `0xFE` data token to the card, then the 512 byte payload. After receiving the block the card returns a data response token. The lower 5 bits are most interesting for us:

```
xxxxx101 = data accepted 
xxxxx010 = CRC error 
xxxxx011 = write error
```

So to check if data has been accepted run this code:
```c
if ((data_response & SD_DATA_RESPONSE_MASK) != SD_DATA_ACCEPTED) { 
        return SD_ERR_BAD_RESPONSE; 
}
```

Only the lower 5 bits have to equal 0x05.

# Waiting for the card to finish writing

After the card has accepted data, it might hold the MISO line low while it writes to the internal flash. When the card is busy, reads return `0x00`. When the card is ready again, reads return `0xFF`.

So we need to build a helper that keeps clocking the card until it releases the bus:

```c
static sd_status_t sd_wait_write_complete(void) {
        for (uint32_t i = 0; i < SD_WRITE_BUSY_ATTEMPTS; ++i) {
                uint8_t value = 0x00u;

                bsp_spi_status_t spi_status = bsp_spi1_transfer(0xFFu, &value);
                if (spi_status != BSP_SPI_OK) { return SD_ERR_SPI; }
                if (value == 0xFFu) { return SD_OK; }
        }
        return SD_ERR_TIMEOUT;
}
```

This is a very crude timeout, a better version would use a hardware timer so that it is not based on loop count, but for now this is fine, and stops the driver hanging if the card never finishes the write cycle.

# The write block function

Now we will build the function to use `CMD24` to write a block:

```c
sd_status_t sd_card_cmd24_write_block(uint32_t block_index,
                                      const uint8_t *buffer) {
        if (buffer == NULL) { return SD_ERR_PARAM; }

        uint32_t address = sd_block_to_card_address(block_index);
        uint8_t r1 = 0xFFu;

        // Select the card
        sd_select();

        // Send CMD24 with the block address
        sd_status_t status = sd_write_command_packet(SD_CMD24,
                                                     address,
                                                     SD_DUMMY_CRC);
        if (status != SD_OK) { 
                sd_deselect();
                return status; 
        }
        
        // Wait for an R1 response
        status = sd_wait_r1(&r1);
        if (status != SD_OK) {
                sd_deselect();
                return status;
        }

        // Check that R1 is 0x00
        if (r1 != SD_R1_READY_STATE) {
                sd_deselect();
                return SD_ERR_BAD_RESPONSE;
        }

        // Send the 0xFE data token
        uint8_t token = SD_DATA_TOKEN;
        bsp_spi_status_t spi_status = bsp_spi1_write_buffer(&token, 1u);
        if (spi_status != BSP_SPI_OK) {
                sd_deselect();
                return SD_ERR_SPI;
        }

        // Send 512 bytes of block data
        spi_status = bsp_spi1_write_buffer(buffer, SD_BLOCK_SIZE);
        if (spi_status != BSP_SPI_OK) {
                sd_deselect();
                return SD_ERR_SPI;
        }
        
        // Send two dummy CRC bytes
        uint8_t crc[SD_CRC_SIZE] = {0xFFu, 0xFFu};
        spi_status = bsp_spi1_write_buffer(crc, sizeof(crc));
        if (spi_status != BSP_SPI_OK) {
                sd_deselect();
                return SD_ERR_SPI;
        }

        // Read the data response token
        uint8_t data_response = 0xFFu;
        spi_status = bsp_spi1_transfer(0xFFu, &data_response);
        if (spi_status != BSP_SPI_OK) {
                sd_deselect();
                return SD_ERR_SPI;
        }

        if ((data_response & SD_DATA_RESPONSE_MASK) != SD_DATA_ACCEPTED) {
                sd_deselect();
                return SD_ERR_BAD_RESPONSE;
        }

        // Wait while the card is busy programming the flash
        status = sd_wait_write_complete();
        if (status != SD_OK) {
                sd_deselect();
                return status;
        }
        
        // Deselect the card
        sd_deselect();
        return SD_OK;
}
```
It might look like a lot but it is all things we've seen before and just follows the write sequence outlined earlier, with some error handling. The sd_wait_write_complete() part towards the end is important, if we skip this and send another command whilst the card is busy writing flash the command will probably fail or worse.

We have to be careful as we are just writing random blocks here without using FatFS, but because we created and accessed the HELLO.TXT file in the last post and I still have it on the card, I now have a known sector that belongs to the existing HELLO.TXT file, so I can overwrite that sector and check whether the file contents change.

HELLO.TXT had these directory details from the last post:

- first_cluster = 182
- file_size = 20 bytes
- file_lba = 70946

We have to be careful to not change the file size. I am not updating the FAT directory entry, so macOS will still think the file is 20 bytes long. I am only overwriting the existing file data inside its already allocated cluster. 

```
HELLO      5
_          1  = 6
FROM       4  = 10
_          1  = 11
SKOOPSY    7  = 18
\r\n       2  = 20
```
Let's replace it with another 20 byte string: `HELLO_FROM_STM32!!\r\n`

# Comparing buffers
We can write this function so that we can compare the buffer on the micro after a write then read of the same block. But we will also probably check on the laptop for good measure. Here is the compare function:

```c
static uint32_t compare_buffers(const uint8_t *a,
                               const uint8_t *b,
                               uint32_t length) {
        uint32_t errors = 0u;

        for (uint32_t i=0; i<length; ++i) {
                if (a[i] != b[i]) {
                        ++errors;
                }
        }

        return errors;
}
```

Let's build the test in main now, we will read the HELLO.TXT sector contents again, then modify the first 20 bytes of the file, write it back, then read it again and run the buffer compare, along with printing it out in hex over UART:

```c
#define HELLO_FILE_LBA 70946u // LBA of the first data sector for HELLO.TXT
uint8_t tx_block[SD_BLOCK_SIZE];
uint8_t rx_block[SD_BLOCK_SIZE];

// Initialise card
sd_status_t status = sd_card_init();
if (status != SD_OK) {
    bsp_uart1_write_string("SD init failed\r\n");
    while (1) {}
}

// Read existing HELLO.TXT data sector into tx_block
status = sd_card_cmd17_read_block(HELLO_FILE_LBA, tx_block);
if (status != SD_OK) {
    bsp_uart1_write_string("Read HELLO.TXT sector failed\r\n");
    while (1) {}
}
debug_uart_print_hex_buffer("HELLO sector readback", tx_block, 32u);

// Create the new message
// This must stay the same length as the existing file: 20 bytes.
const char message[] = "HELLO_FROM_STM32!!\r\n";
// message includes the string terminator, so write sizeof(message) - 1
for (uint32_t i = 0; i < sizeof(message) - 1u; ++i) {
    tx_block[i] = (uint8_t)message[i];
}

// Write the modified 512 byte sector back
status = sd_card_cmd24_write_block(HELLO_FILE_LBA, tx_block);
if (status != SD_OK) {
    bsp_uart1_write_string("Write HELLO.TXT sector failed\r\n");
    while (1) {}
}
bsp_uart1_write_string("Write HELLO.TXT sector OK\r\n");

// Read the block we've just written
status = sd_card_cmd17_read_block(HELLO_FILE_LBA, rx_block);
if (status != SD_OK) {
    bsp_uart1_write_string("Readback failed\r\n");
    while (1) {}
}

// Compare the results
uint32_t errors = compare_buffers(tx_block, rx_block, SD_BLOCK_SIZE);

if (errors == 0u) {
    bsp_uart1_write_string("Raw block verify OK\r\n");
} else {
    bsp_uart1_write_string("Raw block verify failed\r\n");
}

debug_uart_print_hex_buffer("HELLO sector readback", rx_block, 32u);
```

I've uploaded to the board and run a minicom terminal to monitor UART and we have this output:
```
HELLO sector readback: 48 45 4C 4C 4F 5F 46 52 4F 4D 5F 53 4B 4F 4F 50 53 59 0D 0A 00 00 00 00 00 00 00 00 00 00 00 00

Write HELLO.TXT sector OK

Raw block verify OK

HELLO sector readback: 48 45 4C 4C 4F 5F 46 52 4F 4D 5F 53 54 4D 33 32 21 21 0D 0A 00 00 00 00 00 00 00 00 00 00 00 00
```

Great, so the block changed from:
```
48 45 4C 4C 4F 5F 46 52 4F 4D 5F 53 4B 4F 4F 50 53 59 0D 0A
=
HELLO_FROM_SKOOPSY\r\n
``` 
to 
```
48 45 4C 4C 4F 5F 46 52 4F 4D 5F 53 54 4D 33 32 21 21 0D 0A
=
HELLO_FROM_STM32!!\r\n
```

I'm happy with that; we can now confidently read and write raw blocks with the SD card driver. A note on this though, this worked because HELLO.TXT was already a file, and I knew where the file contents were so could overwrite it fairly easily without changing the file size or allocation. I didn't create a new file, update timestamps, or allocate clusters. That is for FatFS to do!

<script src="https://utteranc.es/client.js"
        repo="skoopsy/skoopsy.github.io"
        issue-term="pathname"
        label="blog-embedded15"
        theme="preferred-color-scheme"
        crossorigin="anonymous"
        async>
</script>


Copyright © 2026 David O'Connor
