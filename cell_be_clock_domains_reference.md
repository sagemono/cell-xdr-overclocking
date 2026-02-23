# Cell Broadband Engine Clock Domains, Timebase, and FlexIO

## Source Documents Referenced

All claims in this document are cited with exact document name, section, and page number.

| Abbreviation | Full Title | Version/Date |
|---|---|---|
| **Handbook** | Cell Broadband Engine Programming Handbook | v1.1, April 24, 2007 |
| **CBEA** | Cell Broadband Engine Architecture | v1.01, October 3, 2006 |
| **HIG** | CellBE Hardware Initialization Guide (90nm) | v1.5, November 30, 2007 |
| **Registers** | Cell Broadband Engine Public Registers | v1.5, April 2, 2007 |

---

## 1. The Three Clock Domains

The Cell BE processor operates with three independent, asynchronous clock domains.

**Source: Handbook, Section 13.2.1 "Clock Domains", Page 375:**

> The CBE processor has three clock domains, each running asynchronously to the other two domains:
>
> - **CBE Core Clock (NClk)** -- This clock times the PowerPC processor unit (PPU), the SPUs, and parts of the PowerPC processor storage subsystem (PPSS) and the memory flow controllers (MFCs). The CBE core clock (NClk) is occasionally referred to as the core clock (CORE_CLK) or core clock frequency (CCF).
> - **MIC Clock (MiClk)** -- This clock times the memory interface controller (MIC).
> - **BIC Core Clock (BClk)** -- This clock times the bus interface controller (BIC), which is part of the Cell Broadband Engine interface (BEI) unit to the I/O interface.

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

Yes, but only as a stability constraint. From the main overclock research writeup [here](https://github.com/sagemono/cell-xdr-overclocking/blob/main/README.md):

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

## 5. Can We Control the Cell Multiplier (CCM)?

### 5.1 What the Documentation Says

The public documentation describes x8 as the standard multiplier but does not explicitly state it is immutable.

**Source: HIG, Table 5-10 "Core PLL Pins", Page 156:**

> PLL_REFCLK: PLL reference clock. These differential input pins provide the reference clock input to the core PLL. Typically, this will be a 400 MHz differential clock generated by the XDR clock generator. The PLL multiplies the reference clock frequency by **eight** to produce the very low jitter 3.2 GHz clock (NClk) that is distributed on the internal clock grid.

Note the word "Typically." The HIG describes the standard operating point, not necessarily the only possible configuration.

**Source: Handbook, Section 13.2.3, Page 377:**

> The standard 3.2 GHz CBE core clock (NClk) is configured by specifying the Internal Time-Base Sync State, a 400 MHz PLL reference clock (PLL_REFCLK) frequency, and a core clock multiplier (CCM) of x8.

Again, "standard", not "fixed" or "hardwired."

The Handbook treats CCM as a *variable* in its formulas. The NClk equation `NClk = PLL_REFCLK x CCM` and the Fmax equation `RefDiv >= ceil(MaxSlowModeNclkDivider x 11 / CCM)` both use CCM as a parameter, not a constant.

### 5.2 The Two Clock Paths on Hardware

On retail hardware, the primary mechanism for changing the Cell clock speed is modifying PLL_REFCLK via the external clock generator (ICS9218AGLFT):

```
Path 1 (Retail): NVS 0x3128/0x3129 -> XCG2 registers -> I2C -> IC5003 -> PLL_REFCLK pin
```

On DECR-1000A development hardware, there is a second path that sends configuration bytes directly to the Cell's internal PLL via the SPI scan chain:

```
Path 2 (DECR): NVS 0x3070-0x3077 -> syscon -> SPI -> Cell PLL configuration register
```

This second path is accessed through the `Psbd_SetBePll` function in the syscon firmware. The 8-byte PLL configuration is bit-rearranged into 9 bytes and sent via SPI command to the Cell's internal scan chain (register 0x20).

### 5.3 What We Know and Don't Know About BE_PLL

Experimental data from DECR-1000A testing:

**3.2GHz configuration:**
```
w 3068 84 16           <- XCG2 CELL reg5=0x84, reg6=0x16 (400 MHz refclk)
w 3070 FF FF FF FF FF FF FF FF FF   <- BE_PLL override DISABLED
```

**4.8GHz configuration:**
```
w 3068 FF FF           <- XCG2 bypassed
w 3070 71 47 6A 81 63 48 00 00 00   <- BE_PLL override with custom PLL config
```

Observed behavior: the "fast" configuration results in 55 FPS reported instead of 60, indicating the clock runs faster than what the timebase/software expects.

**What we CAN say with confidence:**

1. The BE_PLL SPI path exists and is functional on DECR hardware.
2. It configures the Cell's internal PLL independently of the external clock generator.
3. The resulting clock speed differs from stock (proven by FPS measurement).
4. The 8 bytes are sent to the Cell's PLL configuration scan chain.
5. This path fails on retail hardware because the `ATTENTION` handshake signal (the Cell telling syscon it's ready to receive PLL commands) does not assert.

**What we CANNOT say with confidence:**

1. Whether these bytes change the PLL multiplier (CCM), the reference clock sensitivity, the feedback divider, the charge pump, the VCO range, or some combination.
2. What the exact bit layout of the 8-byte PLL configuration is. This appears to be proprietary IBM/Sony PLL IP and is not documented in any public Cell specification.
3. Whether the resulting NClk frequency is `(same refclk) x (different multiplier)` or `(different refclk behavior) x 8` or something else entirely.

**The honest answer:** The BE_PLL bytes program the Cell's internal PLL through SPI. They produce a different clock speed. The documentation uses CCM as a variable (not a hardwired constant) in its formulas. But without reverse engineering the exact bit-field layout of the PLL scan chain, we cannot definitively state whether the multiplier is being changed. Claiming "the multiplier is always fixed at 8x" is not supported by the documentation either, it says "typically" and "standard", not "always" or "fixed."

### 5.4 The 0x84 0x16 Values

From the DECR `boardconfig` command and NVS analysis, these are *clock generator register 5 and register 6 values* for the ICS9218AGLFT (or equivalent) chip:

```
XCG2 BE 5:84 6:16  <- This is at NVS 0x3068-0x3069 on DECR
```

From the syscon firmware's frequency lookup table [here](https://github.com/sagemono/cell-xdr-overclocking/tree/main?tab=readme-ov-file#clock-generator-table-0x40188-25-entries), the combined key `0x8416` maps to exactly 400,000,000 Hz. The full table shows 25 frequency entries ranging from 300 MHz (0x8410) to 667 MHz (0x0426). These are the clock generator's output frequencies, NOT the Cell's internal PLL multiplier settings.

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

**A. Software-only (CFW patch):** Hook syscall 147 to return the actual timebase frequency. Games then do correct math. This is what our Cobra patch does.

**B. Hardware divider change:** Write a new Timebase_setting to the TBR MMIO register to change RefDiv so that `overclocked_refclk / new_RefDiv` is as close to 80 MHz as possible. Requires lv1 level access to write `BE_MMIO_Base + 0x509890`.

**C. Choose refclk where math works out:** Find overclock frequencies where `refclk / integer = 80`. The candidates from the syscon frequency table: 400 MHz (stock, /5), 480 MHz (/6, not in table), 560 MHz (/7, not in table). The closest table entries that give near-80 MHz: 475 MHz/6 = 79.17 MHz (-1.04%), 483 MHz/6 = 80.56 MHz (+0.7%).

---

## 8. PS2 Emulation and Timebase

PS2 emulators run as Guest OS directly on lv1, with lv2 (GameOS) completely unloaded. A syscall 147 hook in lv2 does nothing for PS2 emulation because lv2 isn't running.

**Source: PS3 Dev Wiki, PS2 Emulation page:**

> These emulators are not truly PS3 Game OS .elf executables, but rather Guest OS' that run on the LV1 hypervisor of the PS3. This means that the LV2 kernel or the more user friendly Game OS is unloaded before the selected emulator is loaded.

Fixing PS2 emulation timing requires either patching lv1 directly (so the corrected frequency is available to the Guest OS), or patching the PS2 emulator binaries themselves. Someone would need to reverse engineer where the emulator reads/assumes the timebase frequency.
