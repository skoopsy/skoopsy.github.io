---
layout: post
title:  "STM32 #19: Adding CRC error detection"
date:   2026-10-13 10:00:00 +0000
categories: STM32
---

Last time I added a simple storage API around FatFs to make writing data to the microSD card easier from application code. Now I need a way of detecting whether stored/transferred data has been corrupted, using a [Cyclic Redundancy Check (CRC)](https://en.wikipedia.org/wiki/Cyclic_redundancy_check).

A CRC is an error detection method used to identify changes in stored or transmitted data. A CRC value is calculated from the data when it is written. The same calculation can be repeated when the data is read back. If the values don't match then the data has changed.

CRCs are used in communication protocols everywhere in storage systems and file formats because they are relatively cheap to calculate and provide a fairly good (and quantifiable) level of confidence in detection of common error patterns, excluding deliberate modification.

# How CRC works
A CRC treats a sequence of bits as the coefficients of a polynomial over the two element field GF(2), e.g. For a three byte message:

```
Hi\n
```

It is represented as:

```
0x48 0x69 0x0A
```

or in binary:

```
01001000 01101001 00001010
```

The set bits can be interpreted as terms in a polynomial. The message polynomial is divided by a predefined generator polynomial using arithmetic over GF(2). In GF(2), subtraction and addition can both be performed using XOR:

```
0 XOR 0 = 0
0 XOR 1 = 1
1 XOR 0 = 1
1 XOR 1 = 0
```

To calculate the CRC, zero bits equal to the width of the CRC are conceptually appended to the message. The message is divided by the generator polynomial and the remainder becomes the CRC value. Only storing the current remainder is required after each step, so the message can be processed incrementally one byte at a time.

For more details on CRCs check out: 
- Ross N Williams [A Painless Guide to CRC Error Detection Algorithms](http://www.ross.net/crc/crcpaper.html) and more generally the [ross.net/crc website](http://www.ross.net/crc/).
- Ben Eater's [How do CRCs Work?](https://www.youtube.com/watch?v=izG7qT0EpBw&vl=en) on youtube
- Philip Koopman's [lecture slides at Carnegie Mellon University titled: "Software and Digital Systems Program - Data Integrity Techniques"](https://users.ece.cmu.edu/~koopman/pubs/feb2013_koopman_crc_completion_public.pdf)

# Selecting a CRC
There are a bunch of different CRCs variants using different generator polynomials and parameters, with different error detection properties for different message lengths, and error patterns, see the wikipedia for a list of many common variations. Additionally the same CRC algorithm can be calculated using simple bit by bit implementation, a lookup table, or a hardware CRC peripheral. 

I'm using a 32 bit CRC calculated over blocks of stored data for this project. A 32 bit CRC has 2^32 = 4,294,967,296 possible values. For random independent corruption, the probability of an undetected error is approximately 1 / 2^32, or about 0.0000000233%.

This is only an average probability for random corruption. In practice, CRC performance also depends on the generator polynomial, the message length and the structure of the error. Well selected CRC polynomials can guarantee detection of many common error patterns.

For comparison, a 16 bit CRC has only 2^16 = 65,536 possible values, giving an undetected error probability of approximately 1 / 65,536 under the same random corruption assumption. This is substantially higher than for a 32 bit CRC. 

've tried to create a rough illustration of the practical differences here, if every 512 byte block in a data logger were corrupted and those errors behaved randomly, we can calculate how long it would take until a corrupted block goes undetected:

At 200 records/s with 24 records per block: ```200 / 24 ≈ 8.33 blocks/s```

For a 32 bit CRC, the average number of corrupted blocks before one undetected error is approximately: ```2^32 = 4,294,967,296 blocks```

So: ```4,294,967,296 / 8.33 ≈ 515,000,000 s``` ≈ 16.3 years

For a 16 bit CRC: ```65,536 / 8.33 ≈ 7,864 s``` ≈ 2.2 hours

These aren't expected failure rates, they just illustrate the difference in undetected error probability between 16 bit and 32 bit CRCs under the same simplified random error model.

Lastly a practical note; A 32bit CRC adds four bytes of overhead to each protected block, which is small compared to the size of data being written. 

# CRC Implementation
I'm going to start with a simple bitwise CRC. There are faster ways, but this is simple and small. Here is a implementation of a simple reflected CRC:

```c
uint32_t crc32_compute(const uint8_t *data, size_t length)
{
    uint32_t crc = CRC_INITIAL_VALUE;

    for (size_t i = 0; i < length; i++) {
        crc ^= data[i];

        for (unsigned bit = 0u; bit < 8u; ++bit) {
            if ((crc & 1u) != 0u) {
                crc = (crc >> 1u) ^ CRC_POLYNOMIAL;
            }
            else {
                crc >>= 1u;
            }
        }
    }

    return crc ^ CRC_FINAL_XOR;
}```

This processes each byte and updates the CRC register eight times, once for each bit. This processes the data least significant bit first. It uses the reflected form of the selected generator polynomial. I've defined the polynomial and other CRC parameters separately so that the implementation can be checked against known reference values.

# Testing

The implementation needs to be checked against known reference values because CRC implementations are easy to get almost right.

A commonly used test message is:

```c
static const uint8_t test_message[] = "123456789";
```

The calculated result can then be compared with the published check value for the selected CRC variant. This verifies that the polynomial representation, initial value, bit ordering and final XOR are all configured correctly.

I also test the implementation using buffers representative of the data passed through the storage layer, so that both the CRC routine and the way data is supplied to it are covered.

# Using CRC with the storage layer

The next step is to decide exactly what data is protected by each CRC and how the CRC value is stored alongside it. The storage layer can calculate the CRC then write a block and verify it again when that block is read. If the two values do not match, the block can be marked as corrupted rather than silently treating the data as valid.

This gives the storage format a basic integrity check before I start building the higher level logging system around it.

<script src="https://utteranc.es/client.js"
        repo="skoopsy/skoopsy.github.io"
        issue-term="pathname"
        label="blog-embedded19"
        theme="preferred-color-scheme"
        crossorigin="anonymous"
        async>
</script>

Copyright © 2026 David O'Connor
