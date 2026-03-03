# lv0 + lv1 Timebase Mechanism

## 1. Summary

**tb_clk is hardcoded in lv0.** The value 79,800,000 is a literal constant at `0x8006EF0` in lv0, never computed from the actual reference clock. Meanwhile, `be.0.nclk` and `be.0.ref_clk` are dynamically queried from syscon. This is the root cause of the timing bug after overclocking: the hardware timebase physically changes frequency but lv0 always reports 79.8 MHz to lv1.

The TBR hardware register IS configured dynamically (lv0 computes RefDiv from the actual ref_clk), so the hardware ticks at the correct rate. The problem is purely in the reported metadata.

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

### 10.4 lv2 Cached Value (This section may be incorrect)

~~lv2 reads tb_clk at boot via hypercall and caches it. Patch the cached copy in lv2 kernel memory for immediate effect on syscall 147.~~

### 10.5 Timing

- Patch lv1 repo node: affects all future LPAR constructions (PS2 emu, Linux)
- Patch lv2 cache: immediate effect on GameOS syscall 147
- Both should be done early, ideally from a Cobra payload at boot (PoC working!)

---

## 11. Potential lv0 Patch (Firmware Modification)

For a permanent fix in modified firmware, patch `build_clock_config_string` at 0x8006EF0 to compute tb_clk dynamically instead of using the literal 79,800,000:

```
Original:  config_append("be.0.tb_clk", 79800000)
Patched:   config_append("be.0.tb_clk", ref_clk / (ref_clk / 79800000))
```

This would require modifying lv0, which is encrypted and signed. Much harder than runtime lv1 patching but would be a complete fix at the source.

Alternatively, the `configure_timebase_register` function already has the right formula. Its computed RefDiv could be reused: `tb_clk = ref_clk / refdiv_index`.
