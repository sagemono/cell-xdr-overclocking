# PS3 Cell/XDR Overclocking and Memory Timing Research

Reverse engineering of the PlayStation 3's clock generation, XDR memory timing control, and timebase systems. This research enables software-based CPU overclocking and XDR DRAM timing modification on retail PS3 consoles through Syscon NVS modification.

---

## **Disclaimer:** This is not a guide. This is experimental research that may cause irreversible damage to your system. There is a team working behind this and it is still in the experimental stage. We are not responsible for any damaged systems. If you're following along, you're doing so at your own risk.

## Please be aware that this is currently only functional on CECH-2001A PS3 models and prior (and there may be exceptions even among those, which we are currently investigating). Please also note that, depending on Cell process node (90nm, 65nm, 45nm), different overlock ceilings will be achievable, and in most cases, overvolting will be required to achieve them. If you're following along, you're doing so at your own risk.

---

## Hardware

- **Console Model**: CECHA00 and CECHB00 with Samsung XDR
- **XDR DRAM**: Elpida EDX5116ACSE-3C-E (256Mbit, x16, 8 banks, 3.2G C-bin), Samsung series

---

# Part 1: Cell/XDR Clock Overclocking

## Clock Architecture

### Hardware Components

The COK-001 motherboard uses the following clock generation hardware:

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
│ IC5002/4 - ICS9218AGLFT     │ │ IC5003 - ICS9218AGLFT           │
│ XDR Clock Generator         │ │ Cell Clock Generator            │
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
│           XDR DRAM          │ │             CELL BE             │
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
(Mullion)
| NVS Offset | Size | Device | Register | Default | Description |
|------------|------|--------|----------|---------|-------------|
| 0x3122 | 1 | Master Osc | 0 | 0x20 | Master oscillator config (bits 6-4 select frequency) |
| 0x3128 | 1 | IC5002/4 | 5 | 0x84 | XDR clock generator register 5 |
| 0x3129 | 1 | IC5002/4 | 6 | 0x16 | XDR clock generator register 6 |
| 0x312C | 1 | IC5003 | 5 | 0x84 | CELL clock generator register 5 |
| 0x312D | 1 | IC5003 | 6 | 0x16 | CELL clock generator register 6 |

(Sherwood)
| NVS Offset | Size | Register | Default | Description |
|------------|------|----------|---------|-------------|
| 0x61 | 1 | 5 | 0x84 | XDR clock generator register 5 |
| 0x62 | 1 | 6 | 0x16 | XDR clock generator register 6 |
| 0x63 | 1 | 5 | 0x84 | CELL clock generator register 5 |
| 0x64 | 1 | 6 | 0x16 | CELL clock generator register 6 |

### Default Value Behavior

When NVS contains 0xFF (erased/unset), syscon applies hardcoded defaults:

```c
// From sub_2DC9E - Clock Configuration Function
nvs_read(0x3122, v5, 1);
if (v5[0] == 0xFF) v5[0] = 0x20;  // Default master osc config

nvs_read(0x3128, v5, 1);
if (v5[0] == 0xFF) v5[0] = 0x84;  // Default XDR reg5

nvs_read(0x3129, v5, 1);
if (v5[0] == 0xFF) v5[0] = 0x16;  // Default XDR reg6 (22 decimal)

nvs_read(0x312C, v5, 1);
if (v5[0] == 0xFF) v5[0] = 0x84;  // Default CELL reg5

nvs_read(0x312D, v5, 1);
if (v5[0] == 0xFF) v5[0] = 0x16;  // Default CELL reg6
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
            // Initialize XDR clock generator
            LOBYTE(v5[0]) = 0;
            result = sub_38614(&cell_device_slot, 0, 3, 0);  // Init register 0
            
            if (!result) {
                // Configure XDR reg5 (NVS 0x3128)
                result = nvs_read(0x3128, v5, 1);
                if (!result) {
                    if (LOBYTE(v5[0]) == 0xFF)
                        LOBYTE(v5[0]) = 0x84;  // Default
                    result = sub_38614(&cell_device_slot, 5, 0xFF, LOBYTE(v5[0]));
                    
                    if (!result) {
                        // Configure XDR reg6 (NVS 0x3129)
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
int sub_2CEA4()
{
    // read enable flag from nvs 0x3958
    nvs_read(14680, v10, 1);
    
    if (v10[0] != 0xFF)  // custom pll enabled
    {
        // now wait for cell attention signal
        v1 = sub_2ADFA(v13);  // < fails here.
        if (v1)
        {
            log_printf("[POWERSEQ] Error : wait attention timeout.(%s)\n", "Psbd_SetBePll");
            return v1;
        }
        
        // read 8 bytes of pll config from nvs 0x3950
        nvs_read(14672, v10, 8);
        
        // default if empty
        if (v10[0] == 0xFF)
        {
            // stock 400MHz values: 71 67 06 81 6A 4C 00 00
            qmemcpy(v10, "qg", 2);  // 0x71, 0x67
            v10[2] = 0x06;
            v10[3] = 0x81;
            strcpy(v11, "jL");      // 0x6A, 0x4C
            v11[3] = 0;
        }
        
        // bit interleave 8 bytes -> 9 bytes, some obfuscation bs
        // ... transformation loop ...
        
        // send to cell via spi command
        sub_2AB80(v13, 32, v9, 9);
        
        // trigger update
        v12[0] = 0;
        v12[1] = 0x8000000;
        sub_2AC08(v13, 0, v12);
    }
    return 0;
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


# Part 2: Cell BE Clock Domains and Timebase

## 1. The Three Clock Domains

The Cell BE processor operates with three independent, asynchronous clock domains. A more in-depth view can be read [here](https://github.com/sagemono/cell-xdr-overclocking/blob/main/cell_be_clock_domains_reference.md) (outdated)

**Source: Handbook, Section 13.2.1 "Clock Domains", Page 375:**

> The CBE processor has three clock domains, each running asynchronously to the other two domains:
>
> - **CBE Core Clock (NClk)** - This clock times the PowerPC processor unit (PPU), the SPUs, and parts of the PowerPC processor storage subsystem (PPSS) and the memory flow controllers (MFCs). The CBE core clock (NClk) is occasionally referred to as the core clock (CORE_CLK) or core clock frequency (CCF).
> - **MIC Clock (MiClk)** - This clock times the memory interface controller (MIC).
> - **BIC Core Clock (BClk)** - This clock times the bus interface controller (BIC), which is part of the Cell Broadband Engine interface (BEI) unit to the I/O interface.

This is critical: these three domains are *asynchronous to each other*. Changing one does not inherently change the others.

### 1.1 NClk/2 Sub-Domain

Several logic blocks run at half the core clock frequency. From the same section (Handbook p375-376):

> The following CBE logic blocks run as half the CBE core clock frequency (NClk/2):
>
> - Element interconnect bus (EIB) and interfaces to the EIB (parts of the PPSS and MFCs)
> - I/O Interface controller (IOC)
> - MIC logic that is part of the CBE Core
> - BIC logic that is part of the CBE Core
> - Pervasive logic, which is the logic that provides power management, thermal management, clock control, software-performance monitoring, trace analysis, and so forth

Note the distinction: *BClk* (the BIC's own clock domain) and *BIC logic that is part of the CBE Core* (which runs at NClk/2) are different things. Some BIC interface logic on the core side runs at NClk/2 to bridge between the NClk and BClk domains.

### 1.2 Timebase Lives in NClk Domain

**Source: Handbook, Section 13.2.1, Page 376** (the exact quote, appearing between the NClk/2 bullet list and Section 13.2.2):

> The time-base and decrementer facilities described in this section fall entirely within the CBE core clock (NClk) domain.

This statement is on page 376 of the Handbook, immediately after the pervasive logic bullet point and before the "13.2.2 Time-Base Registers" heading. It directly confirms that the timebase mechanism, including all decrementers (PPE DEC0, DEC1, HDEC, WDEC, and 8 SPE decrementers), are clocked exclusively from NClk.

Additionally, **Handbook Section 13.1, Page 375** opens with:

> The Cell Broadband Engine (CBE) time-base facility provides the timing functions for the CBE core-clock (NClk) domain.

And **HIG Glossary, Page 221** defines:

> time base: This is the Cell BE-processor facility that provides the timing functions for the Cell BE core-clock (NClk) domain.

Three independent sources within the documentation all confirm the same thing.

---

## 2. FlexIO and Its Clock

### 2.1 Does FlexIO Have Its Own Clock?

Yes. FlexIO has its own PLL with its own reference clock, separate from the Core PLL.

**Source: HIG, Section 1.2, Pages 27-28, Figure 1-3 "Cell BE Clock Domains in Typical Operation":**

The HIG identifies three separate PLLs on the Cell BE die:

| PLL | Reference Input | Multiplier | Output | Drives |
|---|---|---|---|---|
| **Core PLL** | PLL_REFCLK (400 MHz) | 1:8 | NClk (3.2 GHz) | PPU, SPUs, pervasive, EIB, timebase |
| **XIO PLL** | Y0_RQ_CTM / Y1_RQ_CTM (400 MHz) | 1:4 | XIO Clk / MiClk (1.6 GHz) | Memory Interface Controller (XDR) |
| **FlexIO PLL** | RC_REFCLK (500 MHz) | 1:5 | RO/TO Clk (2.5 GHz) | FlexIO data lanes, BClk |

The HIG text preceding Figure 1-3 states:

> Figure 1-3 on page 27 shows the clock domains in typical operation. The frequency numbers used in this figure are meant as an example only. For actual frequencies supported on the Cell BE processor and for specifications for the three phase-locked loop (PLL) clock inputs, see the Cell Broadband Engine Datasheet.
>
> The following terms are used for the PLL reference clocks and clock multipliers:
> - Core PLL reference clock (PLL_REFCLK). The multiplier for the PLL_REFCLK is called the core clock multiplier in the Cell Broadband Engine Programming Handbook.
> - XIO PLL reference clock per channel (Y0_RQ_CTM, Y1_RQ_CTM).
> - FlexIO PLL reference clock (RC_REFCLK).

### 2.2 Is FlexIO's Clock Relevant to Timebase?

No. The timebase depends solely on PLL_REFCLK and the Core PLL output (NClk). The FlexIO PLL (driven by RC_REFCLK) produces the BClk domain and the FlexIO data clocks (RO/TO). These are on a completely separate clock tree.

From the HIG Figure 1-3 (page 27-28), the typical stock frequencies are:

```
                    PLL_REFCLK (400 MHz)                 RC_REFCLK (500 MHz)
                          |                                     |
                    [Core PLL x8]                        [FlexIO PLL x5]
                          |                                     |
                    NClk = 3.2 GHz                     RO/TO Clk = 2.5 GHz
                          |                                     |
                  NClk/2 = 1.6 GHz                       BClk = 1.667 GHz
                (EIB, IOC, pervasive)                    (BIC core domain)
                          |
               Timebase divider (LFSR)
                          |
            TB freq = PLL_REFCLK / RefDiv
```

FlexIO's clock determines the data rate of the I/O bus between Cell and RSX (and other IOIF devices). It has no involvement in the timebase frequency calculation whatsoever.

### 2.3 Is FlexIO Relevant to Overclocking at All?

Yes, but only as a possible stability constraint.

> The CELL processor communicates with XDR memory through the FlexIO interface. If the frequency difference between CELL and XDR exceeds approximately 800 MHz, the interface becomes unstable and fails during link initialization.

This "800 MHz rule" means that when you raise the Cell core clock via PLL_REFCLK, the FlexIO bus still needs to maintain a viable link. The FlexIO bus itself doesn't change frequency (it has its own PLL), but the asynchronous boundary logic between NClk and BClk domains has timing margins that narrow when the frequency delta grows.

---

## 3. The Timebase Mechanism in Detail

### 3.1 Core Formula

**Source: Handbook, Section 13.2.3 "Time-Base Frequency", Page 377:**

> In the internal time-base sync mode, the CBE core clock (NClk) frequency is:
>
> NClk = PLL_REFCLK x CCM
>
> [...]
>
> The standard 3.2 GHz CBE core clock (NClk) is configured by specifying the Internal Time-Base Sync State, a 400 MHz PLL reference clock (PLL_REFCLK) frequency, and a core clock multiplier (CCM) of x8.

The timebase frequency itself is:

```
timebase_frequency = PLL_REFCLK / RefDiv
```

Where RefDiv is a reference clock divider selected via the TBR[Timebase_setting] LFSR table.

### 3.2 The TBR Register

**Source: Registers, Section 11.6.17 "Time Base Register (TBR)", Page 286:**

| Field | Bits | Description |
|---|---|---|
| Reserved | 0:54 | All bits read back zero |
| Timebase_mode | 55 | 0 = Internal sync, 1 = External sync |
| Timebase_setting | 56:63 | LFSR-based reference clock divider. 0x00 = disabled (POR default) |

MMIO Address: `BE_MMIO_Base + 0x509890`

Access: Privilege 1, Read/Write

### 3.3 RefDiv LFSR Mapping (Table 13-1)

**Source: Handbook, Section 13.2.4.1, Pages 379-381, Table 13-1**

The mapping from Timebase_setting byte to actual RefDiv is non-linear (LFSR-based). The complete table has 255 entries. Key values for overclocking:

| RefDiv | Timebase_setting | TB freq at 400 MHz | TB freq at 450 MHz | TB freq at 475 MHz | TB freq at 500 MHz |
|---|---|---|---|---|---|
| 5 | 0x4F | 80.00 MHz | 90.00 MHz | 95.00 MHz | 100.00 MHz |
| 6 | 0x27 | 66.67 MHz | 75.00 MHz | 79.17 MHz | 83.33 MHz |
| 7 | 0x13 | 57.14 MHz | 64.29 MHz | 67.86 MHz | 71.43 MHz |
| 8 | 0x09 | 50.00 MHz | 56.25 MHz | 59.38 MHz | 62.50 MHz |
| 14 | 0x94 | 28.57 MHz | 32.14 MHz | 33.93 MHz | 35.71 MHz |

Stock uses RefDiv=5, Timebase_setting=0x4F, producing 400/5 = 80 MHz.

### 3.4 Determining Timebase Frequency at Runtime

**Source: Handbook, Section 13.2.4.2 "Determining the System Time-Base Frequency", Page 381:**

> Assuming the time base is configured as internal time-base sync mode, then the frequency can be determined using the following procedure:
>
> 1. Verify that the system is configured in internal time-base sync mode by reading the TBR register and verifying that the TBR[Timebase_mode] field is '0'.
> 2. Determine the RefDiv value by doing a inverse table lookup of Table 13-1 using the value stored in TBR[Timebase_setting].
> 3. Compute the time-base frequency by multiplying the PLL_REFCLK and RefDiv.

And:

> Because the time-base registers are accessible only by privileged software, the operating environment should provide information about the time-base frequency. Consult system documentation for specific details.

This last sentence is key: the OS is supposed to tell userspace what the timebase frequency is. That being syscall 147 (sys_time_get_timebase_frequency). If lv2 reports the wrong value, everything downstream breaks.

### 3.5 Two Timebase Sync Modes

**Internal mode** (standard operation): The timebase derives its tick rate from PLL_REFCLK divided by the LFSR divider. The Cell's internal clock tree handles everything.

**External mode** (TBEN pin): The rising edge of an external signal on the TBEN pin directly drives each tick.

**Source: HIG, Table 5-11 "Miscellaneous I/O Signals", Page 156-157:**

> TBEN: Time base enable. This active-high input is asserted to enable the Cell BE time base function when the Cell BE processor is configured to use an internal time base. When the Cell BE processor is configured to use an external time base, the time base clock is provided on this input and can be from 20 MHz through 286 MHz. The actual external time base maximum frequency depends upon the Cell BE PLL reference frequency and the Cell BE-processor configuration.

### 3.6 Fmax Constraint (Slow State)

The maximum timebase frequency is constrained by the power management slow state setting.

**Source: Handbook, Section 13.2.3, Page 377:**

> RefDiv >= ceil(MaxSlowModeNclkDivider x 11 / CCM)
>
> Fmax (Internal Mode) = PLL_REFCLK / RefDiv

With CCM=8 and all slow states enabled (MaxSlowModeNclkDivider=10): RefDiv >= ceil(10 x 11 / 8) = 14, giving Fmax = 400/14 = 28.57 MHz.

The stock 80 MHz timebase (RefDiv=5) is well above this Fmax, which means the firmware does NOT use the full range of slow states, or the timebase frequency was chosen to not be constant across slow state transitions. The Handbook explicitly states (page 377-378):

> The maximum time-base frequency limit guarantees a constant time-base frequency during functional operation of the processor (including during all CBE slow states). The time base should be set to a frequency below this limit.

---

## 4. The CBEA's Perspective (Architecture vs Implementation)

The CBEA is an *architecture specification*, it defines what a compliant processor must look like to software but deliberately leaves implementation details open.

**Source: CBEA, Section 1.1, Page 23:**

> An element interconnect bus (EIB) connects the various units within the processor. The requirements for the MIC, BIC, and EIB vary widely between implementations. Thus, the definition for these units is beyond the scope of the CBEA.

The CBEA does not define clock domains, PLLs, or the BIC/FlexIO interface at all. These are entirely implementation-specific. The Handbook and HIG describe the *specific 90nm Cell BE implementation* used in PS3 and blade servers.

The only CBEA statement about timebase rate is in the SPU decrementer section:

**Source: CBEA, Section 9.7 "SPU Decrementer", Page 139:**

> Note: The requirement for a constant rate for the time base and the decrementer is an additional requirement beyond the PowerPC Architecture. However, all SPU decrementers are required to run at the same rate as the PPE decrementer, but there is no requirement that the decrementers be synchronized.

This tells us the architecture mandates a *constant* tick rate and that all 11 decrementers (3 PPE + 8 SPE) tick at the same frequency, but says nothing about clock domains because those are implementation-specific.

---

## 5. Cell Core PLL Multiplier (CCM)

### 5.1 The Multiplier is Fuse-Programmed

The Cell core PLL multiplier (CCM) is set from internal fuses during Phase 1 of Power-On Reset. It is controlled by the SYS_CONFIG[0:3] pins, which are tied to ground on all retail PS3 motherboards. This gives a fixed 8x multiplier: 400 MHz * 8 = 3.2 GHz.

**Source: HIG, Section 2.2, POR Phase 1:**

> The PLL configuration register will be set up from internal storage (fuses) as a result of the SYS_CONFIG[0:3] pins being set to '0000'. The PLL configuration register is an internal register that is not accessible in the MMIO register space.

The multiplier is NOT in the config ring, NOT in MMIO, and NOT modifiable via NVS. On retail hardware, the only way to change Cell clock speed is by changing PLL_REFCLK via the external clock generator. Changing the external clock from 400 to 500 MHz pushes NClk from 3.2 GHz to 4.0 GHz with no way to compensate the multiplier.

### 5.2 DECR BE_PLL Path (Development Hardware Only)

On DECR-1000A development hardware, there is a second path that sends configuration bytes directly to the Cell's internal PLL via the SPI scan chain, accessed through the `Psbd_SetBePll` function in the syscon firmware:

```
Path 1 (Retail): NVS 0x3128/0x3129 -> I2C -> IC5003 -> PLL_REFCLK pin -> Cell PLL x8
Path 2 (DECR):   NVS 0x3070-0x3077 -> SPI -> Cell internal PLL configuration register
```

The 8-byte PLL configuration is bit-rearranged into 9 bytes and sent via SPI command to the Cell's internal scan chain. This path is confirmed functional on DECR hardware (4.8 GHz achieved). However, the exact bit layout of the 8-byte PLL configuration is proprietary IBM/Sony PLL IP and is not documented in any public Cell specification. Whether these bytes change the PLL multiplier, the feedback divider, the VCO range, or some combination is unknown.

This path fails on retail hardware because the `ATTENTION` handshake signal (Cell telling syscon it's ready to receive PLL commands) does not assert.

### 5.3 The 0x84 0x16 Values

These are clock generator register 5 and register 6 values for the ICS9218AGLFT chip, NOT Cell PLL multiplier settings:

```
XCG2 BE 5:84 6:16  <- NVS 0x3068-0x3069 on DECR, 0x3128-0x3129 on retail
```

From the syscon frequency lookup table, the combined key `0x8416` maps to 400,000,000 Hz output from the clock generator.

---

## 6. What Changes When You Overclock PLL_REFCLK

When you change PLL_REFCLK from 400 MHz to (say) 450 MHz, the following things scale proportionally:

| Component | Stock (400 MHz) | OC (450 MHz) | Source |
|---|---|---|---|
| NClk | 3.2 GHz | 3.6 GHz | NClk = PLL_REFCLK x 8 (Handbook p377) |
| NClk/2 (EIB, pervasive) | 1.6 GHz | 1.8 GHz | Handbook p375 |
| Timebase frequency | 80 MHz | 90 MHz | PLL_REFCLK / RefDiv, RefDiv=5 (Handbook p377) |
| All 11 decrementers | 80 MHz tick rate | 90 MHz tick rate | CBEA p139: "same rate as PPE decrementer" |

The following things do NOT change (separate clock domains):

| Component | Frequency | Why |
|---|---|---|
| MiClk (XDR memory) | 1.6 GHz | Separate PLL (XIO PLL), separate reference (Y_RQ_CTM) |
| BClk (BIC/FlexIO) | 1.667 GHz | Separate PLL (FlexIO PLL), separate reference (RC_REFCLK) |
| FlexIO data rate | 2.5 GHz | Driven by RC_REFCLK, not PLL_REFCLK |
| RSX clocks | Various | Completely independent chip |

---

## 7. Why Overclocking Breaks Timing

The root cause: lv1 stores and reports the stock timebase frequency (79,800,000 Hz or 80,000,000 Hz) to GameOS. When you overclock PLL_REFCLK but don't update either the hardware divider or the reported frequency, every piece of software that converts timebase ticks to wall-clock time gets the wrong answer.

Game calls `sys_time_get_timebase_frequency()` -> lv1 returns 80,000,000 -> game divides tick count by 80,000,000 to get seconds -> but ticks are actually arriving at 90,000,000/sec -> every "second" of game time is actually 0.889 real seconds -> FPS counter shows 11.2% fewer frames per measured "second."

The fix approaches are:

**A. Static lv0 patch (working):** Patch the hardcoded `lis r5, 0x04C1` / `ori r5, r5, 0xA6C0` instruction pair in `build_clock_config_string` to load the correct tb_clk value for the target overclock frequency. Only modifies 4 bytes (2 bytes in lis immediate, 2 bytes in ori immediate). See `patch_lv0_tb_clk_static.py`. This is the current recommended approach.

**B. Dynamic lv0 patch (working with limitations)**

**C. Runtime patch (working):** Scan lv1 memory at boot or runtime for the `be.0.tb_clk` repo hash table entry and overwrite the value with the correct frequency. Immediate effect on GameOS syscall 147. See Part 3 Section 10 for implementation details.

---

## 8. PS2 Emulation and Timebase

PS2 emulators run as Guest OS directly on lv1, with lv2 (GameOS) completely unloaded. A syscall 147 hook or lv2 memory patch does nothing for PS2 emulation because lv2 is not running.

The PS2 SPU2 emulator (which runs as an SPE program) uses the SPE decrementer (channel 8) to pace audio output at 48 kHz. Two constants are hardcoded for the stock 79.8 MHz timebase:
- `dec_count = 1662` (= 79,800,000 / 48,000) - sample period in ticks
- `dma_timeout = 798` (= 79,800,000 / 100,000) - 10us DMA timeout (ps2_emu only)

When overclocked, the timebase runs faster but these constants remain unchanged, causing audio to play too fast with crackling artifacts.

**Fix status:**
- **ps2_emu.self** (COK-001 boards with hardware EE+GS): **Fixed.** The SPU2 timing constants are patched in the decrypted base ELF to match the actual timebase frequency. See `patch_ps2emu_spu2_timing.py`.
- **ps2_gxemu.self** (COK-002 boards with hardware GS only): **Not fixed.** Patching the binary causes a Green Light of Death when loaded. Under investigation.
- **ps2_netemu.self** (full software emulation): **Not fixed.** Same GLOD issue as gxemu. Under investigation.


# Part 3: lv0/lv1 Timebase Bug Analysis

## 1. Summary

**tb_clk is hardcoded in lv0.** The value 79,800,000 is a literal constant at `0x8006EF0` in lv0, never computed from the actual reference clock. Meanwhile, `be.0.nclk` and `be.0.ref_clk` are dynamically queried from syscon. This is the root cause of the timing bug after overclocking: the hardware timebase physically changes frequency but lv0 always reports 79.8 MHz to lv1.

The TBR hardware register IS configured dynamically (lv0 computes RefDiv from the actual ref_clk), so the hardware ticks at the correct rate. The problem is purely in the reported metadata.

**This bug has been fixed** via `patch_lv0_tb_clk_static.py`, which patches the hardcoded literal to match the target overclock frequency. A runtime Cobra patch that overwrites the lv1 repo hash table entry is also working for GameOS. See Section 11 for full patch status.

---

## 2. Root Cause

In `build_clock_config_string` (lv0 @ `0x8006E3C`):

```c
ref_clk = get_reference_clock();              // DYNAMIC: queries syscon, e.g. 399,000,000
ccm     = get_core_clock_multiplier();        // DYNAMIC: queries syscon, e.g. 8
config_append("be.0.nclk",    ref_clk * ccm); // DYNAMIC: 3,192,000,000
config_append("be.0.ref_clk", ref_clk);       // DYNAMIC: 399,000,000
config_append("be.0.tb_clk",  79800000LL);    // HARDCODED LITERAL <-- BUG
```

After hardware OC to 475 MHz (3.8GHz):
- `ref_clk` becomes ~473,812,500 (475M with 0.25% crystal correction)
- `nclk` becomes ~3,790,500,000 (ref_clk * 8)
- `tb_clk` stays 79,800,000 (frozen constant)
- Actual TBR tick rate = 473,812,500 / 5 = 94,762,500 Hz
- Reported rate = 79,800,000 Hz
- Timing error = **18.7%** (every "60 seconds" of game time takes ~50.5 real seconds)

---

## 3. Complete Data Flow

```
BOOT SEQUENCE:

    hardware             syscon                 lv0                  lv1                 lv2/guest
       |                   |                     |                    |                     |
  25 MHz xtal ----------> PLL(?) ---> SPI --> report ref_clk          |                     |
       |                   |           |         |                    |                     |
       |                   |           |    get_reference_clock()     |                     |
       |                   |           |    = syscon_spi(3,0x10)      |                     |
       |                   |           |    -> 400M * 399/400         |                     |
       |                   |           |    = 399,000,000             |                     |
       |                   |           |         |                    |                     |
       |                   |           |    get_core_clock_mult()     |                     |
       |                   |           |    = syscon_spi(3,0x00)      |                     |
       |                   |           |    -> id=3 -> 8x             |                     |
       |                   |           |         |                    |                     |
       |                   |           |    build config string:      |                     |
       |                   |           |      nclk    = 3192000000    |                     |
       |                   |           |      ref_clk = 399000000     |                     |
       |                   |           |      tb_clk  = 79800000      |  <-- HARDCODED      |
       |                   |           |         |                    |                     |
       |                   |           |    configure_timebase_reg()  |                     |
       |                   |           |      refdiv = ref_clk / 79800000 = 5               |
       |                   |           |      LFSR_table[5] = 0x4F    |                     |
       |                   |           |      write 0x4F -> TBR MMIO  |                     |
       |                   |           |         |                    |                     |
       |                   |           |         |--- config str ---> |                     |
       |                   |           |         |                    |                     |
       |                   |           |         |              repo_init()                 |
       |                   |           |         |              parse config -> hash table  |
       |                   |           |         |                    |                     |
       |                   |           |         |                    |-- hcall 0x100C1 ->  |
       |                   |           |         |                    |   "sys.be.0.tb_clk" |
       |                   |           |         |                    |   -> 79800000    -> |
       |                   |           |         |                    |                     |
       |                   |           |         |                    |                 lv2 caches
       |                   |           |         |                    |                 syscall 147
```

---

## 4. lv0 Identified Functions

### 4.1 Clock Config Builder

| Address | Name | Description |
|---|---|---|
| 0x8006E3C | `build_clock_config_string` | Builds config entries for all clock/platform values. Passes to lv1 as boot params |
| 0x8006D54 | `config_string_append_entry` | Appends "key=value" pair to config string buffer |

Key lines in `build_clock_config_string`:
```
0x8006EAC: append("be.0.nclk",    ref_clk * multiplier)   // DYNAMIC
0x8006EBC: append("be.0.ref_clk", ref_clk)                // DYNAMIC
0x8006EF0: append("be.0.tb_clk",  79800000)               // HARDCODED
```

### 4.2 Clock Query Functions

| Address | Name | Description |
|---|---|---|
| 0x8003BFC | `get_reference_clock` | Returns ref_clk in Hz. Primary: syscon SPI. Fallback: EEPROM flag * 10M. Default: 400M or 600M |
| 0x8003C7C | `get_core_clock_multiplier` | Returns CCM integer. Primary: syscon SPI. Fallback: EEPROM flag. Default: 4 or 6 |

### 4.3 Syscon Interface

| Address | Name | Description |
|---|---|---|
| 0x800B16C | `syscon_get_ref_clock` | Wrapper: `syscon_spi_query(out, 3, 0x10, apply_correction)` |
| 0x800B338 | `syscon_get_multiplier` | Sends cmd {3, 0}, maps response byte to {2,4,6,8,10,12,16,20} |
| 0x800B074 | `syscon_spi_query` | Low-level SPI command. When a4=1, applies crystal correction: `value -= value/400` (0.25% reduction) |

### 4.4 SC EEPROM Fallback Readers

| Address | Name | Reads | SC EEPROM Offset |
|---|---|---|---|
| 0x8002ED4 | `read_eeprom_ref_clk_flag` | `*(region3_base + 1)` = be_nclck_flag2 | 0x48C23 |
| 0x8002F20 | `read_eeprom_nclk_flag` | `*(region3_base + 0)` = be_nclck_flag1 | 0x48C22 |

Usage in fallback path:
- If syscon query fails, read EEPROM flag byte
- `ref_clk = flag2 * 10,000,000` (e.g. flag2=40 -> 400 MHz)
- `multiplier = flag1` directly (e.g. flag1=8 -> 8x)
- If flag is 0xFF (retail default), skip to hardcoded defaults

### 4.5 TBR Register Configuration

| Address | Name | Description |
|---|---|---|
| 0x800A600 | `configure_timebase_register` | Computes RefDiv from actual ref_clk, writes LFSR byte to TBR MMIO at BE_MMIO+0x509890 |

Key logic:
```c
ref_clk = get_reference_clock();          // e.g. 399,000,000
refdiv_index = ref_clk / 79800000;        // stock: 5 (integer division)
if (refdiv_index > 255) ERROR();
lfsr_byte = LFSR_TABLE[refdiv_index];     // table[5] = 0x4F
*(BE_MMIO + 0x509890) = lfsr_byte;        // write to TBR register
```

### 4.6 LFSR Lookup Table (0x8018808)

256-byte table mapping RefDiv index to LFSR Timebase_setting values. Matches Cell BE Handbook Table 13-1.

| Index | LFSR Byte | Meaning |
|---|---|---|
| 0 | 0x00 | Disabled |
| 1 | 0xFF | RefDiv=1 |
| 2 | 0x7F | RefDiv=2 |
| 3 | 0x3F | RefDiv=3 |
| 4 | 0x9F | RefDiv=4 |
| **5** | **0x4F** | **RefDiv=5 (stock)** |
| **6** | **0x27** | **RefDiv=6 (OC ~500 MHz)** |
| 7 | 0x13 | RefDiv=7 |
| 8 | 0x09 | RefDiv=8 |
| 9 | 0x84 | RefDiv=9 |
| 10 | 0x42 | RefDiv=10 |

---

## 5. Crystal Tolerance Correction (the 0.25% factor)

`syscon_spi_query` applies: `value -= value / 400` when the correction flag is set (a4=1).

This reduces the syscon-reported value by 0.25%, accounting for the 25 MHz crystal oscillator running slightly slow:

```
Syscon reports:  400,000,000 Hz
After correction: 400,000,000 - 1,000,000 = 399,000,000 Hz
Divided by 5:    399,000,000 / 5 = 79,800,000 Hz  <-- matches hardcoded tb_clk exactly
```

Sony calculated 79,800,000 once using this formula and baked it into lv0 as a constant (lol).

The correction is applied to `get_reference_clock()` (called with a4=1), which feeds into `nclk` and `ref_clk` config entries dynamically. But tb_clk doesn't benefit because it's never computed.

---

## 6. RefDiv Behavior Under Overclocking

The `configure_timebase_register` function dynamically computes RefDiv using integer division of ref_clk by 79,800,000. This means RefDiv can change at certain OC thresholds:

| OC Target     | ref_clk (corrected) | ref_clk / 79.8M | RefDiv | LFSR | Actual TB freq | Reported | Error |
|---------------|---------------------|------------------|--------|------|----------------|----------|-------|
| Stock 400M    | 399,000,000         | 5.000            | 5      | 0x4F | 79,800,000     | 79.8M    | 0%    |
| 425 MHz       | 423,937,500         | 5.313            | 5      | 0x4F | 84,787,500     | 79.8M    | 6.2%  |
| 450 MHz       | 448,875,000         | 5.625            | 5      | 0x4F | 89,775,000     | 79.8M    | 12.5% |
| 475 MHz       | 473,812,500         | 5.937            | 5      | 0x4F | 94,762,500     | 79.8M    | 18.7% |
| 478.8 MHz     | 477,603,000         | 5.985            | 5      | 0x4F | 95,520,600     | 79.8M    | 19.7% |
| 479.4 MHz     | 478,201,500         | 5.993            | 5      | 0x4F | 95,640,300     | 79.8M    | 19.8% |

RefDiv boundary: `ref_clk >= 479,400,000 (corrected)` ≈ `480,600,000 (raw)` triggers `RefDiv = 6`

| OC Target | ref_clk (corrected) | ref_clk / 79.8M | RefDiv | LFSR | Actual TB freq | Reported | Error |
|-----------|---------------------|------------------|--------|------|----------------|----------|-------|
| 480 MHz   | 478,800,000         | 6.000            | 6      | 0x27 | 79,800,000     | 79.8M    | 0%!   |
| 500 MHz   | 498,750,000         | 6.250            | 6      | 0x27 | 83,125,000     | 79.8M    | 4.2%  |
| 550 MHz   | 548,625,000         | 6.875            | 6      | 0x27 | 91,437,500     | 79.8M    | 14.6% |

RefDiv boundary at ~559.2 MHz corrected -> RefDiv = 7

Notable: at exactly 480 MHz (corrected = 478,800,000), ref_clk / 79,800,000 = exactly 6, and 478,800,000 / 6 = 79,800,000. The timing error accidentally becomes zero because the new RefDiv produces the same timebase frequency as stock. This is a coincidence of the integer division. Though, the closest usable `ref_clk` to 480 is 483 (3.9GHz)

---

## 7. lv1 Repository Hash Table Structure

### 7.1 Layout

```
Offset 0x00:   [16 bytes]  header (2x 64-bit metadata fields)
Offset 0x10:   [40 bytes]  entry 0
Offset 0x38:   [40 bytes]  entry 1
...
Offset 0x1400: [40 bytes]  entry 127
Total: 16 + (128 * 40) = 5,136 bytes
```

### 7.2 Entry Format (40 bytes)

```
+0x00: [32 bytes] key name (null-terminated ASCII)
+0x20: [8 bytes]  value (64-bit integer)
```

### 7.3 Hash Function

```
hash = (sum of all ASCII bytes in key) & 0x7F
```

Collision resolution: linear probing (increment hash mod 128).

### 7.4 Hash Values for Clock Nodes

| Key | Byte Sum | Hash (mod 128) |
|---|---|---|
| `be.0.tb_clk` | 962 | 66 |
| `be.0.nclk` | 895 | 127 |
| `be.0.clock` | 924 | 28 |
| `be.0.ref_clk` | 987 | 91 |

---

## 8. lv1 Identified Functions

### 8.1 Repository Infrastructure

| Address | Name | Description |
|---|---|---|
| 0x2BADBC | `get_repo_base` | Returns global repo hash table pointer |
| 0x2BACB4 | `repo_read` | Hash lookup: key -> 64-bit value |
| 0x2BADC4 | `repo_write` | Hash insert (only caller: config parser) |
| 0x2BAEF8 | `repo_parse_config_string` | Parses "key=value\n" text, calls repo_write |
| 0x2BB658 | `repo_init` | Allocates 5136-byte table, parses lv0 params then defaults |

### 8.2 Timebase Consumers (12 functions reference tb_clk)

| Address | Name | Operation |
|---|---|---|
| 0x209AC0 | `timebase_init` | tb_clk/1M -> MHz. Default 80. Calls be_hardware_init. |
| 0x20964C | `be_hardware_init` | Uses tb_freq for mftb-based hardware delays |
| 0x2626A0 | `lpar_boot_init` | tb_clk/1M at LPAR+208. Default 16. |
| 0x27496C | `lpar_tb_freq_init` | tb_clk/1M at LPAR+6520. Default 1. |
| 0x2745F4 | `lpar_timer_init` | tb_clk/1M at LPAR+5768. Default 16. |
| 0x27A94C | `lpar_context_init` | tb_clk at LPAR+5768. |
| 0x26E274 | `timer_subsystem_init` | tb_clk/1M at +7664. Computes 16ms timer ticks. |
| 0x2F5F18 | `timer_freq_setup` | tb_clk/1K at +16/+32, tb_clk/500K at +24 |
| 0x3798C0 | `timebase_subsystem_init` | Caches raw Hz globally. Overrides to 2.7M on DECR. |
| 0x2C268C | `cell_be_hw_config_from_repo` | Writes repo values to Cell MMIO registers |

### 8.3 Time Tick Utilities

| Address | Name | Returns |
|---|---|---|
| 0x2B9A84 | `get_tb_freq_div100k` | tb_clk/100000 (798 for stock) |
| 0x2B9AFC | `get_time_ticks_div100k` | TBR / (100000 * cached) = seconds |
| 0x2B9BB0 | `get_time_ticks_div100` | TBR / (100 * cached) = milliseconds |
| 0x2B9C44 | `get_time_ticks_div10` | TBR / (10 * cached) = 100us units |

---

## 9. SC EEPROM Clock Flags

The flags at 0x48C22/0x48C23 are now fully understood:

**be_nclck_flag1 (0x48C22):** Fallback core clock multiplier. Used by `get_core_clock_multiplier()` when syscon query fails. Byte value used directly as multiplier (e.g. 8 = 8x). 0xFF = invalid/skip.

**be_nclck_flag2 (0x48C23):** Fallback reference clock. Used by `get_reference_clock()` when syscon query fails. `ref_clk = flag * 10,000,000` (e.g. 40 = 400 MHz). 0xFF = invalid/skip.

On retail consoles both are 0xFF, so the syscon primary path always takes precedence. These flags only matter if syscon communication fails, which doesn't happen under normal conditions. They do NOT provide a mechanism to override the hardcoded multiplier and `ref_clk`.

---

## 10. Runtime Patching Strategy

### 10.1 What Needs Patching

After hardware OC, the correct tb_clk value is:

```
new_tb_clk = corrected_ref_clk / floor(corrected_ref_clk / 79800000)

Where corrected_ref_clk = raw_ref_clk * 399/400

Examples (assuming exact clock generator output):
  475 MHz raw -> 473,812,500 corrected -> /5 = 94,762,500
  500 MHz raw -> 498,750,000 corrected -> /6 = 83,125,000
  480 MHz raw -> 478,800,000 corrected -> /6 = 79,800,000 (patch not needed in this case, but the closest `ref_clk` to 480 is 483 as stated before)
```

### 10.2 Scan and Patch via lv1 Peek/Poke

Scan for the key string bytes in lv1 memory, then patch the value at a fixed offset:

```c
// scan for "be.0.tb_" at 8-byte aligned addresses
// full key is "be.0.tb_clk\0" = 62 65 2E 30 2E 74 62 5F 63 6C 6B 00
uint64_t sig1 = 0x62652E302E74625FULL;  // "be.0.tb_"
uint64_t sig2 = 0x636C6B0000000000ULL;  // "clk\0\0\0\0\0"

for (uint64_t addr = 0x300000; addr < 0x1000000; addr += 8) {
    if (lv1_peek(addr) == sig1) {
        uint64_t next = lv1_peek(addr + 8);
        if ((next >> 40) == 0x636C6B) {  // starts with "clk"
            // value is at addr + 0x20 (32 bytes after key start)
            uint64_t current = lv1_peek(addr + 0x20);
            if (current == 0x04C1A6C0ULL) {  // 79,800,000
                lv1_poke(addr + 0x20, new_tb_clk);
            }
        }
    }
}
```

### 10.3 Also Patch Cached Globals

lv1's time tick utilities cache `tb_clk / 100000`. After patching the repo node, also scan for:
- Raw Hz value: 0x04C1A6C0 (79,800,000)
- Div100k value: 0x31E (798)

### 10.4 lv2 Cached Value

lv2 reads tb_clk at boot via hypercall and caches it. The runtime patch overwrites all known timebase values with the new one, so the correct value propagates automatically to syscall 147.

### 10.5 Timing

- Patch lv1 repo node at boot: affects all future LPAR constructions and GameOS syscall 147
- Should be done early, ideally from a stage2 payload at boot. Running a patch to scan every mention of the old value and patching it works too.

---

## 11. lv0 Patch Status

### 11.1 Static Patch (Working)

`patch_lv0_tb_clk_static.py` patches the hardcoded 79,800,000 Hz literal in `build_clock_config_string` to the correct value for a given overclock frequency. Only 4 bytes are modified: 2 bytes in the `lis r5, 0x04C1` immediate and 2 bytes in the `ori r5, r5, 0xA6C0` immediate. The patched lv0 must be re-signed and rebuilt into the firmware.

This is the simplest and most reliable fix. The tradeoff is that it hardcodes a specific overclock frequency, changing the overclock speed requires re-patching lv0, thus, requiring a new CFW for each specific overclock. 

### 11.2 Dynamic Patch (Working)

`patch_lv0_tb_clk.py` replaces the hardcoded literal with a patched instructions that calls `get_reference_clock()` and computes `tb_clk = ref_clk / (ref_clk / 79800000)` dynamically.  This would make lv0 automatically report the correct tb_clk at any overclock frequency. This approach has its limitations, RefDiv **cannot** be calculated dynamically as it would require two extra instructions which we simply don't have space for. 

### 11.3 PS2 Emulator SPU2 Patch (Working for ps2_emu)

`patch_ps2emu_spu2_timing.py` patches the SPU2 emulator's hardcoded decrementer constants in the decrypted base ELF. It finds all `il $rN, 1662` (dec_count) and `il $rN, 798` (dma_timeout) SPU instructions using context-aware pattern matching (validates against nearby `rdch ch8` instructions) and replaces them with the correct values for the target overclock frequency.

Works with ps2_emu.self (hardware EE+GS boards). ps2_gxemu.self and ps2_netemu.self cause GLOD and emulator bootloops when patched and loaded. The cause is under investigation.

# Part 4: XDR Memory Timing Research

## How It Works

### Warning: This section is incredibly technical and gets deep into JEDEC DRAM standards and other nerdbabble. This section is mosty a copy and paste from the source documents listed below but rephrased into a more palatable format.

The PS3's XDR memory timing is controlled by the Memory Interface Controller (MIC) inside the Cell BE processor. The MIC is configured via MMIO registers that are programmed by lv0 during boot. The syscon firmware provides a 112-byte config block to lv0 via SERV_SETCFG, built from 46 NVS entries (0x3140-0x31AF) merged with ROM defaults at firmware address 0x402AC.

Any NVS entry left as all-FF uses the ROM default. When an entry has any non-FF byte, the firmware uses the NVS data for that entire entry.

## Source Documents

| Document | Description |
|----------|-------------|
| CellBE HIG 90nm v1.5 (Nov 2007) | Hardware Initialization Guide, Appendix E (MIC config rules) |
| CellBE HIG 65nm v1.01 (Jun 2007) | Slow Core Mode MvWrDelay/MvRdDelay formulas |
| CBE Public Registers v1.5 (Apr 2007) | Section 7: Complete MIC MMIO register bitfield definitions |
| Elpida EDX5116ACSE Datasheet (E0881E20 v2.0) | XDR DRAM timing specs per speed bin |
| CXR713F syscon firmware (reverse engineered) | NVS layout, descriptor table, config ring |

## Confirmed Stable Profile

Confirmed working on both Elpida EDX5116ACSE-3C-E and Samsung series XDR DRAM @ 3.2GHz.

```
w 3160 51 40                # MIC_CMD_DUR: tRC=21, tRAS/tRP tightened
eepcsum
```

### Samsung Benchmark Results (3.7 GHz Cell/XDR, memtest86+ port)

| Config | Read avg | Read min | Write avg | Write min |
|--------|----------|----------|-----------|-----------|
| Stock | 527ns | 162ns | 106ns | 75ns |
| `w 3160 51 40` (tRC=21, tRAS/tRP=0x40) | 442ns | 162ns | 106ns | 75ns |

**16% read latency reduction** from a single 2-byte register write. Zero errors in stress testing.

Minimum read/write latencies are identical because page-hit performance is determined by the MIC dataflow pipeline (MIC_DF_CTL), which is XIO-coupled and cannot be changed. All gains are in the page-miss path (row cycle, precharge, turnaround), where CMD_DUR dominates.

### Isolation Testing: CMD_DUR Is the Only Register That Matters

Systematic isolation testing on Samsung XDR revealed that MIC_CMD_DUR (entry 16) accounts for virtually all measurable read latency improvement. Every other register change from the Tight-3 multi-register profile produced zero measurable standalone impact.

| Change (standalone, tRC=20 base) | Read avg | Delta |
|----------------------------------|----------|-------|
| tRC=20 alone (4D 70) | 461ns | baseline |
| + tRAS/tRP=0x30 (4D 30) | 436ns | -25ns |
| + tRDP=1 (entry 15) | 433ns | -28ns |
| + tDeltaWR=8 (entry 17) | 437ns | -24ns |
| + BurstSize ReadQ=2 (entry 21) | 435ns | -26ns |
| All combined | 433ns | -28ns |

| Change (standalone from full stock 527ns) | Read avg | Delta |
|-------------------------------------------|----------|-------|
| tRDP=1 alone | 526ns | 0ns |
| tDeltaWR=8 alone | 526ns | 0ns |
| BurstSize ReadQ=2 alone | 532ns | 0ns |
| tRC=21 + tRAS/tRP=0x40 alone | 442ns | -85ns |

The combined gains from tRDP, tDeltaWR, and BurstSize are noise (under 2ns). The tRAS/tRP byte within CMD_DUR provides additional improvement beyond tRC alone, but all gains come from entry 16.

### Hard Limits Found

Tested on CECHA00 with both Elpida EDX5116ACSE-3C-E and Samsung series at 3.7 GHz. Both DRAM vendors hit identical floors on all parameters, despite Samsung generally being considered faster silicon. The limits are likely MIC controller or XIO cell constraints rather than DRAM speed limits.

| Parameter | Floor | Failure Mode |
|-----------|-------|-------------|
| tRCD-R | 7 (stock) | 5 = XDR link fail (0xb0002001) |
| tRCD-W | 3 (stock) | Already at minimum |
| tRC | 21 | 20 = bit flips in random stress test (both vendors) |
| tRAS/tRP | 0x40 | 0x30 = bit flips in stress test (117 errors in minutes) |
| tRDP | 1 | MIC hardware minimum. 75% below C-bin spec. |
| tWRP | 0x83 | 0x82 = lv0 not found, 0x81 = XDR link fail |
| tDeltaRW | 9 (stock) | 8 = lv0 auth fail (data corruption) |
| tDeltaWR | 8 | Rule 2 hard floor: must be > tRCD_R (7) |
| MIC_DF_CTL | Stock only | tQRD+/-1 both crash (XIO-coupled) |

## Complete 46-Entry NVS Map

### Legend

- **TUNED**: Optimized value in active profile
- **OPEN**: Accepts non-stock values, potential tuning target
- **LOCKED**: Only stock value works (XIO-coupled)
- **WALL**: Zeroing crashes, stock-only or needs range probing
- **STOCK OK**: Stock write confirmed, no non-stock values tested

### Entry Table

| # | NVS | Size | ROM Default | Register | Status | Notes |
|---|-----|------|-------------|----------|--------|-------|
| 0 | 0x3140 | 1 | 02 | MIC_DEV_CFG | STOCK OK | Bank count. Do not change. |
| 1 | 0x3141 | 1 | 02 | MIC_DEV_CFG | STOCK OK | Device width. Do not change. |
| 2 | 0x3142 | 1 | 10 | MIC_MEM_CFG | STOCK OK | Memory config byte 0. |
| 3 | 0x3143 | 1 | 20 | MIC_MEM_CFG | WALL | 00=HANG (ERAW enable). lv0 lacks ERAW init. |
| 4 | 0x3144 | 1 | 08 | unknown | OPEN | 00=HANG but 04/07/09/10 all boot. Wide range. |
| 5 | 0x3145 | 1 | 00 | unknown | STOCK OK | Already zero. |
| 6 | 0x3146 | 1 | 00 | unknown | STOCK OK | Already zero. |
| 7 | 0x3147 | 1 | 01 | unknown | OPEN | 00/02/FF all boot. Non-critical. |
| 8 | 0x3148 | 2 | 80 00 | MIC_TRCD_PCHG | STOCK OK | TRCD_PCHG partial. |
| 9 | 0x314A | 2 | FF C0 | MIC_TRCD_PCHG | STOCK OK | TRCD_PCHG partial. |
| 10 | 0x314C | 2 | 32 00 | MIC_TRCD_PCHG | STOCK OK | TRCD_PCHG partial. |
| 11 | 0x314E | 4 | 06 11 01 70 | MIC_REF_SCB | STOCK OK | Refresh counter. |
| 12 | 0x3152 | 2 | 7C FE | MIC_MNT_CFG | STOCK OK | 10us timer + channel pop. |
| 13 | 0x3154 | 4 | 50 20 00 00 | MIC_CTL_CNFG | WALL | SpecRead=CRASH(0xb0002002), HalfOp=HANG, FPath=HANG |
| 14 | 0x3158 | 2 | 00 E0 | spacing | STOCK OK | Unknown purpose. |
| 15 | 0x315A | 6 | 62 84 05 5A D6 B0 | MIC_TRCD_PCHG | OPEN | tRDP=1 stable but zero measured read impact. |
| 16 | 0x3160 | 2 | 5D 70 | MIC_CMD_DUR | **TUNED** | tRC=21, tRAS/tRP=0x40. Only impactful register. |
| 17 | 0x3162 | 6 | 71 80 02 10 00 00 | MIC_CMD_SPC | OPEN | tDeltaWR=8 stable but zero measured read impact. |
| 18 | 0x3168 | 4 | 0A 96 3D 60 | MIC_DF_CTL | **LOCKED** | XIO-coupled. tQRD=12 and 14 both crash. |
| 19 | 0x316C | 2 | E1 C0 | MIC_DF_CONFIG | OPEN | ECC enable (C1 40) boots. Async_Delay=0. |
| 20 | 0x316E | 2 | C8 00 | unknown | STOCK OK | Unknown config. |
| 21 | 0x3170 | 8 | 00 00 00 00 00 00 00 00 | MIC_QUE_BURSTSIZE | OPEN | ReadQ/WriteQ arbitration. Zero measured read impact. |
| 22 | 0x3178 | 8 | ED D6 12 29 59 4B A6 B4 | MIC_TM_THRESHOLD | OPEN | HIG values (3B F0 77 E0...) boot fine. |
| 23 | 0x3180 | 8 | 53 49 AC B6 88 C4 62 20 | MIC_XIO_PTCAL | STOCK OK | Periodic timing cal pattern data. |
| 24 | 0x3188 | 4 | 00 00 00 00 | MIC_REF_SCB | STOCK OK | Refresh/scrub config. |
| 25 | 0x318C | 2 | 00 40 | MIC_REF_SCB | STOCK OK | Refresh/scrub config. |
| 26 | 0x318E | 2 | 00 00 | unknown | -- | Already zero. |
| 27 | 0x3190 | 2 | 08 54 | XIO_CTL_PCAL_0a | STOCK OK | XIO periodic calibration timing. |
| 28 | 0x3192 | 2 | 0C 54 | XIO_CTL_PCAL_0b | STOCK OK | XIO periodic calibration timing. |
| 29 | 0x3194 | 2 | 14 9F | XIO_CTL_PCAL_1a | STOCK OK | XIO periodic calibration timing. |
| 30 | 0x3196 | 2 | 18 9F | XIO_CTL_PCAL_1b | STOCK OK | XIO periodic calibration timing. |
| 31 | 0x3198 | 2 | 00 58 | YREG_INIT_CNTS | STOCK OK | tTKE-W, tD-TKE, tKE-L. |
| 32 | 0x319A | 2 | 00 80 | YREG_INIT_CNTS | STOCK OK | tPMT. |
| 33 | 0x319C | 2 | 00 01 | YREG_INIT_CNTS | OPEN | 00 00 boots. DynClkEna toggleable. |
| 34 | 0x319E | 2 | FC 01 | YREG_INIT_CNTS | STOCK OK | Init constants. |
| 35 | 0x31A0 | 2 | 00 06 | XIO_PLL_0a | OPEN | Full 00-FF range boots. PLL field. |
| 36 | 0x31A2 | 2 | 00 0F | XIO_PLL_0b | OPEN | 00 10 boots. Flexible. |
| 37 | 0x31A4 | 2 | FC 0A | XIO_PLL_1a | partial | Upper byte 0xFC critical (00=link fail). Lower byte flexible. |
| 38 | 0x31A6 | 2 | 00 06 | XIO_PLL_1b | WALL | 00 00 = XMB freeze. Must keep stock. |
| 39 | 0x31A8 | 2 | 00 0F | XIO_PLL_1c | WALL | 00 00 = XMB freeze. Must keep stock. |
| 40 | 0x31AA | 1 | 37 | per-ch (PTC ctrl) | WALL | 00=CRASH. 36/38=CRASH. 27/17=BOOT. Controls PTC enable. |
| 41 | 0x31AB | 1 | 00 | per-ch | -- | Already zero. |
| 42 | 0x31AC | 1 | 00 | per-ch | -- | Already zero. |
| 43 | 0x31AD | 1 | 3F | per-ch | WALL | 00=CRASH+YLOD(0xb000200d). 3E=CRASH. FF=BOOT. |
| 44 | 0x31AE | 1 | 23 | per-ch | OPEN | 00=CRASH(0xb000200f). 22/24/33/13 all boot. Range found. |
| 45 | 0x31AF | 1 | 28 | per-ch | OPEN | 00/14/50/FF all boot. Fully flexible. |

## MIC Register Bitfield Definitions

Source: CBE Public Registers v1.5, Section 7.3 (pages 171-180).

### MIC_TRCD_PCHG (0x0D0/0x190) - Entry 15, NVS 0x315A, 6 bytes

| Bits | Field | Formula | Stock | Tight-3 |
|------|-------|---------|-------|---------|
| [0:3] | tRCD_R | Encoded: 0x2=3, 0x4=5, 0x6=7, 0x8=9 | 0x6 (7) | 0x6 (7) |
| [4:7] | tRCD_W | Encoded: 0x2=3, 0x4=5, 0x6=7, 0x8=9 | 0x2 (3) | 0x2 (3) |
| [8:12] | RPrecharge | max(tRAS-1, tRCD_R+2+tRDP-1). Min=5 | 16 | 13 |
| [13:17] | WPrecharge | max(tRAS-1, tRCD_W+2+tWRP-1). Min=5 | 16 | 12 |
| [18:23] | tPM0 | register = tPM0 - 3. Min=5 (tPM0 min=8, rec=45) | 5 (tPM0=8) | 2 (tPM0=5) |
| [24:28] | tPM_CALZ | register = tPM_CALZ - 3. Min=5 | 11 (14) | 11 (14) |
| [29:33] | tPM_CALC | register = tPM_CALC - 3. Min=5 | 11 (14) | 11 (14) |
| [34:38] | tCALZE | register = tCALZE - 1. Min=7 | 11 (12) | 11 (12) |
| [39:43] | tCALCE | register = tCALCE - 1. Min=7 | 11 (12) | 11 (12) |

### MIC_CMD_DUR (0x0D8/0x198) - Entry 16, NVS 0x3160, 2 bytes

| Bits | Field | Formula |
|------|-------|---------|
| [0:5] | ERCTR | max(tRC-1, tRAS+tRP-1, tRCD_R+2+tRDP+tRP-1). Min=7, Max=60 |
| [6:11] | ERCTW | max(tRC-1, tRAS+tRP-1, tRCD_W+2+tWRP+tRP-1). Min=7 |

tRC encoder: `byte0 = ((tRC - 1) << 2) | (lower_2_bits)`. Stock 0x5D = tRC 24. Tight-3 0x51 = tRC 21.

### MIC_CMD_SPC (0x0E0/0x1A0) - Entry 17, NVS 0x3162, 6 bytes

| Bits | Field | Formula (BL=16) |
|------|-------|-----------------|
| [0:4] | RTW_ComIntvl | tRCD_R + 2 + tDeltaRW - tRCD_W - 2 |
| [5:9] | WTR_ComIntvl_SB | tRCD_W + 2 + tDeltaWR - tRCD_R - 2 (non-ERAW) |
| [10:14] | WTR_ComIntvl_OB | tRCD_W + 2 + tDeltaWR_D - tRCD_R - 2 (ERAW only) |
| [15:21] | WTR_Same_Ctr | (tDeltaWR * 4) - 18 (ERAW only) |
| [22:25] | LCASCR | tRCD_R + 1 |
| [26:29] | LCASCW | tRCD_W + 1 |
| [30:34] | RowPrechDiff | WPrecharge - RPrecharge - 2 (if neg, use 0) |
| [35] | tWR_D_NR | 1 if tDeltaWR_D unrestricted per DRAM datasheet |

### MIC_DF_CTL (0x0E8/0x1A8) - Entry 18, NVS 0x3168, 4 bytes

| Bits | Field | Formula | Stock |
|------|-------|---------|-------|
| [0:4] | LDLValue | tRCD_W + tQTD - 5 | 1 |
| [5:9] | ELDLValue | tRCD_R + (tQRD - tERD) - 5 | 10 |
| [10:14] | RCDLValue | tQRD - 2 | 11 |
| [15:20] | MvWrDelay | [(tRCD_W + tQTD - 3) * 4] - 5 | 7 |
| [21:26] | MvRdDelay | [(tRCD_R + tQRD - tERD - 3) * 4] - 5 | 43 |

Derived parameters: tQTD=3, tQRD=13, tERD=5. These are XIO-coupled and cannot be changed independently. Sony changed only tQRD from HIG's 12 to 13 for 1.6 GHz MiClk.

### MIC_Que_BurstSize (0x0B0/0x1F0) - Entry 21, NVS 0x3170, 8 bytes

| Bits | Field | Recommended |
|------|-------|-------------|
| [0:4] | ReadQ_BurstSize | 4 (stock). Tight-3 uses 2 (read-biased). |
| [5:9] | WriteQ_BurstSize | 12 (stock). Tight-3 uses 16. |

### MIC_DF_Config (0x218) - Entry 19, NVS 0x316C, 2 bytes

| Bits | Field | Stock |
|------|-------|-------|
| [0] | Parity_Disable0 | 1 (parity OFF) |
| [1] | ECC_Disable0 | 1 (ECC OFF) |
| [2] | Report_SingleBit0 | 1 |
| [3:6] | Async_Delay0 | 0 (correct for MiClk/NClk = 0.5) |
| [7] | Parity_Disable1 | 1 (parity OFF) |
| [8] | ECC_Disable1 | 1 (ECC OFF) |
| [9] | Report_SingleBit1 | 1 |
| [10:13] | Async_Delay1 | 0 |

Formula: Async_Delay = 18 - 38 * (MiClk/NClk). At 1:1 ratio, MiClk/NClk = 0.5, result = -1, clamped to 0.

### MIC_CTL_CNFG_n (0x080/0x1C0) - Entry 13, NVS 0x3154, 4 bytes

| Bits | Field | Stock |
|------|-------|-------|
| [0] | Disable_Power_Sav | 0 (power savings ON) |
| [1] | Disable_Spec_Read | 1 (speculative reads OFF) |
| [2] | Enable_Half_Op | 0 (64B ops OFF) |
| [3] | AUX_TRACE_16BUF | 1 |

Speculative reads (bit 1=0) causes crash 0xb0002002 with partial XIO calibration data. Half ops (bit 2=1) hangs after boot loader. Neither is usable via NVS config at OC speed.

## Key Discoveries

### 1. MIC_DF_CTL is XIO-coupled (LOCKED)

Both increasing and decreasing tQRD from stock value 13 crashes with XDR link failure. The XIO cell has its own internal register programmed during init that expects exactly tQRD=13. The MIC DF_CTL fields must match. This register cannot be tuned independently through the NVS config system.

### 2. Entry 40 (0x31AA) controls Periodic Timing Calibration

Changing from stock 0x37 in either direction (+/-1) causes link failure with "PTC is not enabled" in the crash log. The MIC_DUMP shows TM_Threshold replaced with PTC-related values (0x1024E10F instead of stock 0xEDD612...). Values 0x27 and 0x17 boot, suggesting bits [4:3] (the 0x30 mask) control a feature flag and the lower bits are configuration.

### 3. Per-channel bytes (40-45) produce unique error codes

| Entry | Stock | 00 Result | Error Code |
|-------|-------|-----------|------------|
| 40 | 0x37 | CRASH | 0xb0002001 |
| 43 | 0x3F | CRASH + YLOD | 0xb000200d |
| 44 | 0x23 | CRASH | 0xb000200f |
| 45 | 0x28 | BOOTS | n/a |

Different error subcodes suggest per-lane or per-DQ-group configuration. Entry 44 accepts values 0x13-0x33 (wide range). Entry 45 accepts 0x00-0xFF (completely flexible).

### 4. Sony ships with ECC and Parity disabled

MIC_DF_Config has both ECC_Disable and Parity_Disable set. Enabling ECC (`w 316C C1 40`) boots and works. This can be used as a diagnostic tool to detect marginal bit errors in tightened timing profiles. But this shouldn't be enabled if you're chasing performance.

### 5. ERAW is blocked by lv0

Early Read After Write (MIC_Mem_Cfg bit 10) hangs at XDR ASSERT/DEASSERT regardless of supporting CMD_SPC and DF_CTL changes. It appears lv0's XDR init sequence does not support ERAW mode?

### 6. Elpida C-bin tRDP has 75%+ margin

Stock C-bin spec: tRDP=4. A-bin spec: tRDP=3. Confirmed stable at tRDP=1 (MIC hardware minimum). This is the largest margin discovered on any parameter.

### 7. XIO PLL entries are writable but effect is unknown

Entries 35-36 (XIO_PLL_0) accept wide value ranges without crashing. Entry 37 upper byte (0xFC) is critical. Entries 38-39 crash at 0x00 but work at stock. No measurable bandwidth change was observed during testing.

### 8. CMD_DUR is the ONLY register that measurably affects read latency

Isolation testing on Samsung XDR with cellmark latency tests proved that MIC_CMD_DUR (entry 16, NVS 0x3160) accounts for the entire measurable read latency improvement. tRDP (entry 15), tDeltaWR (entry 17), and BurstSize (entry 21) all showed zero standalone impact on read latency despite being valid timing changes that pass stability tests. The min read latency (page-hit) is locked behind MIC_DF_CTL pipeline depth and cannot be improved.

### 9. Samsung and Elpida hit identical timing floors

Both DRAM vendors reach the same hard limits (tRC=21, tRAS/tRP=0x40) at 3.7 GHz despite Samsung being generally faster silicon. This suggests the floor is set by the MIC controller or XIO cell, not the DRAM chips themselves.

## Elpida Speed Bin Comparison

All values in tCYCLE counts at rated 3.2G speed (tCYCLE = 2.500 ns).

| Parameter | Bin A (fast) | Bin B | Bin C (PS3) | Tight-3 | Tight-3 ns |
|-----------|-------------|-------|-------------|---------|------------|
| tRC | 16 | 20 | 24 | 21 | 45.4 |
| tRAS | 10 | 13 | 17 | 14 | 30.3 |
| tRP | 6 | 7 | 7 | 5 | 10.8 |
| tRCD-R | 5 | 7 | 7 | 7 | 15.1 |
| tRCD-W | 1 | 3 | 3 | 3 | 6.5 |
| tRDP | 3 | 4 | 4 | 1 | 2.2 |
| tWRP | 10 | 12 | 12 | 11 | 23.8 |
| tDeltaRW | 8 | 9 | 9 | 9 | 19.5 |
| tDeltaWR | 9 | 10 | 10 | 8 | 17.3 |

## HIG Programming Rules Summary

16 rules from CellBE HIG 90nm v1.5, Table E-2. Critical rules for Tight-3:

| Rule | Constraint | Tight-3 Value | Status |
|------|-----------|---------------|--------|
| 1 | tDeltaRW > tRCD_W | 9 > 3 | PASS |
| 2 | tDeltaWR > tRCD_R | 8 > 7 | PASS (margin=1) |
| 4 | tDeltaWR >= tWRP - tRDP + tPP + tCC - 4 | 8 >= 11-1+4+2-4 = 12 | FAIL (by 4) |
| 8 | tRCD_W + tQTD >= 5 | 3+3 = 6 | PASS |
| 12 | tDeltaRW >= (tCAC-tCWD) + tCC + tRW_BUB_min | 9 >= 4+2+0+3 = 9 | PASS (margin=0) |
| 14 | tWRP >= tCWD + tDP_min | 11 >= 3+9 = 12 | FAIL (by 1) |
| 15 | tDeltaWR >= tCWD + tDR_min | 8 >= 3+7 = 10 | FAIL (by 2) |

Rules 4, 14, 15 are technically violated but use worst-case tDP/tDR values from the C-bin datasheet. Actual silicon has more margin, confirmed by stability testing.

## Open Questions

1. What do XIO PLL entries 35-36 actually control? We need a memory bandwidth benchmark to measure impact.
2. Can entry 40 (PTC control) be tuned to reduce calibration overhead without disabling it?
3. What are entries 4-7? Entry 4 accepts 04-10 but not 00. Could be a counter or threshold.
4. What does entry 44 (per-ch 4, stock 0x23) configure? Accepts wide range. Could be drive strength.
5. Could speculative reads work?
6. Is the MIC_Ctl_Cnfg2 register (0x040, has Fast Path bit) accessible through a different NVS entry?

## Next Steps

1. Probe entry 40 bit-by-bit to map which bits control PTC enable vs PTC configuration.
2. Test on additional PS3 models (slim, super slim) with different Cell process nodes.

# Part 5: Complete NVS-to-MIC Register Map

## How It Works

1. Write timing bytes to entries in NVS 0x3140-0x31AF
2. Any entry left as all-FF uses the ROM default from firmware table at 0x402AC
3. On boot, sub_1E548 builds a 112-byte config block from NVS + ROM defaults
4. Cell requests the block via SERV_SETCFG (SID 18, cmd 0, sub 0x10)
5. lv0 programs the MIC (Memory Interface Controller) registers inside Cell
6. MIC controls all XDR DRAM timing and scheduling

CRITICAL: When overriding an entry, you MUST write ALL bytes for that entry.
If any byte is non-FF, the firmware uses NVS data for the ENTIRE entry.
Partial writes will feed garbage (mixed real data + 0xFF) to the MIC.

---

## Complete Descriptor Table (46 entries)

| #  | Ofs  | Sz | NVS Addr | ROM Default                  | MIC Register              | MMIO HIG | Timing Params                                  | Tier     |
|----|------|----|----------|------------------------------|---------------------------|----------|------------------------------------------------|----------|
| 0  | 0x00 | 1  | 0x3140   | 02                           | MIC_DEV_CFG               | 0x0C0    |                                                | STRUCT   |
| 1  | 0x01 | 1  | 0x3141   | 02                           | MIC_DEV_CFG               | 0x0C0    |                                                | STRUCT   |
| 2  | 0x02 | 1  | 0x3142   | 10                           | MIC_MEM_CFG               | 0x0C8    |                                                | STRUCT   |
| 3  | 0x03 | 1  | 0x3143   | 20                           | MIC_MEM_CFG               | 0x0C8    |                                                | STRUCT   |
| 4  | 0x04 | 1  | 0x3144   | 08                           | (config)                  |          |                                                | STRUCT   |
| 5  | 0x05 | 1  | 0x3145   | 00                           | (config)                  |          |                                                | STRUCT   |
| 6  | 0x06 | 1  | 0x3146   | 00                           | (config)                  |          |                                                | STRUCT   |
| 7  | 0x07 | 1  | 0x3147   | 01                           | (config)                  |          |                                                | STRUCT   |
| 8  | 0x08 | 2  | 0x3148   | 80 00                        | MIC_TRCD_PCHG             | 0x0D0    | tRCD-R, tRCD-W                                 | CAUTION  |
| 9  | 0x0A | 2  | 0x314A   | FF C0                        | MIC_TRCD_PCHG             | 0x0D0    | tPM0, tPM-CALZ, tPM-CALC                       | CAUTION  |
| 10 | 0x0C | 2  | 0x314C   | 32 00                        | MIC_TRCD_PCHG             | 0x0D0    | tCALZE, tCALCE                                 | CAUTION  |
| 11 | 0x0E | 4  | 0x314E   | 06 11 01 70                  | MIC_REF_SCB (partial)     | 0x200    | tREF                                           | CAUTION  |
| 12 | 0x12 | 2  | 0x3152   | 7C FE                        | MIC_MNT_CFG               | 0x210    |                                                | STRUCT   |
| 13 | 0x14 | 4  | 0x3154   | 50 20 00 00                  | MIC_CTL_CNFG_0            | 0x080    |                                                | STRUCT   |
| 14 | 0x18 | 2  | 0x3158   | 00 E0                        | (spacing)                 |          |                                                | STRUCT   |
| 15 | 0x1A | 6  | 0x315A   | 62 84 05 5A D6 B0            | MIC_TRCD_PCHG             | 0x0D0    | tRCD-R/W, tRDP, tWRP, tPM0, tPM-CALZ/C, tCALZE/CE | TUNE  |
| 16 | 0x20 | 2  | 0x3160   | 5D 70                        | MIC_CMD_DUR               | 0x0D8    | tRC, tRAS, tRP                                 | TUNE     |
| 17 | 0x22 | 6  | 0x3162   | 71 80 02 10 00 00            | MIC_CMD_SPC               | 0x0E0    | tDeltaRW, tDeltaWR, tDeltaWR-D, tRFC           | TUNE     |
| 18 | 0x28 | 4  | 0x3168   | 0A 96 3D 60                  | MIC_DF_CTL                | 0x0E8    | tQTD, tQRD, tERD                               | MODERATE |
| 19 | 0x2C | 2  | 0x316C   | E1 C0                        | MIC_DF_CONFIG (partial)   | 0x218    |                                                | STRUCT   |
| 20 | 0x2E | 2  | 0x316E   | C8 00                        | (config)                  |          |                                                | STRUCT   |
| 21 | 0x30 | 8  | 0x3170   | 00 00 00 00 00 00 00 00      | MIC_QUE_BURSTSIZE         | 0x0B0    |                                                | MODERATE |
| 22 | 0x38 | 8  | 0x3178   | ED D6 12 29 59 4B A6 B4      | MIC_TM_THRESHOLD          | 0x0A8    |                                                | MODERATE |
| 23 | 0x40 | 8  | 0x3180   | 53 49 AC B6 88 C4 62 20      | MIC_XIO_PTCAL_DATA        | 0x0F0    |                                                | CAL_DATA |
| 24 | 0x48 | 4  | 0x3188   | 00 00 00 00                  | MIC_REF_SCB               | 0x200    | tREF interval                                  | CAUTION  |
| 25 | 0x4C | 2  | 0x318C   | 00 40                        | MIC_REF_SCB (cfg)         | 0x200    |                                                | CAUTION  |
| 26 | 0x4E | 2  | 0x318E   | 00 00                        | (config)                  |          |                                                | STRUCT   |
| 27 | 0x50 | 2  | 0x3190   | 08 54                        | XIO_CTL_PCAL_0            |          | tPCAL                                          | DANGER   |
| 28 | 0x52 | 2  | 0x3192   | 0C 54                        | XIO_CTL_PCAL_0            |          |                                                | DANGER   |
| 29 | 0x54 | 2  | 0x3194   | 14 9F                        | XIO_CTL_PCAL_1            |          |                                                | DANGER   |
| 30 | 0x56 | 2  | 0x3196   | 18 9F                        | XIO_CTL_PCAL_1            |          |                                                | DANGER   |
| 31 | 0x58 | 2  | 0x3198   | 00 58                        | YREG_INIT_CNTS            | 0x120    | tTKE-W, tD-TKE, tKE-L                          | DANGER   |
| 32 | 0x5A | 2  | 0x319A   | 00 80                        | YREG_INIT_CNTS            | 0x120    | tPMT                                           | DANGER   |
| 33 | 0x5C | 2  | 0x319C   | 00 01                        | YREG_INIT_CNTS            | 0x120    |                                                | DANGER   |
| 34 | 0x5E | 2  | 0x319E   | FC 01                        | YREG_INIT_CNTS            | 0x120    |                                                | DANGER   |
| 35 | 0x60 | 2  | 0x31A0   | 00 06                        | XIO_PLL_0                 |          |                                                | DANGER   |
| 36 | 0x62 | 2  | 0x31A2   | 00 0F                        | XIO_PLL_0                 |          |                                                | DANGER   |
| 37 | 0x64 | 2  | 0x31A4   | FC 0A                        | XIO_PLL_1                 |          |                                                | DANGER   |
| 38 | 0x66 | 2  | 0x31A6   | 00 06                        | XIO_PLL_1                 |          |                                                | DANGER   |
| 39 | 0x68 | 2  | 0x31A8   | 00 0F                        | XIO_PLL_1                 |          |                                                | DANGER   |
| 40 | 0x6A | 1  | 0x31AA   | 37                           | (per-ch)                  |          |                                                | STRUCT   |
| 41 | 0x6B | 1  | 0x31AB   | 00                           | (per-ch)                  |          |                                                | STRUCT   |
| 42 | 0x6C | 1  | 0x31AC   | 00                           | (per-ch)                  |          |                                                | STRUCT   |
| 43 | 0x6D | 1  | 0x31AD   | 3F                           | (per-ch)                  |          |                                                | STRUCT   |
| 44 | 0x6E | 1  | 0x31AE   | 23                           | (per-ch)                  |          |                                                | STRUCT   |
| 45 | 0x6F | 1  | 0x31AF   | 28                           | (per-ch)                  |          |                                                | STRUCT   |

---

## Should I poke this value?

| Tier | Meaning | Entries |
|------|---------|---------|
| TUNE | Safe timing knobs -- DDR-equivalent tuning. Start here. | 15, 16, 17 |
| MODERATE | Dataflow/arbitration. Tunable with understanding. | 18, 21, 22 |
| CAUTION | Calibration periods, refresh. Risky to change. | 8-10, 11, 24-25 |
| DANGER | XIO PHY, PLL, init counters. Can brick link training. | 27-39 |
| STRUCT | Device/memory geometry, bus config. Do NOT change. | 0-7, 12-14, 19-20, 26, 40-45 |
| CAL_DATA | Calibration pattern data. Do NOT change. | 23 |

---

## Key Timing Entries (Tier TUNE) - Detailed

### Entry 15: MIC_TRCD_PCHG (NVS 0x315A, 6 bytes)

Stock: `62 84 05 5A D6 B0`
MMIO: 0x0D0 (MIC_TRCD_PCHG_0), 0x190 (MIC_TRCD_PCHG_1)

Contains the most latency-sensitive timing parameters:

| Bit Position (IBM) | Field | Stock Value | Range | Notes |
|-------------------|-------|-------------|-------|-------|
| [2:3] | tRCD-R | 2 (=7 tCYCLEs) | 0=3, 1=5, 2=7, 3=9 | Row-to-column delay for reads. XDR equivalent of CAS latency. TESTED: 5 CRASHES on CECHA00 |
| [4:5] | tRCD-W | 0 (=3 tCYCLEs) | 0=3, 1=5, 2=7, 3=9 | Row-to-column delay for writes. Already at minimum! |
| [6:17] | tRDP, tWRP | - | tRDP: 1-8, tWRP: 6-16 | Read-to-precharge, write recovery |
| [18:23] | tPM0 | 5 | 8-64 (rec: 45) | Periodic calibration master timeout. HIG recommends 45. |
| [24:47] | tPM-CALZ/C, tCALZE/CE | - | 8-32 each | Calibration sub-periods |

```
Stock:       w 315A 62 84 05 5A D6 B0
tRCD-R=5:    w 315A 52 84 05 5A D6 B0   <- CRASHED on test unit
Revert:      w 315A FF FF FF FF FF FF
```

### Entry 16: MIC_CMD_DUR (NVS 0x3160, 2 bytes)

Stock: `5D 70`
MMIO: 0x0D8 (MIC_CMD_DUR_0), 0x198 (MIC_CMD_DUR_1)

Controls row cycle time and derived command durations:

| Bit Position (IBM) | Field | Stock Value | Range | Notes |
|-------------------|-------|-------------|-------|-------|
| [0:5] | tRC - 1 | 23 (=tRC 24) | 7-39 (tRC 8-40) | Row cycle time. TESTED: tRC=22 (0x55) WORKS! |
| [6:10] | (derived) | 11 | - | Packed read command duration |
| [11:15] | (derived) | 16 | - | Packed write command duration |

HIG: 'effective tRC = command duration register value + 1'
HIG: tRC range 8-40. Stock 24, designed for 3.0 GHz. PS3 runs 3.2 GHz.

```
Stock:    w 3160 5D 70    (tRC=24)
tRC=23:   w 3160 59 70    (byte0: 010110|01)
tRC=22:   w 3160 55 70    (byte0: 010101|01) <- STABLE
tRC=21:   w 3160 51 70    (byte0: 010100|01) <- STABLE (best confirmed)
tRC=20:   w 3160 4D 70    (byte0: 010011|01) <- UNSTABLE (intermittent corruption)
tRC=18:   w 3160 45 70    (byte0: 010001|01) <- CRASHED
Revert:   w 3160 FF FF
```

### Entry 17: MIC_CMD_SPC (NVS 0x3162, 6 bytes)

Stock: `71 80 02 10 00 00`
MMIO: 0x0E0 (MIC_CMD_SPC_0), 0x1A0 (MIC_CMD_SPC_1)

Controls read/write turnaround spacing. This is where bandwidth gains hide.
Every clock cycle of turnaround penalty is dead bus time.

| Bit Position | Field | Stock | Tuned | Range | Notes |
|-------------|-------|-------|-------|-------|-------|
| Byte 0 [7:4] | tDeltaRW - 2 | 0x7 (=9) | 0x7 (=9) | 4-16 | Read-to-write spacing. Floor is 9 on tested unit. |
| Byte 1 [7:4] | tDeltaWR - 2 | 0x8 (=10) | 0x6 (=8) | 6-16 | Write-to-read spacing. Tuned to minimum per Rule 2. |
| Byte 2 | tDeltaWR-D / tRFC | 0x02 | 0x02 | 2-6 | Already at minimum. |
| Byte 3 | (flags) | 0x10 | 0x10 | - | Unknown, preserve. |

HIG programming rules (MUST follow):
  Rule 1: tDeltaRW > tRCD-W  (9 > 3, satisfied)
  Rule 2: tDeltaWR > tRCD-R  (8 > 7, minimum possible)
  Rule 3: tDeltaRW >= (tRDP + 3) - tWRP + tPP + tCC - 4
  Rule 4: tDeltaWR >= tWRP - tRDP + tPP + tCC - 4

```
Stock:         w 3162 71 80 02 10 00 00   (tDeltaRW=9, tDeltaWR=10)
tDeltaWR=9:    w 3162 71 70 02 10 00 00   <- STABLE
tDeltaWR=8:    w 3162 71 60 02 10 00 00   <- STABLE (minimum per Rule 2)
tDeltaRW=8:    w 3162 61 70 02 10 00 00   <- CRASHED
Revert:        w 3162 FF FF FF FF FF FF
```

### Entry 18: MIC_DF_CTL (NVS 0x3168, 4 bytes)

Stock: `0A 96 3D 60` (Sony tuned - differs from HIG sample 0A 54 3C E0)
MMIO: 0x0E8 (MIC_DF_CTL_0), 0x1A8 (MIC_DF_CTL_1)

Dataflow control - Sony already customized this for PS3 XDR chips.
Contains tQTD (2-11), tQRD (6-20), tERD (3-5).
Change with caution - Sony's values may already be optimal for their chip binning.

```
Stock:    w 3168 0A 96 3D 60
HIG demo: w 3168 0A 54 3C E0   <- NOT recommended, generic config
Revert:   w 3168 FF FF FF FF
```

---

## Testing Results Log

| Entry | Change | NVS Write | Result |
|------|-------|--------|-----------|
| 15 | Stock MIC_Trcd_Pchg baseline | `w 315A 62 84 05 5A D6 B0` | BOOT OK |
| 15 | tRCD-R: 7->5 | `w 315A 52 84 05 5A D6 B0` | CRASH (0xb0002001 XDR link fail) |
| 16 | tRC: 24->22 | `w 3160 55 70` | STABLE |
| 16 | tRC: 24->21 | `w 3160 51 70` | STABLE |
| 16 | tRC: 24->20 | `w 3160 4D 70` | UNSTABLE (intermittent lv0/lv1 auth fail) |
| 17 | tDeltaWR: 10->9 | `w 3162 71 70 02 10 00 00` | STABLE |
| 17 | tDeltaRW: 9->8 | `w 3162 61 70 02 10 00 00` | CRASH (lv0 auth fail) |
| 17 | tDeltaWR: 10->8 | `w 3162 71 60 02 10 00 00` | STABLE |
| ALL | Combined: tRC=21 + tDeltaWR=8 | see below | STABLE, game tested |
| 16 | tRAS/tRP: 0x70->0x50 | `w 3160 51 50` | STABLE |
| 15 | tRDP: 0x05->0x04 | `w 315A 62 84 04 5A D6 B0` | STABLE |
| 15 | tRDP: 0x05->0x03 | `w 315A 62 84 03 5A D6 B0` | STABLE |
| 18 | MIC_DF_CTL byte1: 0x96->0x86 | `w 3168 0A 86 3D 60` | CRASH (DMA/IO fail, not XDR link) |
| 16 | tRAS/tRP: 0x70->0x40 | `w 3160 51 40` | STABLE |
| 16 | tRAS/tRP: 0x70->0x30 | `w 3160 51 30` | UNSTABLE (intermittent lv0 auth fail, hangs) |
| 15 | tRDP: 0x05->0x02 | `w 315A 62 84 02 5A D6 B0` | STABLE |
| 15 | tWRP: byte1 0x84->0x80 | `w 315A 62 80 03 5A D6 B0` | CRASH (hangs at XDR ASSERT) |
| 15 | tWRP: byte1 0x84->0x81 | `w 315A 62 81 03 5A D6 B0` | CRASH (0xb0002001 XDR link fail) |
| 15 | tWRP: byte1 0x84->0x82 | `w 315A 62 82 03 5A D6 B0` | CRASH (lv0 not found, data corruption) |
| 15 | tWRP: byte1 0x84->0x83 | `w 315A 62 83 03 5A D6 B0` | STABLE |
| 17 | CMD_SPC byte3: 0x10->0x09 | `w 3162 71 60 02 09 00 00` | CRASH (hangs after SB UART connect) |
| 17 | CMD_SPC byte3: 0x10->0x08 | `w 3162 71 60 02 08 00 00` | CRASH (hangs at XDR DEASSERT) |
| 17 | CMD_SPC byte0 lower: 0x71->0x70 | `w 3162 70 60 02 10 00 00` | CRASH (lv0 auth fail) |
| ALL | **Tight-2 final** | see profile | **STABLE, 30min Just Cause 2** |

---

## Confirmed Stable Profile (detailed)

Confirmed working XDR timing modification. Tested on COK-001 CECHA00/B00
with both Elpida and Samsung XDR DRAM, Evilnat 4.92 CFW.

```
# Optimal profile (single register):
w 3160 51 40
```

| Parameter | Stock | Tuned | Change | Register | Entry |
|-----------|-------|-------|--------|----------|-------|
| tRC | 24 (0x5D) | 21 (0x51) | -12.5% | MIC_Cmd_Dur byte 0 (0x3160) | 16 |
| tRAS/tRP | 0x70 | 0x40 | -3 steps | MIC_Cmd_Dur byte 1 (0x3161) | 16 |

**2 timing parameters tightened in a single MIC register.**

Isolation testing proved that entries 15 (tRDP, tWRP), 17 (tDeltaWR), and 21 (BurstSize) produce zero measurable standalone read latency improvement despite passing all stability tests. The entire 16% read latency reduction comes from MIC_CMD_DUR (entry 16).

**Impact:** Row cycle time reduced by 3 cycles (faster bank recycling), row active and precharge times tightened (faster row operations). These affect page-miss latency, which dominates average memory access time in real workloads.

**Hard limits found on this unit (CECHA00/BOO with Elpida and Samsung XDR):**
- tRC: 21 is floor (20 causes bit flips in stress test on both vendors)
- tRAS/tRP: 0x40 is floor (0x30 causes bit flips in stress test)
- MIC_DF_CTL: DO NOT CHANGE (tQRD+/-1 both crash, XIO-coupled)
- MIC_CMD_SPC byte0 lower nibble: DO NOT CHANGE (0x1->0x0 crashes)
- MIC_CMD_SPC byte3: DO NOT CHANGE (0x10->0x09 or 0x08 hangs during boot)

---

## Quick Reference: Shell Commands

```
# Apply optimal profile
w 3160 51 40

# Revert MIC_Cmd_Dur to ROM default
w 3160 FF FF
```

# Part 6: NVS-to-Rambus XDR Timing Cross-Reference

## Entry 15: MIC_TRCD_PCHG (NVS 0x315A, 6 bytes, MMIO 0x0D0/0x190)

Stock: `62 84 05 5A D6 B0`
Tight-3: `62 83 01 5A D6 B0`

### Byte 0 (0x315A) = 0x62

```
Bits [7:6] = 01  ->  tRCD-R encoding
Bits [5:4] = 10  ->  tRCD-W encoding
Bits [3:0] = 0010 -> upper bits of RPrecharge/WPrecharge
```

**tRCD-R** (bits [7:6] of byte 0, mapped to MIC_TRCD_PCHG[0:3])
- NVS encoding: 0x6 in register field = 7 tCYCLEs
- Encoding map: 0x2=3, 0x4=5, 0x6=7, 0x8=9 (only odd values)
- Rambus Table 13: tRCD-R architectural range 3-9 tCYCLE, recommended initial 5 or 7
- Elpida Table 17: Bin A=5, Bin B=7, Bin C=7
- Samsung Table 18: Bin A=5, Bin B=7, Bin C=7 (identical)
- Stock value: 7 (C-bin). Tested 5 -> XDR link fail (0xb0002001).
- Rambus spec note: tRCD-R only takes odd values {3,5,7,9} per architecture.
- Why 5 crashes: The Elpida C-bin DRAM core physically cannot complete row sensing in 5 tCYCLEs at 466.7 MHz. At stock 400 MHz, 5 tCYCLEs = 12.5 ns. At 466.7 MHz, 5 tCYCLEs = 10.7 ns. The A-bin minimum at 3.2G is 5*2.5 = 12.5 ns. The actual minimum would be 12.5/2.143 = 5.83, rounded up to 7 (next odd).

**tRCD-W** (bits [5:4] of byte 0, mapped to MIC_TRCD_PCHG[4:7])
- NVS encoding: 0x2 in register field = 3 tCYCLEs
- Elpida Table 17: Bin A=1, Bin B=3, Bin C=3
- Already at C-bin minimum. A-bin supports tRCD-W=1 but this is only usable with non-ERAW mode (which PS3 uses), and reducing it would violate Rule R1 parity (tCWD - tDeltaWR-D must be odd).
- Rambus spec note (Table 13 footnote c): "Support for tRCD-W=1 is provided to help in reducing tRC for writes. When using Early Read, Rambus recommends tRCD-W match tRCD-R."

### Byte 1 (0x315B) = 0x84 (stock), 0x83 (Tight-3)

This byte packs RPrecharge (lower bits) and WPrecharge (upper bits), which are derived values containing tRDP and tWRP implicitly.

**RPrecharge** (MIC_TRCD_PCHG[8:12])
- Formula: max(tRAS-1, tRCD-R + 2 + tRDP - 1), minimum 5
- Stock: 16 = max(17-1, 7+2+4-1) = max(16, 12) = 16. Checks out with tRAS=17, tRDP=4.
- Tight-3: tRDP=1 -> max(14-1, 7+2+1-1) = max(13, 9) = 13. This is the RPrecharge implicit in Tight-3.

**WPrecharge** (MIC_TRCD_PCHG[13:17])
- Formula: max(tRAS-1, tRCD-W + 2 + tWRP - 1), minimum 5
- Stock: 16 = max(16, 3+2+12-1) = max(16, 16) = 16. Checks out.
- Tight-3 (0x83): WPrecharge went from 16 to ~15. tWRP reduced by ~1 step.

**tWRP** (derived from WPrecharge field)
- Rambus Table 13: tWRP architectural range 6-16 tCYCLE
- Elpida Table 17: Bin A=10, Bin B=12, Bin C=12
- Stock: 12. Tight-3: ~11.
- Rambus Rule R3a: tWRP > tCWD + tDP_MIN
  - tCWD=3, tDP_MIN=9 (C-bin) -> tWRP > 12. Stock barely passes.
  - At Tight-3 tWRP=11: 11 > 12 is FALSE. Rule violated.
  - But stable because tDP_MIN=9 is a worst-case spec value. Actual silicon likely has tDP=7 or less.
- Test results: 0x83 stable, 0x82 = data corruption, 0x81 = link fail, 0x80 = hang.
  This gradient (progressively worse failures) suggests approaching true silicon limit.

### Byte 2 (0x315C) = 0x05 (stock), 0x01 (Tight-3)

**tRDP** (MIC_TRCD_PCHG field, exact bit position TBD but byte 2 controls it)
- Rambus Table 13: tRDP architectural range 1-8 tCYCLE
  - For tCC=2 devices: minimum 1-8 (full range)
  - For tCC=4 devices: minimum 3-8
  - PS3 uses tCC=2 per both Elpida and Samsung datasheets.
- Elpida Table 17: Bin A=3, Bin B=4, Bin C=4
- Samsung Table 18: Bin A=3, Bin B=4, Bin C=4 (identical)
- Stock NVS: 0x05 -> maps to tRDP=4 (matches C-bin)
- Tight-3 NVS: 0x01 -> maps to tRDP=1
- Absolute ns: tRDP=1 at 2.143 ns = 2.14 ns. A-bin spec = 3*2.5 = 7.5 ns.
- This is the largest margin found on any parameter (75%+ below spec).
- Rambus architecture explicitly allows tRDP=1 for tCC=2 devices.
  The DRAM core only needs enough time to latch the column I/O amps before
  precharge begins. The Elpida core completes this much faster than spec.

### Bytes 3-5 (0x315D-0x315F) = 0x5A D6 B0

**tPM0** (MIC_TRCD_PCHG[18:23])
- Formula: register = tPM0 - 3
- Stock register value = 5, so tPM0 = 8 tCYCLE
- Rambus Table 15: tPM0 range 8-8 tCYCLE (fixed at 8 for all bins)
- HIG recommends tPM0 = 45 for some reason. Sony uses 8 (Rambus minimum).
- This controls the Periodic Calibration master timeout pattern duration.

**tPM-CALZ, tPM-CALC** (MIC_TRCD_PCHG[24:33])
- Rambus Table 15: tPM-CALC/CALZ recommended 14 tCYCLE for initial range
- Stock register = 11 each -> tPM-CALZ = tPM-CALC = 14. Matches Rambus recommendation.

**tCALZE, tCALCE** (MIC_TRCD_PCHG[34:43])
- Rambus Table 15: tCALZE/tCALCE recommended 30 tCYCLE
- Stock register = 11 each -> tCALZE = tCALCE = 12. Sony uses 12, not Rambus's 30.
- This is one of Sony's custom optimizations - shorter calibration windows.


## Entry 16: MIC_CMD_DUR (NVS 0x3160, 2 bytes, MMIO 0x0D8/0x198)

Stock: `5D 70`
Tight-3: `51 40`

### Byte 0 (0x3160) = 0x5D (stock), 0x51 (Tight-3)

**tRC** (MIC_CMD_DUR[0:5], stored as tRC-1)
- Stock: 0x5D -> bits [7:2] = 010111 = 23 -> tRC = 24
- Tight-3: 0x51 -> bits [7:2] = 010100 = 20 -> tRC = 21
- Rambus Table 13: tRC architectural range 8-40 tCYCLE, recommended initial 16-24 (even)
- Elpida Table 17: Bin A=16, Bin B=20, Bin C=24
- Samsung Table 18: identical values
- Absolute ns at 466.7 MHz: tRC=21 * 2.143 = 45.0 ns
  - A-bin at 3.2G: 16 * 2.5 = 40.0 ns
  - Your value sits between A-bin and B-bin absolute times.
- Rambus spec says "even" values recommended but odd values are valid.
- Elpida defines tRC as a composite: tRC >= tRAS + tRP, and also
  tRC-R,2tCC >= tRCD-R + tCC + tRDP + tRP (for 2-column-per-row-access pattern).
  At Tight-3: 7 + 2 + 1 + 5 = 15. tRC=21 >> 15. Not the binding constraint.
  Binding constraint is tRC >= tRAS + tRP. At Tight-3 tRAS~14, tRP~5: 14+5=19. tRC=21 passes.
- Test results: tRC=21 stable, tRC=20 intermittent corruption. The DRAM core needs ~43 ns
  minimum for a full row cycle on this silicon.

**Byte 0 bits [1:0]** = 01 (both stock and Tight-3)
- Part of the packed read/write command duration fields.
- These lower bits encode tRAS/tRP derived values in combination with byte 1.

### Byte 1 (0x3161) = 0x70 (stock), 0x40 (Tight-3)

**tRAS and tRP** (packed into MIC_CMD_DUR[6:15])
- These are packed as command durations, not direct tCYCLE counts.
- Stock 0x70: upper nibble 0x7 -> tRAS region, lower nibble 0x0 -> tRP region
- Tight-3 0x40: upper nibble 0x4 -> reduced tRAS, lower nibble 0x0 -> tRP unchanged

**tRAS** (Row-asserted time)
- Rambus Table 13: architectural range 6-32 tCYCLE, max 64 us
- Elpida Table 17: Bin A=10, Bin B=13, Bin C=17
- Stock: ~17 (matches C-bin). Tight-3: ~14 (between B and A bin).
- Absolute ns: 14 * 2.143 = 30.0 ns. A-bin at 3.2G: 10 * 2.5 = 25.0 ns.
- Test: 0x40 stable, 0x30 intermittent. Silicon limit around tRAS~12 at this frequency.

**tRP** (Row-precharge time)
- Rambus Table 13: architectural range 2-10 tCYCLE
- Elpida Table 17: Bin A=6, Bin B=7, Bin C=7
- Derived from tRC - tRAS. At Tight-3: tRC=21, tRAS~14 -> tRP~7 (stock C-bin value).
  But the tRAS reduction to ~14 with byte1=0x40 suggests tRP may have reduced too.
  At 0x40: if tRAS~11, tRP = 21-11 = 10. More likely tRAS~14, tRP~5.
  Testing 0x30 crashed, suggesting tRP went below ~4 which violates Rambus min of 2
  only if combined with tRAS going too low simultaneously.


## Entry 17: MIC_CMD_SPC (NVS 0x3162, 6 bytes, MMIO 0x0E0/0x1A0)

Stock: `71 80 02 10 00 00`
Tight-3: `71 60 02 10 00 00`

### Byte 0 (0x3162) = 0x71

**Upper nibble 0x7: tDeltaRW** (Read-to-Write turnaround)
- Encoding: upper nibble = tDeltaRW - 2. So 0x7 = 9 tCYCLE.
- Rambus Table 13: t_deltaRW architectural range 4-16 tCYCLE
- Elpida Table 17: Bin A=8, Bin B=9, Bin C=9
- Samsung Table 18: identical
- Stock: 9 (matches C-bin).
- Tested 8 (0x61) -> CRASH (lv0 auth fail from corrupted reads).

**Rambus Rule R2a** (the binding constraint):
  t_deltaRW > (tCAC - tCWD) + tCC + (tPD_CYC - tPD_CYC_MIN) + tRW_BUB_XDRDRAM_MIN
  = (7 - 3) + 2 + (tPD_CYC - 1) + 3
  = 8 + tPD_CYC
  If tPD_CYC = 1 (minimum, short traces on COK-001): minimum t_deltaRW = 9.
  This *exactly* explains why 9 is the floor and 8 crashes.
  The round-trip flight time of the DQ links sets a hard physics constraint.

**HIG Rule 12** (equivalent formulation):
  t_deltaRW >= (tCAC - tCWD) + tCC + tRW_BUB_min + tPD_CYC
  = 4 + 2 + 3 + tPD_CYC = 9 + tPD_CYC (if tPD_CYC >= 1, min = 9)
  Both formulations give the same answer. tDeltaRW = 9 is at zero margin.

**Rambus Guideline G3a**: t_deltaRW > tRCD-W = 3. 9 > 3, trivially satisfied.

**Rambus Guideline G4a**: t_deltaRW > (tRDP - tWRP) + tPP + (tCC - 4)= (1 - 11) + 4 + (2-4) = -10 + 4 - 2 = -8. Always satisfied (negative).

**Lower nibble 0x1: tCC/tPP-D related**
- This nibble is critical. 0x0 crashes, 0x1 works, nothing else tested.
- Rambus Table 13: tCC = 2 or 4 tCYCLE. PS3 uses tCC=2.
- Rambus Table 13: tPP-D = 1 tCYCLE minimum (precharge to different bank sets).
- The 0x1 likely encodes tPP-D = 1. Setting to 0x0 would make tPP-D = 0,
  violating the minimum spacing between precharges to different banks.

### Byte 1 (0x3163) = 0x80 (stock), 0x60 (Tight-3)

**Upper nibble: tDeltaWR** (Write-to-Read turnaround, same bank set)
- Encoding: upper nibble = tDeltaWR - 2. Stock 0x8 = 10, Tight-3 0x6 = 8.
- Rambus Table 13: t_deltaWR architectural range 6-16 tCYCLE
- Elpida Table 17: Bin A=9, Bin B=10, Bin C=10
- Samsung Table 18: identical

**Rambus Rule R3b** (write data must reach core before read):
  t_deltaWR > tCWD + tDR_MIN
  tDR_MIN: Elpida Table 17 Bin C = 7. So t_deltaWR > 3 + 7 = 10.
  Stock 10 barely passes. Tight-3 value 8 violates this by 2.
  But stable because actual tDR on this silicon is faster than 7.

**Rambus Guideline G5** (TDATA conflict avoidance):
  t_deltaWR > tQTD - tQRD + tERD + tCC
  From MIC_DF_CTL: tQTD=3, tQRD=13, tERD=5.
  = 3 - 13 + 5 + 2 = -3. Always satisfied.

**Rambus Rule R2b** (W-R bubble minimum for Early Read):
  t_deltaWR-D > (tCWD - tCAC) + tCC + tWR_BUB_XDRDRAM_MIN
  = (3 - 7) + 2 + 3 = -2 + 2 + 3 = 2 (matches Elpida spec t_deltaWR-D min = 2)

**Rambus Rule R1** (parity constraint for Early Read staggering):
  tCWD - t_deltaWR-D = odd. PS3 doesn't use ERAW so this rule doesn't apply.
  But the MIC still enforces: t_deltaWR > tRCD-R (HIG Rule 2).
  8 > 7 = TRUE. This is the actual binding constraint with margin of 1.

**Lower nibble 0x0** (byte 1): likely tDeltaWR-D encoding for non-ERAW mode.
  Stock 0x0 -> t_deltaWR-D = 2 (Elpida minimum). Matches spec.

### Byte 2 (0x3164) = 0x02

Likely encodes tRR-D (row-to-row time for different bank sets).
- Rambus Table 13: tRR = 4 tCYCLE (fixed), tRR-D = 2-4 tCYCLE
- Elpida Table 17: tRR = 4, tRR-D = 4 (all bins)
- The 0x02 could be tRR-D = 2 (Rambus architectural minimum).
  Sony may have reduced this below Elpida's spec of 4 for better bank interleaving.
  This is safe because tRR-D controls how fast you can activate *different* bank sets,
  and shorter values just allow denser command scheduling.

### Byte 3 (0x3165) = 0x10

This is the critical byte that crashes at 0x09 and 0x08.
- Most likely encodes tRC for refresh operations or tRFC (refresh cycle time).
- Rambus Table 17 (Elpida): tLRRn-LRRn = tREFx-LRRn = tLRRn-REFx
  - Bin A=16, Bin B=20, Bin C=24
- 0x10 = 16 decimal, matching A-bin refresh interval.
  Sony uses A-bin refresh timing even with C-bin DRAM - aggressive but valid because
  refresh timing is often more conservative than core timing in DRAM specs.
- Reducing below 16 violates the architectural minimum for refresh-to-refresh spacing
  and causes the DRAM to lose data during refresh cycles -> hangs during boot.

### Bytes 4-5 (0x3166-0x3167) = 0x00 0x00

Reserved or additional CMD_SPC fields. Stock is zero, left at zero.


## Entry 18: MIC_DF_CTL (NVS 0x3168, 4 bytes, MMIO 0x0E8/0x1A8)

Stock: `0A 96 3D 60`
Status: LOCKED (any change crashes)

This register controls the XIO dataflow pipeline timing. The fields are computed
from XIO interface parameters defined in Rambus Table 15.

### Decoded fields (from CBE Public Registers Section 7.3):

| Field | Formula | Decoded Value | Rambus Param |
|-------|---------|---------------|--------------|
| LDLValue [0:4] | tRCD-W + tQTD - 5 | 1 (= 3+3-5) | tQTD=3 |
| ELDLValue [5:9] | tRCD-R + (tQRD - tERD) - 5 | 10 (= 7+13-5-5) | tQRD=13, tERD=5 |
| RCDLValue [10:14] | tQRD - 2 | 11 (= 13-2) | tQRD=13 |
| MvWrDelay [15:20] | [(tRCD-W + tQTD - 3) * 4] - 5 | 7 (= [3+3-3]*4-5) | - |
| MvRdDelay [21:26] | [(tRCD-R + tQRD - tERD - 3) * 4] - 5 | 43 (= [7+13-5-3]*4-5) | - |

### Derived XIO parameters (verified by back-solving):

| Rambus Parameter | Value | Rambus Table 15 Range | Notes |
|-----------------|-------|----------------------|-------|
| tQTD | 3 | 2-5 (recommended initial) | QDATA-to-TDATA offset. Controls write data pipeline depth. |
| tQRD | 13 | 11-14 (recommended initial) | QDATA-to-RDATA offset. Controls read data return timing. Sony uses 13 vs HIG's 12. |
| tERD | 5 | 3-5 (recommended initial) | Expected Data to RDATA offset. At maximum of recommended range. |

### Why this register is LOCKED:

The XIO cell (Rambus hard macro inside Cell) has its own internal registers programmed
during initialization that expect exactly these tQTD/tQRD/tERD values. The MIC_DF_CTL
register must match the XIO's expectations. Changing MIC_DF_CTL without also changing
the XIO's internal state (which requires XIO register writes during init, controlled by
lv0 firmware) creates a pipeline mismatch. Data arrives at the wrong time relative to
what the XIO expects, causing DMA/IO failures.

From the Rambus Co-Design paper: "Both TX (write) and RX (read) mixers for precision phase
control were implemented in XIO." The XIO has per-bit phase control that is calibrated
during init against these exact delay values. Changing them post-calibration is invalid.

### Relationship to turnaround bubbles (Rambus Table 9):

The Rambus spec defines turnaround bubbles at both the DRAM and XIO:
- tRW_BUB_XIO = tRW_BUB_XDRDRAM - tPD_DQi (per-link, fractional)
- tRW_BUB_XIO_MIN = 2 tCYCLE (first gen XIO)
- tWR_BUB_XIO = tWR_BUB_XDRDRAM + tPD_DQi
- tWR_BUB_XIO_MIN = 3 tCYCLE (first gen XIO, for Early Read W-R)

The XIO bubble minimums are fixed by the XIO design and cannot be reduced
through register changes. This is why tDeltaRW has a hard floor even when
the DRAM itself could theoretically support faster turnarounds.


## Entry 8-10: MIC_TRCD_PCHG partial (NVS 0x3148-0x314D)

These are the OTHER parts of the same MIC_TRCD_PCHG register (0x0D0).
Entry 15 overwrites the primary timing fields, but entries 8-10 contain
additional calibration sub-fields that are loaded separately.

Entry 8 (0x3148, 2 bytes): `80 00` - contains the upper portion of the
TRCD_PCHG register. Likely tRCD-R/tRCD-W for the *second* XDR channel
(MIC_TRCD_PCHG_1 at MMIO 0x190).

Entry 9 (0x314A, 2 bytes): `FF C0` - calibration timeout fields.
Entry 10 (0x314C, 2 bytes): `32 00` - calibration period fields.


## Entry 21: MIC_Que_BurstSize (NVS 0x3170, 8 bytes, MMIO 0x0B0/0x1F0)

Stock: `00 00 00 00 00 00 00 00`
Tight-3: `14 00 00 00 00 00 00 00`

From the IBM Cell MIC presentation: "32 read and 32 write queues for each channel.
Command selection will tend to group commands into burst of 8 or 16 in a row
before switching the bus."

| Field | Bits | Stock | Tight-3 | Effect |
|-------|------|-------|---------|--------|
| ReadQ_BurstSize | [0:4] | 0 (=4 default) | 2 | Smaller read bursts = more frequent bus switches, lower read latency |
| WriteQ_BurstSize | [5:9] | 0 (=12 default) | 16 | Larger write bursts = more efficient write draining |

Rambus spec context: The MIC's command reorder arbitration groups commands into
bursts to minimize R/W turnaround overhead (each turnaround costs tDeltaRW or
tDeltaWR cycles of dead bus time). Smaller ReadQ means reads get serviced sooner
(lower latency) at the cost of more turnarounds (lower peak throughput).
For gaming workloads (latency-sensitive), ReadQ=2 with WriteQ=16 favors reads.


## Entry 11: MIC_REF_SCB (NVS 0x314E, 4 bytes, MMIO 0x200)

Stock: `06 11 01 70`

Controls refresh timing. Rambus Table 10 specifies:
- tREF_MAX = 16 or 32 ms (vendor dependent)
- Elpida datasheet: 0.98 us refresh intervals (16K rows / 16ms)
- Samsung datasheet: identical 0.98 us intervals

The PS3 uses single-bank refresh during active operation (not simultaneous refresh).
From Rambus spec: "If a controller is to take full advantage of zero-overhead
simultaneous refresh, it must incorporate a request queue which keeps track of
the last three transactions and has the next four queued." The MIC does support
this per the IBM presentation (32-deep queues), but Sony chose single-bank for simplicity.


## Entry 22: MIC_TM_THRESHOLD (NVS 0x3178, 8 bytes, MMIO 0x0A8)

Stock: `ED D6 12 29 59 4B A6 B4`

Token Manager thresholds. From the IBM Cell MIC presentation:
"Resource Allocation Management -- critical resource's time is distributed among
groups of requestors. Managed resources include Rambus XDR DRAM memory banks (0 to 15)."

The Token Manager controls how bandwidth is distributed between the PPE, 8 SPEs,
and 4 I/O virtual channels (17 requestors total, organized into 4 RAGs).
Sony's values differ from HIG defaults (3B F0 77 E0...), meaning they tuned
bandwidth allocation for PS3's specific workload mix (heavy SPE DMA, RSX reads).


## Rambus Rules Verification: Tight-3 Profile at 466.7 MHz RefClk (3.7GHz)

Using Rambus DL-0161 Table 14 rules with current Tight-3 values:
tRCD-R=7, tRCD-W=3, tCAC=7, tCWD=3, tCC=2, tRDP=1, tWRP=11,
tDeltaRW=9, tDeltaWR=8, tDeltaWR-D=2, tRC=21, tRAS=14, tRP=7,
tQTD=3, tQRD=13, tERD=5, tPD_CYC=1 (assumed, short PCB traces)
tRW_BUB_XDRDRAM=3, tWR_BUB_XDRDRAM=3, tPP=4, tPP-D=1, tRR=4

### Required Rules (must pass for reliable operation):

| Rule | Goal | Equation | Result | Status |
|------|------|----------|--------|--------|
| R1 | ERAW W-R stagger | tCWD - tDeltaWR-D = odd | 3-2=1 (odd) | N/A (no ERAW) |
| R2a | Min R-W bubble at DRAM+XIO | tDeltaRW > (tCAC-tCWD) + tCC + (tPD_CYC-1) + tRW_BUB_min | 9 > 4+2+0+3 = 9 | MARGINAL (9 > 9 is false, 9 >= 9) |
| R2b | Min W-R bubble (ERAW) | tDeltaWR-D > (tCWD-tCAC) + tCC + tWR_BUB_min | 2 > -4+2+3 = 1 | PASS |
| R3a | Write data to core before PRE | tWRP > tCWD + tDP_MIN | 11 > 3+9 = 12 | FAIL (spec) |
| R3b | Write data to core before RD | tDeltaWR > tCWD + tDR_MIN | 8 > 3+7 = 10 | FAIL (spec) |

### Guidelines (optional, simplify MC design):

| Guideline | Goal | Equation | Result | Status |
|-----------|------|----------|--------|--------|
| G1a | Efficient tiling | tRCD-R = odd | 7 = odd | PASS |
| G1b | tRR = tPP = 4*n | tRR=4, tPP=4 | 4 = 4*1 | PASS |
| G2a | ERAW tiling | tRCD-W = tRCD-R | 3 != 7 | N/A (no ERAW) |
| G3a | No ROWA/COL-RD collision on R-W | tDeltaRW > tRCD-W | 9 > 3 | PASS |
| G3b | No ROWA/COL-WR collision on W-R | tDeltaWR > tRCD-R | 8 > 7 | PASS (margin=1) |
| G4a | tPP met on R-W turn | tDeltaRW > (tRDP-tWRP) + tPP + (tCC-4) | 9 > (1-11)+4+(2-4) = -8 | PASS |
| G4b | tPP met on W-R turn | tDeltaWR > (tWRP-tRDP) + tPP + (tCC-4) | 8 > (11-1)+4+(2-4) = 12 | FAIL |
| G5 | No TDATA conflict W-R | tDeltaWR > tQTD - tQRD + tERD + tCC | 8 > 3-13+5+2 = -3 | PASS |

### Analysis:

R2a is at exact zero margin - this is why tDeltaRW=8 crashes. The round-trip
flight time (tPD_CYC) sets a physics floor that cannot be overcome without
physically shortening the PCB traces between Cell and XDR DRAM.

R3a and R3b technically fail against C-bin datasheet values, but these rules
use worst-case tDP and tDR numbers. The actual DRAM silicon completes write
data propagation faster than the spec guarantees. This is normal - DRAM
vendors add margin to their specs.

G4b fails because it assumes you need tPP cycles between a precharge caused
by a write and a precharge caused by a read. In practice the MIC handles this
through its command reorder logic, and the Rambus spec explicitly labels G4b
as "not likely to ever be a problem" (Table 14 footnote b).


## What Can Still Be Tightened

Based on the Rambus rules analysis:

1. **tDeltaRW: HARD WALL at 9.** Bound by R2a and round-trip flight time.
   Cannot go lower without modifying PCB trace lengths (physically impossible).

2. **tDeltaWR: HARD WALL at 8.** Bound by G3b (must be > tRCD-R=7).
   Going to 7 would make tDeltaWR = tRCD-R, causing ROWA/COL collisions.

3. **tRC: SOFT WALL at 21.** Bound by DRAM core tRAS + tRP minimum.
   Could potentially reach 20 with voltage increase (more sense amp current),
   but intermittent corruption at 20 suggests ~43 ns is the true silicon limit.

4. **tRDP: AT MIC HARDWARE MINIMUM of 1.** Cannot go lower.

5. **tWRP: SOFT WALL at ~11.** Each step below 12 risks write data corruption.
   The 0x83/0x82 boundary is the practical silicon limit.

6. **tRCD-R: HARD WALL at 7.** Next step is 5 (only odd values), which crashes.
   5 * 2.143 = 10.7 ns, below even A-bin absolute minimum of 12.5 ns.

7. **tCAC (entry 18/MIC_DF_CTL): LOCKED.** Cannot change without XIO re-init.

8. **Entry 22 (TM_THRESHOLD): OPEN.** Could be tuned for workload-specific
   bandwidth allocation. Needs a benchmark to measure impact.

9. **Entry 21 (BurstSize): TUNED.** Current ReadQ=2/WriteQ=16 favors latency.
   Could try ReadQ=1 for even lower read latency at cost of throughput.

10. **Entries 40-45 (per-channel): PARTIALLY OPEN.** Entry 44 accepts a wide
    range (0x13-0x33). Could be per-DQ-group drive strength or timing skew.
    The Rambus Co-Design paper confirms per-bit phase control in XIO.
    Entry 45 accepts 0x00-0xFF and could be a phase offset value.


# Part 7: Config Ring System

## Cell BE Config Ring System

### What is the Config Ring?

The Cell BE config ring is a serial scan chain (~2700 bits / ~338 bytes on 90nm, documented in Table 4-1 of the 90nm HIG across 14 pages) that configures the chip's internal hardware during Power-On Reset (POR). It's shifted in via SPI before the Cell begins executing code. It programs:

- FlexIO PLL configuration (BIF/IOIF transmit and receive)
- XIO PLL configuration (XDR I/O cell phase-locked loop)
- EIB (Element Interconnect Bus) parameters
- MMIO base address mapping (BE_MMIO_Base)
- IOIF base addresses and masks
- AC0/AC1 address concentrator configuration
- Bank enable/disable masks
- SPE count restrictions
- Various feature flags and operating modes

**Note**: The core PLL multiplier (NClk) is NOT in the config ring - it's fuse-programmed (see 90nm HIG comparison section above).

### Config Ring Source Selection

NVS 0x3C00 controls the source of the config ring data:

- **0xFF**: Build from ROM defaults, then apply all NVS patches (normal path).
- **Non-0xFF**: Load the entire raw config ring from NVS 0x3C01 onwards. This gives COMPLETE control over every bit in the config ring.

### NVS Patch System

When building from ROM defaults (0x3C00 == 0xFF), 6 independent patch sources are applied sequentially. Each patches specific bit positions in the ~338-byte config ring buffer:

| NVS Offset | Size | CR Bit Start | Bit Count | Lookup Table | Description |
|------------|------|-------------|-----------|--------------|-------------|
| 0x3970 | 1 | 0xA52 (2642) | 6 | dword_200C9CC | XIO/calibration config |
| 0x3971 | 4 | 0x414 (1044) | 32 | dword_200C9D8 | Core/bus configuration (large field) |
| 0x3975 | 2 BE | 0x968 (2408) | 16 | dword_200C9D4 | PLL/clock divider config |
| 0x3977 | 2 BE | 0xA2E (2606) | 16 | dword_200C9DC | I/O timing config |
| 0x3979 | 2 BE | 0xA3E (2622) | 16 | dword_200C9E0 | I/O timing config |
| 0x39B0 | 64 | variable | 1-40 per entry | scatter table | Arbitrary bit patches (8 entries) |

All patches use 0xFF... as "no patch" sentinel.

### Key Config Ring Bit Positions

From the firmware initialization (`sub_1ABA6`):

| Global | Value | Bits | Used By | Purpose |
|--------|-------|------|---------|---------|
| dword_200C9B8 | 0xA89 (2697) | - | Bank enable | XDR bank enable mask |
| dword_200C9BC | 0xA7E (2686) | 8 | XIO PLL init | XIO PLL configuration |
| dword_200C9C0 | 0x40F (1039) | - | Config ring build | Feature enable bit |
| dword_200C9C4 | 0xA86 (2694) | 3 | VRM sense | VRM mode select (2=normal, 3=alt) |
| dword_200C9C8 | 0x801 (2049) | 16 | Hardcoded | Fixed config bits |
| dword_200C9CC | 0xA52 (2642) | 6 | NVS 0x3970 | XIO/calibration |
| dword_200C9D0 | 0x875 (2165) | 16 | Hardcoded | Fixed config bits |
| dword_200C9D4 | 0x968 (2408) | 16 | NVS 0x3975 | PLL/divider |
| dword_200C9D8 | 0x414 (1044) | 32 | NVS 0x3971 | Core/bus config |
| dword_200C9DC | 0xA2E (2606) | 16 | NVS 0x3977 | I/O timing |
| dword_200C9E0 | 0xA3E (2622) | 16 | NVS 0x3979 | I/O timing |

### The Scatter Table (NVS 0x39B0)

The most powerful config ring override mechanism. 8 entries of 8 bytes each:

```
Per entry:
  [0-1]  Start bit position (2 bytes, big-endian)
  [2]    Bit count (1-40 bits)
  [3-7]  Data bits (5 bytes, enough for 40 bits)
```

If byte 0 == 0xFF, the entry is skipped. Bit count must be 1..40 and the range must fit within the config ring.

This allows patching ANY arbitrary range of up to 40 bits at ANY position in the ~2700-bit config ring. Combined with the ability to load a completely raw config ring from NVS 0x3C01, this gives total control over the Cell's internal configuration.

---

## 90nm vs 65nm HIG Comparison

### MIC Timing Parameter Ranges: IDENTICAL

Both the 90nm HIG (v1.5) and 65nm HIG (v1.01) have the exact same Table E-1 timing parameter ranges. The MIC is functionally identical between the two process nodes. Same register offsets, same bit fields, same supported ranges.

### Sample Configuration: IDENTICAL

Both HIGs use the same Table E-3 sample configuration (3.0 GHz NClk, 1.5 GHz MiClk) with the same timing values (tRC=24, tRAS=17, tRP=7, etc). Both produce the same Table E-4 register values.

### Key Difference: 90nm HIG Has Actual Init Code

The 90nm HIG includes complete C pseudocode for the entire MIC initialization sequence (Section 2.2.2), which the 65nm HIG lacks. This gives us the actual MMIO addresses and real register values used during boot.

### Firmware ROM Defaults vs HIG vs PS3 Init Code

| Register | MMIO Offset | HIG Table E-4 | Firmware ROM Default | PS3 Init Code (90nm) |
|----------|-------------|---------------|---------------------|---------------------|
| MIC_Trcd_Pchg | 0x0D0/0x190 | 0x6284055AD6B0 | **62 84 05 5A D6 B0** | **0x6284055AD6B00000** |
| MIC_Cmd_Dur | 0x0D8/0x198 | 0x5D700000 | **5D 70** | **0x5D7000000000** |
| MIC_Cmd_Spc | 0x0E0/0x1A0 | 0x71800210 | **71 80 02 10 00 00** | 0x71841A10 |
| MIC_DF_Ctl | 0x0E8/0x1A8 | 0x0A543CE0 | **0A 96 3D 60** | **0x0A543CE00000** |
| MIC_Que_BurstSize | 0x0B0/0x1F0 | 0x23000000 | 00 00 00 00 00 00 00 00 | 0x23000000 |
| MIC_TM_Threshold | 0x0A8/0x1E8 | 0x3BF077E0 | ED D6 12 29 59 4B A6 B4 | 0x09127754 |
| MIC_Ref_Scb | 0x200 | 0x05B04058 | 06 11 01 70 | 0x06104058 |
| MIC_Mnt_Cfg | 0x210 | 0x752E0000 | 7C FE (partial) | 0x7CFE0000 |

**MIC_Trcd_Pchg and MIC_Cmd_Dur match exactly** - the firmware ROM defaults ARE the HIG sample values. These control the core row/column timing (tRC, tRAS, tRP, tRCD, tRDP, tWRP, calibration periods).

**MIC_DF_Ctl differs** - firmware has 0x0A963D60 vs HIG sample 0x0A543CE0. This is the dataflow control register containing tQTD, tQRD, tERD. Sony tuned this for the actual PS3 XDR chips.

**MIC_TM_Threshold differs significantly** - the token manager thresholds are completely custom. This controls read vs write arbitration priority.

### Critical: Core PLL Multiplier

From the 90nm HIG (Section 2.2, POR Phase 1):

> "The PLL configuration register will be set up from internal storage (fuses) as a result of the SYS_CONFIG[0:3] pins being set to '0000'. The PLL configuration register is an internal register that is not accessible in the MMIO register space."

The core PLL multiplier (which converts 400 MHz reference -> 3.2 GHz NClk) is set from **fuses** during phase 1 of POR, controlled by SYS_CONFIG pins. It is NOT in the config ring and NOT in MMIO. The config ring handles FlexIO PLL and XIO PLL configuration, but the core clock multiplier comes from fuses.

This means: changing the external clock from 400 to 500 MHz **will** push NClk from 3.2 GHz to 4.0 GHz, and you **cannot** reduce the core multiplier via software. The only way to change the core multiplier is hardware modification of the SYS_CONFIG pins (which are documented as "tied to ground for typical system operation" per Table 5-12 in both HIGs).

### PLL Reference Clocks (90nm HIG Section 1.4.2)

Three independent PLL reference clock inputs:
- **PLL_REFCLK** - Core PLL. Multiplied to get NClk (3.2 GHz). Shares the same external 400 MHz oscillator as XDR.
- **Y0_RQ_CTM / Y1_RQ_CTM** - XIO PLL per channel. Runs XDR at the reference clock frequency (400 MHz -> 3.2 Gbps octuple data rate).
- **RC_REFCLK** - FlexIO PLL. Independent reference for the Rambus FlexIO bus (RSX interconnect). **This is separately controllable via NVS 0x312C/0x312D.**

---

## Overclocking Implications

### What Can Be Controlled

| What | NVS Bytes | Range | Effect |
|------|-----------|-------|--------|
| Cell/XDR base clock | 0x3128, 0x3129 | 300-667 MHz | Changes Cell + XDR freq together |
| Cell core voltage | 0x3110, 0x361C | VID lookup | Higher voltage for stability |
| RSX voltage | 0x3111, 0x361D | VID lookup | Higher voltage for stability |
| XDR timings | 0x3140-0x31AF | MIC register values | Memory latency/bandwidth |
| XDR per-lane cal | 0x31B0-0x31BF | 16 bytes | Per-lane timing trim |
| Spread spectrum | 0x3122 | 0x00-0x7F | Disable for cleaner clock |
| FlexIO/XIO PLL | Config ring | Patch bits | FlexIO and XIO PLL tuning |
| Config ring (full) | 0x3C00-0x3DFD | ~338 bytes | Total Cell internal config (except core PLL) |

**Cannot be controlled via NVS:** Cell core PLL multiplier (fuse-programmed, requires SYS_CONFIG pin hardware mod).

### Practical Considerations

1. **Clock change affects Cell AND XDR together.** The core PLL multiplier is fuse-programmed (from 90nm HIG), NOT in the config ring or MMIO. Increasing the external clock from 400 to 500 MHz WILL push NClk from 3.2 GHz to 4.0 GHz. You cannot reduce the multiplier via software. The SYS_CONFIG pins that control fuse behavior are tied to ground on the motherboard.

2. **XDR timing tightening at stock frequency** is the safest and most practical approach. The firmware ROM defaults for MIC_Trcd_Pchg (tRC/tRAS/tRP/tRCD timing) EXACTLY match the HIG demo config, which was designed for a generic 3.0 GHz/1.5 GHz system. The actual PS3 runs at 3.2 GHz/1.6 GHz. Sony may have left timing margin on the table by not optimizing for the higher clock rate.

3. **MIC_DF_Ctl is already Sony-tuned.** The firmware default (0x0A963D60) differs from the HIG sample (0x0A543CE0), meaning Sony already customized the dataflow timing (tQTD/tQRD/tERD) for PS3-specific XDR chips. This register is XIO-coupled and cannot be changed independently -- both tQRD+1 and tQRD-1 crash with XDR link failure.

4. **All timing changes are loaded at boot** - the syscon serves the config to lv0 during the POR sequence. Changes take effect on next power cycle.

### Example: Tightening XDR Latency

To reduce XDR memory latency:

1. Write tighter timing values to 0x3160 (MIC_Cmd_Dur) - reduce tRC, tRAS, tRP
2. Run `eepcsum` to update checksum
3. Power cycle to apply

Benchmark testing confirmed that MIC_CMD_DUR (entry 16) is the only register that produces measurable read latency improvement. Other registers (MIC_TRCD_PCHG for tRDP/tWRP, MIC_CMD_SPC for tDeltaWR, MIC_Que_BurstSize for read/write arbitration) showed zero standalone impact despite being technically valid timing changes.


# Source Documents

| Document | Description |
|----------|-------------|
| CellBE HIG 90nm v1.5 (Nov 2007) | Hardware Initialization Guide. Appendix E (MIC config rules), Section 2 (POR sequence) |
| CellBE HIG 65nm v1.01 (Jun 2007) | Slow Core Mode MvWrDelay/MvRdDelay formulas |
| CBE Public Registers v1.5 (Apr 2007) | Section 7: Complete MIC MMIO register bitfield definitions |
| Cell BE Programming Handbook v1.1 (Apr 2007) | Section 13: Clock domains, timebase mechanism |
| CBEA v1.01 (Oct 2006) | Section 9.7: SPU decrementer requirements |
| Rambus XDR Spec Sheet DL-0161 v0.8 (Dec 2005) | Tables 13-15: Architectural timing rules and ranges |
| Elpida EDX5116ACSE Datasheet E0881E20 v2.0 | Table 17: XDR DRAM timing parameters per speed bin |
| Samsung K4Y5416 Datasheet v1.0 (Jan 2005) | Table 18: XDR DRAM timing parameters |
| IBM Cell Memory Connect Interface (ASP-DAC) | MIC queue architecture, closed-page policy |
| Rambus System Co-Design and Co-Analysis Approach to Implementing the XDR™ Memory System of the Cell Broadband Engine™ Processor (ASP-DAC 2007) | PS3-specific XIO design, per-bit FlexPhase |
| Rambus XIO Link (VLSI Design 2009) | XIO PLL, FlexPhase phase mixers, jitter data |
| CXR713F syscon firmware (reverse engineered) | NVS layout, descriptor table, clock config |
| lv0 firmware (reverse engineered) | Clock queries, timebase config, config string builder |
| lv1 firmware (reverse engineered) | Repository hash table, timebase consumers |
| The thousands of forums posts about RAM overclocking I scrolled through in ocean of Firefox tabs I had open | Various abbreviations and hardware rules |
| Whatever JEDEC DRAM documents I could find | Terminology and other stupid hardware rules


# Credits

- [M4j0r](https://x.com/MinaRalwasser/) - Discovering Cell can be overclocked via DECR-1000A back in [2021](https://x.com/MinaRalwasser/status/1458862608384155650)
- [Nascar1243](https://youtube.com/@nascar1243) - Figuring out the offsets for the Cell clock generator registers
- [aomsin2526](https://github.com/aomsin2526/) - [CellOCPico](https://github.com/aomsin2526/CellOCPico) project and spearheading software-based overclocking
- [RIP Felix](https://www.youtube.com/@ripfelix3020) - Testing clock generator register values
- [villahed94](https://www.youtube.com/@villahed94/) - Testing clock generator register values
- [Sampsonay](https://www.youtube.com/@Sampsonay/) - Testing clock generator register values
- [gypsy](https://www.github.com/losgatosbandidos/) - Testing clock generator register values
- [RGBeter](https://x.com/RGBeter32X) - Testing clock generator register values
- [sage](https://github.com/sagemono/) - Reverse engineering, documentation, XDR timing research
- PS3DevWiki contributors - Hardware documentation
- graf_chokolo - Syscon communication research

---

This documentation is provided for educational and research purposes. The information contained herein is derived from reverse engineering for interoperability and repair purposes as permitted by law.
