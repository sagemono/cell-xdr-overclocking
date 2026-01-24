# PS3 CELL/XDR Overclocking Research and Reverse Engineering

---
## Disclaimer: This is not a guide in any way shape or form, this is only a proof of concept. There is a team working behind this and is still in the experimental stage. This may cause irreversible damage to your system and should not be performed by the average user. We are not responsible for any damaged systems. 
---

## Project Overview

This document details the reverse engineering of the PlayStation 3 syscon clock generation subsystem, specifically focusing on CELL and XDR clock configuration. The research resulted in the discovery of software based overclocking capabilities on retail PS3 consoles through NVS modification.

### Target System
- **Console Model**: CECHA00

### Achievements
- Decoded the complete clock frequency lookup table (25 entries)
- Mapped NVS offsets controlling clock generators
- Identified lv0 firmware whitelist restrictions (this can potentially be solved on winning the silicon lottery and chips with a differnet revision, 90/65/40/28nm)
- Achieved semi stable 4.0 GHz CELL overclock (25% over stock 3.2 GHz)
- Documented XDR clock limitations and CELL/XDR ratio requirements

---

## Clock Architecture

### Hardware Components

The PS3 COK-001 (CECHA) motherboard uses the following clock generation hardware:

**Clock Generator ICs:**
- **IC5002, IC5003**: ICS9218AGLFT (IC CLOCK GEN RAMBUS XDR 28-TSSOP)
  - IC5003 generates CELL reference clock
  - IC5002/IC5004 generates XDR reference clock

**Crystal Oscillators on COK-001:**
- 25 MHz (×2) - Primary reference for clock generators

### Clock Chain

```
┌─────────────────────────────────────────────────────────────────┐
│             25 MHz Crystal Oscillator (X5001/X3402)             |   
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│               Master Clock Oscillator (IC5001)                  │
│                   NVS 0x3122 → Register 0                       |
│              I2C programmable clock synthesizer                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
┌─────────────────────────────┐ ┌─────────────────────────────────┐
│ IC5003 - ICS9218AGLFT       │ │ IC5002/5004 - ICS9218AGLFT      │
│ CELL Clock Generator        │ │ XDR Clock Generator             │
│ (cell_device_slot: 0x42080) │ │ (xdr_context_id_probable)       │
│                             │ │                                 |
│ NVS 0x3128 → Register 5     │ │ NVS 0x312C → Register 5         │
│ NVS 0x3129 → Register 6     │ │ NVS 0x312D → Register 6         │
│                             │ │                                 │
│ Output: 400 MHz (stock)     │ │ Output: 400 MHz (stock)         │
└──────────────┬──────────────┘ └────────────────┬────────────────┘
               │                                 │
               ▼                                 ▼
┌─────────────────────────────┐ ┌─────────────────────────────────┐
│           CELL BE           │ │             XDR DRAM            │
│   Internal PLL: 8× mult     │ │      Internal PLL: 8× mult      │
│                             │ │                                 │
│   400 MHz × 8 = 3.2 GHz     │ │       400 MHz × 8 = 3.2 GHz     │
│          (stock)            | │              (stock)            │
└─────────────────────────────┘ └─────────────────────────────────┘
```

### Device Context Structures

Each clock generator has an associated device context structure in syscon memory. All clock generators are accessed via I2C using the same register read/write functions.

| Device | Context Address | Function |
|--------|-----------------|----------|
| Master Oscillator | 0x42074 | Reference clock source |
| IC5003 (CELL) | 0x42080 | CELL reference clock |
| IC5002/5004 (XDR) | 0x4208C | XDR reference clock |

The device context structure format:
```c
struct device_context {
    uint32_t bus_handle;      // +0x00: I2C bus handle
    uint32_t reserved;        // +0x04
    uint8_t  i2c_address;     // +0x08: 7-bit I2C address
    uint8_t  padding[3];      // +0x09
    // ... additional fields
};
```

---

## NVS Memory Map

### Clock Configuration Offsets

| NVS Offset | Size | Device | Register | Default | Description |
|------------|------|--------|----------|---------|-------------|
| 0x3122 | 1 | Master Osc | 0 | 0x20 | Master oscillator config (bits 6-4 select frequency) |
| 0x3128 | 1 | IC5003 | 5 | 0x84 | CELL clock generator register 5 |
| 0x3129 | 1 | IC5003 | 6 | 0x16 | CELL clock generator register 6 |
| 0x312C | 1 | IC5002/4 | 5 | 0x84 | XDR clock generator register 5 |
| 0x312D | 1 | IC5002/4 | 6 | 0x16 | XDR clock generator register 6 |

### Default Value Behavior

When NVS contains 0xFF (erased/unset), syscon applies hardcoded defaults:

```c
// From sub_2DC9E - Clock Configuration Function
nvs_read(0x3122, v5, 1);
if (v5[0] == 0xFF) v5[0] = 0x20;  // Default master osc config

nvs_read(0x3128, v5, 1);
if (v5[0] == 0xFF) v5[0] = 0x84;  // Default CELL reg5

nvs_read(0x3129, v5, 1);
if (v5[0] == 0xFF) v5[0] = 0x16;  // Default CELL reg6 (22 decimal)

nvs_read(0x312C, v5, 1);
if (v5[0] == 0xFF) v5[0] = 0x84;  // Default XDR reg5

nvs_read(0x312D, v5, 1);
if (v5[0] == 0xFF) v5[0] = 0x16;  // Default XDR reg6
```

---

## Firmware Analysis

### Clock Configuration Function (sub_2DC9E)

This function initializes all clock generators during boot. It reads NVS values, applies defaults if necessary, and writes to the clock generator registers via I2C.

```c
int __fastcall sub_2DC9E(int a1, int a2, int a3, int a4)
{
    int result;
    _DWORD v5[6];
    
    v5[0] = a4;
    
    // Configure master oscillator (NVS 0x3122 → Register 0)
    result = nvs_read(0x3122, v5, 1);
    if (!result) {
        if (LOBYTE(v5[0]) == 0xFF)
            LOBYTE(v5[0]) = 0x20;  // Default: bits 6-4 = 001
        result = sub_3851C(&master_clock_oscillator_device_slot, 0, 0x7F, LOBYTE(v5[0]));
        
        if (!result) {
            // Initialize CELL clock generator
            LOBYTE(v5[0]) = 0;
            result = sub_38614(&cell_device_slot, 0, 3, 0);  // Init register 0
            
            if (!result) {
                // Configure CELL reg5 (NVS 0x3128)
                result = nvs_read(0x3128, v5, 1);
                if (!result) {
                    if (LOBYTE(v5[0]) == 0xFF)
                        LOBYTE(v5[0]) = 0x84;  // Default
                    result = sub_38614(&cell_device_slot, 5, 0xFF, LOBYTE(v5[0]));
                    
                    if (!result) {
                        // Configure CELL reg6 (NVS 0x3129)
                        result = nvs_read(0x3129, v5, 1);
                        if (!result) {
                            if (LOBYTE(v5[0]) == 0xFF)
                                LOBYTE(v5[0]) = 0x16;  // Default (22 decimal)
                            result = sub_38614(&cell_device_slot, 6, 0xFF, LOBYTE(v5[0]));
                            
                            // ... XDR clock generator configured identically ...
                        }
                    }
                }
            }
        }
    }
    return result;
}
```

### I2C Register Access Functions

All clock generators use the same I2C access functions:

**sub_384A4 / sub_3859C - Register Read:**
```c
int __fastcall sub_384A4(int context, int reg, uint8_t *value, int unused)
{
    char buffer[2];
    buffer[0] = 0;
    buffer[1] = 0;
    
    // I2C transaction: write register number, read 2 bytes back
    result = i2c_write_then_read(
        *(uint32_t*)context,           // Bus handle at offset +0
        *(uint8_t*)(context + 8),      // I2C address at offset +8
        reg,                            // Register number to read
        buffer,                         // Read buffer
        2                               // Read 2 bytes
    );
    
    *value = buffer[1];  // Return second byte as the register value
    
    if (result) {
        int slot = errlog_slot_for_ctx(context);
        errlog_emit_event(slot + 0x2100);  // Log I2C read error
    }
    return result;
}
```

**sub_385DE - Register Write:**
```c
int __fastcall sub_385DE(int context, uint8_t reg, uint8_t value, int unused)
{
    uint8_t packet[3];
    packet[0] = reg;      // Register number
    packet[1] = 0x01;     // Write command byte
    packet[2] = value;    // Value to write
    
    result = bus_xfer_write3_select_mode(
        *(uint32_t*)context,           // Bus handle
        *(uint8_t*)(context + 8),      // I2C address
        packet,
        3                               // 3 bytes (?)
    );
    
    if (result) {
        int slot = errlog_slot_for_ctx(context);
        errlog_emit_event(slot + 0x2000);  // Log I2C write error
    }
    return result;
}
```

**sub_38614 - Read-Modify-Write:**
```c
int __fastcall sub_38614(int context, int reg, uint8_t mask, uint8_t value)
{
    uint8_t current;
    
    result = sub_3859C(context, reg, &current, 0);  // Read current value
    if (!result) {
        // Apply mask: clear masked bits, then set new value's masked bits
        uint8_t new_value = (current & ~mask) | (mask & value);
        return sub_385DE(context, reg, new_value, 0);  // Write back
    }
    return result;
}
```

### Frequency Lookup Function (sub_1E430)

This function retrieves the current clock frequency by reading the clock generator registers and looking up the corresponding frequency in a ROM table.

```c
int __fastcall sub_1E430(int device_type, uint32_t *freq_out, uint32_t *unused, int arg)
{
    uint8_t reg5, reg6;
    
    *unused = 0;
    
    switch (device_type) {
        case 0:  // XDR clock generator
            // Read registers 5 and 6 from XDR clock gen
            result1 = sub_3859C(&xdr_context_id_probable, 5, &reg5, arg);
            result2 = sub_3859C(&xdr_context_id_probable, 6, &reg6, arg);
            if (result1 | result2) return 254;
            goto lookup_clock_gen;
            
        case 3:  // CELL clock generator
            // Read registers 5 and 6 from CELL clock gen
            result1 = sub_3859C(&cell_device_slot, 5, &reg5, arg);
            result2 = sub_3859C(&cell_device_slot, 6, &reg6, arg);
            if (result1 | result2) return 254;
            
        lookup_clock_gen:
            // Build 16-bit lookup key: (reg5 << 8) | reg6
            // e.g., reg5=0x84, reg6=0x16 → key=0x8416
            uint16_t key = (reg5 << 8) | reg6;
            
            // Search clock generator table (25 entries at byte_40148+64)
            for (int i = 0; i < 25; i++) {
                // Table stores key as little-endian: [reg6, reg5, pad, pad, freq...]
                uint16_t table_key = *(uint16_t*)&byte_40148[64 + i*8];
                if (table_key == key) {
                    // Found match - get frequency (bytes 4-7, little-endian)
                    uint32_t freq_le = *(uint32_t*)&byte_40148[64 + i*8 + 4];
                    // Convert from little-endian to big-endian via byte swap
                    *freq_out = sub_2CD3E(freq_le);
                    return 0;
                }
            }
            return 254;  // Not found in table
            
        case 2:  // Fixed reference (used for calculations)
            // Returns 100 MHz as a reference constant
            *freq_out = sub_2CD3E(100000000);
            return 0;
            
        case 4:  // Master oscillator
            // Read register 0 from master oscillator
            sub_384A4(&master_clock_oscillator_device_slot, 0, &reg5, arg);
            
            // Extract bits 6-4 as index: (reg5 << 25) >> 29
            // e.g., 0x20 → bits 6-4 = 001 → index 1
            int index = (reg5 << 25) >> 29;
            
            // Search master oscillator table (8 entries at byte_40148)
            for (int i = 0; i < 8; i++) {
                if (byte_40148[i*8] == index) {
                    uint32_t freq_le = *(uint32_t*)&byte_40148[i*8 + 4];
                    *freq_out = sub_2CD3E(freq_le);
                    return 0;
                }
            }
            return 254;
            
        default:
            *freq_out = 0;
            return 2;
    }
}
```

### Byte Swap Function (sub_2CD3E)

The frequency values in the lookup table are stored in little-endian format in firmware. This function performs a 32-bit byte swap to convert between endianness.

```c
unsigned int __fastcall sub_2CD3E(unsigned int a1)
{
    // Swap byte order: 0xAABBCCDD → 0xDDCCBBAA
    return (a1 << 24) |              // Move byte 0 to position 3
           (a1 >> 24) |              // Move byte 3 to position 0
           ((a1 >> 8) & 0xFF00) |    // Move byte 2 to position 1
           ((a1 << 8) & 0xFF0000);   // Move byte 1 to position 2
}
```

**Example - Stock 400 MHz frequency:**
```
ROM bytes at 0x40170: 00 84 D7 17

When loaded as 32-bit by ARM (little-endian load): 0x17D78400
After byte swap via sub_2CD3E:                     0x0084D717

Decimal: 0x17D78400 = 400,000,000 Hz = 400 MHz
```

The byte swap is necessary because the syscons's ARM core loads values in little-endian format, but the firmware expects to work with the raw frequency value as stored in the table.

---

## Frequency Lookup Tables

### Table Structure

The frequency lookup table at ROM address `byte_40148` contains two sections:

1. **Master Oscillator Table** (offset 0x00, 8 entries × 8 bytes)
2. **Clock Generator Table** (offset 0x40/64, 25 entries × 8 bytes)

### Entry Format

**Master Oscillator Entry (8 bytes):**
```
Offset: +0   +1   +2   +3   +4   +5   +6   +7
        [idx][pad][pad][pad][  frequency (LE)  ]
        
idx: Value matching bits 6-4 of register 0
frequency: 32-bit little-endian frequency in Hz
```

**Clock Generator Entry (8 bytes):**
```
Offset: +0   +1   +2   +3   +4   +5   +6   +7
        [r6 ][r5 ][pad][pad][  frequency (LE)  ]
        
r6: Register 6 value
r5: Register 5 value  
frequency: 32-bit little-endian frequency in Hz

The lookup key is built as: (reg5 << 8) | reg6
So entry [16 84 ...] matches key 0x8416 (reg5=0x84, reg6=0x16)
```

### Master Oscillator Table (0x40148, 8 entries)

| Index | Bits 6-4 | NVS 0x3122 | Frequency | Raw Bytes |
|-------|----------|------------|-----------|-----------|
| 0 | 000 | 0x00 | 300,000,000 Hz | `00 00 00 00 00 A3 E1 11` |
| 1 | 001 | 0x20 | 400,000,000 Hz | `01 00 00 00 00 84 D7 17` |
| 2 | 010 | 0x40 | 500,000,000 Hz | `02 00 00 00 00 65 CD 1D` |
| 3 | 011 | 0x60 | 600,000,000 Hz | `03 00 00 00 00 46 C3 23` |
| 4 | 100 | 0x80 | 800,000,000 Hz | `04 00 00 00 00 08 AF 2F` |
| 5 | 101 | 0x50 | 450,000,000 Hz | `05 00 00 00 80 74 D2 1A` |
| 6 | 110 | 0x60 | 750,000,000 Hz | `06 00 00 00 80 17 B4 2C` |
| 7 | 111 | 0x70 | 375,000,000 Hz | `07 00 00 00 C0 0B 5A 16` |

**Note:** Index 1 (0x20) is the default, outputting 400 MHz.

### Clock Generator Table (0x40188, 25 entries)

| reg6 | reg5 | Key | Frequency (Hz) | Ref Clock | CELL (×8) | Raw Bytes |
|------|------|-----|----------------|-----------|-----------|-----------|
| 0x10 | 0x84 | 0x8410 | 300,000,000 | 300 MHz | 2.4 GHz | `10 84 00 00 00 A3 E1 11` |
| 0x0B | 0x82 | 0x820B | 325,000,000 | 325 MHz | 2.6 GHz | `0B 82 00 00 40 1B 5F 13` |
| 0x13 | 0x84 | 0x8413 | 350,000,000 | 350 MHz | 2.8 GHz | `13 84 00 00 80 93 DC 14` |
| 0x14 | 0x84 | 0x8414 | 366,503,851 | 367 MHz | 2.9 GHz | `14 84 00 00 AB E3 DA 15` |
| 0x0D | 0x82 | 0x820D | 375,000,000 | 375 MHz | 3.0 GHz | `0D 82 00 00 C0 0B 5A 16` |
| 0x15 | 0x84 | 0x8415 | 383,328,213 | 383 MHz | 3.1 GHz | `15 84 00 00 D5 33 D9 16` |
| 0x16 | 0x84 | 0x8416 | 400,000,000 | 400 MHz | **3.2 GHz** | `16 84 00 00 00 84 D7 17` |
| 0x17 | 0x84 | 0x8417 | 416,671,787 | 417 MHz | 3.3 GHz | `17 84 00 00 2B D4 D5 18` |
| 0x0F | 0x02 | 0x020F | 425,000,000 | 425 MHz | 3.4 GHz | `0F 02 00 00 40 FC 54 19` |
| 0x18 | 0x04 | 0x0418 | 433,343,573 | 433 MHz | 3.5 GHz | `18 04 00 00 55 24 D4 19` |
| 0x19 | 0x04 | 0x0419 | 450,000,000 | 450 MHz | 3.6 GHz | `19 04 00 00 80 74 D2 1A` |
| 0x1A | 0x04 | 0x041A | 466,656,427 | 467 MHz | 3.7 GHz | `1A 04 00 00 AB C4 D0 1B` |
| 0x11 | 0x02 | 0x0211 | 475,000,000 | 475 MHz | 3.8 GHz | `11 02 00 00 C0 EC 4F 1C` |
| 0x1B | 0x04 | 0x041B | 483,328,213 | 483 MHz | 3.9 GHz | `1B 04 00 00 D5 14 CF 1C` |
| 0x1C | 0x04 | 0x041C | 500,000,000 | 500 MHz | 4.0 GHz | `1C 04 00 00 00 65 CD 1D` |
| 0x1D | 0x04 | 0x041D | 516,671,787 | 517 MHz | 4.1 GHz | `1D 04 00 00 2B B5 CB 1E` |
| 0x1E | 0x04 | 0x041E | 533,343,573 | 533 MHz | 4.2 GHz | `1E 04 00 00 55 05 CA 1F` |
| 0x1F | 0x04 | 0x041F | 550,000,000 | 550 MHz | 4.4 GHz | `1F 04 00 00 80 55 C8 20` |
| 0x20 | 0x04 | 0x0420 | 566,656,427 | 567 MHz | 4.5 GHz | `20 04 00 00 AB A5 C6 21` |
| 0x21 | 0x04 | 0x0421 | 583,328,213 | 583 MHz | 4.7 GHz | `21 04 00 00 D5 F5 C4 22` |
| 0x22 | 0x04 | 0x0422 | 600,000,000 | 600 MHz | 4.8 GHz | `22 04 00 00 00 46 C3 23` |
| 0x23 | 0x04 | 0x0423 | 616,656,427 | 617 MHz | 4.9 GHz | `23 04 00 00 2B 96 C1 24` |
| 0x24 | 0x04 | 0x0424 | 633,328,213 | 633 MHz | 5.1 GHz | `24 04 00 00 55 E6 BF 25` |
| 0x25 | 0x04 | 0x0425 | 650,000,000 | 650 MHz | 5.2 GHz | `25 04 00 00 80 36 BE 26` |
| 0x26 | 0x04 | 0x0426 | 666,656,427 | 667 MHz | 5.3 GHz | `26 04 00 00 AB 86 BC 27` |

**Stock configuration:** reg5=0x84, reg6=0x16 → 400 MHz reference → 3.2 GHz CELL

### Raw Table Dump (0x40148)

```
; Master Oscillator Table (8 entries × 8 bytes)
; Format: [index] [pad×3] [frequency as little-endian 32-bit]
0x00: 00 00 00 00  00 A3 E1 11  ; idx=0: 0x11E1A300 = 300,000,000 Hz
0x08: 01 00 00 00  00 84 D7 17  ; idx=1: 0x17D78400 = 400,000,000 Hz (DEFAULT)
0x10: 02 00 00 00  00 65 CD 1D  ; idx=2: 0x1DCD6500 = 500,000,000 Hz
0x18: 03 00 00 00  00 46 C3 23  ; idx=3: 0x23C34600 = 600,000,000 Hz
0x20: 04 00 00 00  00 08 AF 2F  ; idx=4: 0x2FAF0800 = 800,000,000 Hz
0x28: 05 00 00 00  80 74 D2 1A  ; idx=5: 0x1AD27480 = 450,000,000 Hz
0x30: 06 00 00 00  80 17 B4 2C  ; idx=6: 0x2CB41780 = 750,000,000 Hz
0x38: 07 00 00 00  C0 0B 5A 16  ; idx=7: 0x165A0BC0 = 375,000,000 Hz

; Clock Generator Table (25 entries × 8 bytes, starting at offset 0x40)
; Format: [reg6] [reg5] [pad×2] [frequency as little-endian 32-bit]
0x40: 10 84 00 00  00 A3 E1 11  ; 0x8410: 300,000,000 Hz
0x48: 0B 82 00 00  40 1B 5F 13  ; 0x820B: 325,000,000 Hz
0x50: 13 84 00 00  80 93 DC 14  ; 0x8413: 350,000,000 Hz
0x58: 14 84 00 00  AB E3 DA 15  ; 0x8414: 366,503,851 Hz
0x60: 0D 82 00 00  C0 0B 5A 16  ; 0x820D: 375,000,000 Hz
0x68: 15 84 00 00  D5 33 D9 16  ; 0x8415: 383,328,213 Hz
0x70: 16 84 00 00  00 84 D7 17  ; 0x8416: 400,000,000 Hz (STOCK)
0x78: 17 84 00 00  2B D4 D5 18  ; 0x8417: 416,671,787 Hz
0x80: 0F 02 00 00  40 FC 54 19  ; 0x020F: 425,000,000 Hz
0x88: 18 04 00 00  55 24 D4 19  ; 0x0418: 433,343,573 Hz
0x90: 19 04 00 00  80 74 D2 1A  ; 0x0419: 450,000,000 Hz
0x98: 1A 04 00 00  AB C4 D0 1B  ; 0x041A: 466,656,427 Hz
0xA0: 11 02 00 00  C0 EC 4F 1C  ; 0x0211: 475,000,000 Hz
0xA8: 1B 04 00 00  D5 14 CF 1C  ; 0x041B: 483,328,213 Hz
0xB0: 1C 04 00 00  00 65 CD 1D  ; 0x041C: 500,000,000 Hz
0xB8: 1D 04 00 00  2B B5 CB 1E  ; 0x041D: 516,671,787 Hz
0xC0: 1E 04 00 00  55 05 CA 1F  ; 0x041E: 533,343,573 Hz
0xC8: 1F 04 00 00  80 55 C8 20  ; 0x041F: 550,000,000 Hz
0xD0: 20 04 00 00  AB A5 C6 21  ; 0x0420: 566,656,427 Hz
0xD8: 21 04 00 00  D5 F5 C4 22  ; 0x0421: 583,328,213 Hz
0xE0: 22 04 00 00  00 46 C3 23  ; 0x0422: 600,000,000 Hz
0xE8: 23 04 00 00  2B 96 C1 24  ; 0x0423: 616,656,427 Hz
0xF0: 24 04 00 00  55 E6 BF 25  ; 0x0424: 633,328,213 Hz
0xF8: 25 04 00 00  80 36 BE 26  ; 0x0425: 650,000,000 Hz
0x100: 26 04 00 00 AB 86 BC 27  ; 0x0426: 666,656,427 Hz
```

---

## lv0 Firmware Whitelist 

### Note: This entire section can be potentially ignored I was under the assumption that lv0 validates a set of whitelisted clock values due to encountering mutiple errors past a certain point. This may be the result of pushing the silicon to its limits, users have reported to have gotten further than my overclocking attempts as they have tried this on much newer hardware than my CECHA00

### Discovery Process

During boot, lv0 (the lowest-level PS3 firmware) validates clock frequencies by calling `get_reference_clock()`. This function queries syscon for the current clock generator settings and verifies they match an internal whitelist.

**Error codes observed:**
- `0xb0000001 get_reference_clock fail` - Frequency not in lv0 whitelist
- `0xb0000004 lv0 authentication fail` - Security validation failure
- `0xb0000004 lv0 not found` - Extreme underclock prevents lv0 from loading

### CELL Clock Whitelist (Tested)

| reg5 | reg6 | Frequency | lv0 Result |
|------|------|-----------|------------|
| 0x84 | 0x10 | 2.4 GHz | Accepted |
| 0x84 | 0x13 | 2.8 GHz | Accepted |
| 0x84 | 0x14 | 2.9 GHz | Accepted |
| 0x84 | 0x15 | 3.1 GHz | Accepted |
| 0x84 | 0x16 | 3.2 GHz | Accepted (stock) |
| 0x84 | 0x17 | 3.3 GHz | Accepted |
| 0x84 | 0x18 | 3.4 GHz | Accepted |
| 0x02 | 0x0F | 3.4 GHz | Accepted |
| 0x04 | 0x18 | 3.5 GHz | Accepted |
| 0x04 | 0x19 | 3.6 GHz | Accepted |
| 0x04 | 0x1A | 3.7 GHz | Accepted |
| 0x02 | 0x11 | 3.8 GHz | Accepted |
| 0x04 | 0x1B | 3.9 GHz | Accepted |
| 0x04 | 0x1C | 4.0 GHz | Accepted |
| 0x04 | 0x1D | 4.1 GHz | Accepted |
| 0x04 | 0x1E | 4.2 GHz | Rejected |

### XDR Clock Whitelist (Tested)

The XDR whitelist is significantly more restrictive:

| reg5 | reg6 | Frequency | lv0 Result |
|------|------|-----------|------------|
| 0x84 | 0x10 | 2.4 GHz | lv0 not found |
| 0x84 | 0x11 | 2.5 GHz | Rejected |
| 0x84 | 0x12 | 2.6 GHz | Rejected |
| 0x84 | 0x13 | 2.8 GHz | Accepted |
| 0x84 | 0x14 | 2.9 GHz | Accepted |
| 0x84 | 0x15 | 3.1 GHz | Accepted |
| 0x84 | 0x16 | 3.2 GHz | Accepted (stock) |
| 0x84 | 0x17 | 3.3 GHz | Rejected |
| 0x02 | 0x0F | 3.4 GHz | Accepted |
| 0x02 | 0x10 | 3.6 GHz | Rejected |
| 0x02 | 0x11 | 3.8 GHz | Rejected |
| 0x04 | 0x** | Any | Rejected |

**XDR ceiling: 3.4 GHz (0x020F)**

---

## CELL/XDR Frequency Ratio Requirement

### The 800 MHz Rule

The CELL processor communicates with XDR memory through the FlexIO interface. If the frequency difference between CELL and XDR exceeds approximately 800 MHz, the interface becomes unstable and fails during link initialization.

**Error observed when ratio exceeded:**
```
[ERROR]: 0xb0002001 (FATAL) XDR Link not initialized.
ITC_DUMP0000....
PTC_DUMP0000...
MIC_DUMP0000...
XIO_DUMP0001...
```

### Maximum Configurations

Given XDR ceiling of 3.4 GHz and 800 MHz rule:
- **Theoretical CELL max**: 3.4 + 0.8 = 4.2 GHz
- **Practical CELL max**: 4.0-4.1 GHz (silicon/voltage limited)

---

## Failed Approaches

### Custom PLL via 0x3958/0x3950

The syscon contains code for sending custom PLL configuration directly to CELL via SPI (sub_2CEA4). This path is enabled by setting NVS 0x3958 to non-0xFF value.

**Why it fails on retail:**
```c
result = nvs_read(0x3958, v10, 1);
if (v10[0] != 0xFF) {
    v1 = sub_2ADFA(v13);  // Wait for CELL attention - TIMES OUT
    if (v1) {
        log_printf("[POWERSEQ] Error : wait attention timeout.(%s)\n", "Psbd_SetBePll");
        return v1;
    }
    // ... send PLL config
}
```

---

## Boot Sequence Context

### Where Clock Configuration Occurs

```
Boot Phase 1 (sub_74A4)
    │
    ├── Step 0: Initial checks
    ├── Step 1-2: Power sequencing  
    ├── Step 3: PowerSeq_Setup called
    │       │
    │       └── sub_7758 (PowerSeq_Setup)
    │               │
    │               ├── sub_2CFA0: Unknown init
    │               ├── sub_2CFA4: Board config
    │               ├── sub_2D09C: Device init
    │               │       └── Calls sub_2DC9E (clock config)
    │               │               ├── Configure master oscillator
    │               │               ├── Configure IC5003 (CELL)
    │               │               └── Configure IC5004 (XDR)
    │               └── sub_2D094: Finalize
    │
    ├── Step 4-9: Power rail validation
    ├── Step 10: VID setup
    ├── Step 20-22: CELL attention
    ├── Step 30-31: CELL SPI init
    ├── Step 40: FlexIO training
    ├── Step 50-52: ByteTraining
    └── Step 60-62: Flash handoff
    
Boot Phase "0x400" (System Running)
    │
    └── lv0 loads and validates clocks
            │
            └── get_reference_clock() called
                    │
                    ├── Queries syscon for clock settings
                    ├── Validates against internal whitelist
                    └── Returns error if not whitelisted
```

## Credits

- [M4j0r](https://x.com/MinaRalwasser/) - For discovering that CELL can be overclocked in the first place via a DECR-1000A Reference Tool back in [2021](https://x.com/MinaRalwasser/status/1458862608384155650). 
- [Nascar1243](https://youtube.com/@nascar1243) - Figuring out the offets for the CELL clock generator registers
- [aomsin2526](https://github.com/aomsin2526/) - For his [CellOCPico](https://github.com/aomsin2526/CellOCPico) project and spearheading the idea for this to be possible without requiring external hardware
- [RIP Felix](https://www.youtube.com/@ripfelix3020) - Attempting different values for the CELL clock generator registers
- [villahed94](https://www.youtube.com/@villahed94/) - Attempting different values for the CELL clock generator registers
- [Sampsonay](https://www.youtube.com/@Sampsonay/) - Attempting different values for the CELL clock generator registers
- gypsy - Attempting different values for the CELL clock generator registers
- [RGBeter](https://x.com/RGBeter32X) - Attempting different values for the CELL clock generator registers
- [sage](https://codeberg.org/derg/) - This document and extensive reverse engineering of the syscon firmware
---
