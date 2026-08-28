# Research: Speeduino schematic comparison (ATmega2560 vs STM32F407) for custom Blackpill ECU PCB

Source files reviewed:
- `pref/schematic_official_board_v0.4.3d.pdf` — official Speeduino v0.4.3d (ATmega2560, 5V logic)
- `pref/schematic_Arduino_Mega_F407_1.3.pdf` — pazi88 "Arduino Mega STM32F407" board, a Mega-footprint replacement module (STM32F407VET6, 3.3V logic) that plugs into the same Mega-format Speeduino main PCB
- `pref/schematic_Black_Pill_V2.0.pdf` — WeAct "MiniF4" (Blackpill) V2.0 module schematic, the actual MCU module this project is built around
- `pref/datasheet/SP721.pdf` — **now verified** (the Read tool renders PDFs directly; the earlier "couldn't text-extract" note was from before that was discovered). Full specs in §4.
- `pref/datasheet/sn74ahct541.pdf` — still not independently re-verified; details on this one part are from general datasheet knowledge
- `pref/beat-mcu.md`, `pref/beat-ecu.jpg` — Honda Beat factory ECU connector pinout, used as the target harness pinout for this custom build (see §7)

Goal: figure out what to reuse/adapt when designing a custom PCB around a Blackpill (STM32) board instead of an ATmega2560.

## 1. Official board (ATmega2560, 5V logic) — key I/O ICs

| IC | Role | Notes |
|---|---|---|
| Trig Cond Socket (IC3, socketed) | Conditions VR/Hall crank & cam signals into clean digital edges (D18/D19) | Socketed; JP2/JP3 select VR vs Hall pull-up mode |
| SP721 ×2 (U1, U3) | TVS diode array, clamps 6 ADC lines each to VDDA/GND | ESD/transient protection on analog sensor inputs |
| MPX4250 (U5) | Integrated MAP pressure sensor | Analog output straight into MCU ADC via RC filter |
| TC4424A ×2 (U2, U4) | Dual 3A high-speed MOSFET gate drivers | Drives ignition module trigger lines with fast, clean, high-current edges |
| Discrete N-MOSFET (Q1–Q4) | Low-side injector drivers | Logic-level MOSFET switch per injector; series diode + LED provides flyback + indicator |
| ULN2003 (U6) | 7-channel Darlington array | Low-side driver for idle valve, fuel pump, fan, tacho |
| Pololu DRV8825 breakout | Stepper driver | Drives idle-air-control stepper motor |
| Discrete NMOS (Q5/Q6, "HVM") | High-current spare outputs | Same low-side switch pattern as injectors |
| LM2940S-5.0 | 5V LDO | Regulator from 12V-SW rail |

Everything here is native 5V logic (VDD=5V, VDDA=5V).

### Digital inputs on the official board (passive, no active buffering except crank/cam)

| Signal | Circuit | Pin |
|---|---|---|
| Crank (CRANK-IN) | IC3 conditioner + 1k pull-up (R41) to VDDA | MCU-D19 |
| Cam (CAM-IN) | Same conditioner socket, JP3 selects VR/Hall | MCU-D18 |
| Clutch | 1k pull-up resistor (R37) only | MCU-D51 |
| Flex sensor | RC filter only (R33 2.4k / C5 0.22µF) | MCU-D2 |

## 2. STM32F407 reference board (pazi88) — adapting to 3.3V logic

| IC | Role | Why it's there |
|---|---|---|
| STM32F407VET6 | 32-bit MCU, 3.3V logic | More ADC channels/resolution, hw timers, native CAN, much faster than ATmega2560 |
| ~16× quad comparator/op-amp units (U3–U6, LM339-style pinout) + 470Ω/910Ω dividers | Scales 5V-referenced sensor signals down into STM32's 3.3V ADC range, buffered per channel | STM32 ADC max input is VDDA (3.3V); feeding 5V straight in risks exceeding absolute max |
| SN74AHCT541 ×3 (U7, U8, U9 — octal buffer/line driver, powered from +5V) | Level-shifts 3.3V GPIO outputs up to a clean 5V logic swing | Drives the existing 5V-logic injector/ignition/idle output stages unmodified. Covers ~24 of the digital I/O pins (D4–D12 via U7; D12/D26/D28/D30–D33/D35 via U8; D37/D39/D41/D42/D45/D47/D49/D50 via U9) |
| MCP2551 CAN transceiver + SP0503BAHTG TVS array | Native CAN bus | Not present on the ATmega board at all |
| W25Q16 SPI flash + microSD slot | Onboard datalogging/storage | Not present on ATmega board |
| FT230XS USB-UART bridge | USB-serial for programming/tuning | Needed because this design uses a bare STM32F407VET6 with no onboard USB-serial. **Not needed on a Blackpill**, which already exposes native USB (OTG FS) |
| LM3940IMP-3.3 | 3.3V LDO for MCU core | |
| CR1220 + VBAT | RTC backup battery | Feature the ATmega board doesn't have |
| 8MHz HSE crystal + 32.768kHz crystal | Precise timing reference + RTC | Important for ignition timing accuracy |

### Digital inputs on the STM32 board — traced directly from the rendered schematic

Confirmed by rendering the PDF to images (`pymupdf`) and tracing pin labels — text extraction alone was ambiguous about buffer direction.

| Signal | STM32 pin | Path |
|---|---|---|
| CAM (D18) | PD4 | Direct to J1 connector, **unbuffered** |
| CRANK (D19) | PD3 | Direct to J1 connector, **unbuffered** |
| Clutch (D51) | PB8 | Direct to J1 connector, **unbuffered** |
| Flex (D2) | PD7 | Direct to J1 connector, **unbuffered** |

Contrast with digital **outputs**, which are explicitly routed through the SN74AHCT541 buffers (U7/U8/U9) before reaching the header.

**Why the asymmetry:** this board is a Mega-footprint replacement that plugs into the *same* main Speeduino PCB carrying the original conditioning circuitry (trigger conditioner, dividers, SP721 clamps). Crank/cam/clutch/flex arrive at the header already conditioned and referenced to 5V, and the STM32 side connects them straight to GPIOs because STM32F407 pins are largely 5V-tolerant ("FT") pins. Outputs get buffered because the STM32's native output swing is only 0–3.3V and the downstream driver ICs (TC4424, ULN2003, DRV8825) need a proper 5V logic level. Leaving crank/cam unbuffered also avoids adding propagation delay/jitter to a timing-critical trigger signal.

## 3. Architecture-level tradeoffs

**ATmega2560 / 5V platform**
- Simple, proven, well documented by the Speeduino community
- Everything is natively 5V — no level-shifting needed anywhere
- Slower MCU, less RAM/flash, fewer ADC channels, no hardware CAN, coarser timer resolution for high-RPM ignition/injection scheduling

**STM32F407 (Blackpill-class) / 3.3V platform**
- Much faster core, more timers/ADCs, native CAN, more flash/RAM
- Every interface to the 5V ecosystem needs adapting: ADC scaling+buffering, digital level shifting, and TVS/clamp thresholds need re-checking against the STM32's lower absolute-max pin ratings
- More total parts (buffers, dividers, extra regulator rail) than the ATmega design

## 4. Design notes for the custom Blackpill PCB

### Analog inputs
- STM32 ADC pins are **not** 5V tolerant (abs max ~VDDA + 0.3V, ~3.6V). Don't feed 5V-referenced sensor signals (MAP/TPS/CLT/IAT/O2/battery-ref) straight into an ADC pin.
- Either add a resistor divider + buffer per channel (as pazi88 did) and keep sensor excitation at 5V, or re-reference sensor dividers to 3.3V from the start (simpler, but changes calibration math for ratiometric sensors — factor into firmware). **Decided: divider + buffer, buffer first** (see below) — keeps sensor excitation at 5V, matching the official board's calibration.

### Analog input chain — final per-channel design (6 buffered channels: MAP/TPS/CLT/IAT/O2/+1)
Visual reference: published as an Artifact, [Analog Input Chain](https://claude.ai/code/artifact/17192f6d-9966-490d-a39e-6c692dd6c81c) — full schematic with a component table.

**Chain order** (corrected after initially assuming divider-then-buffer; checked pazi88's actual schematic and it buffers first):
```
sensor pin → R1 (470Ω) → D1 (SP721, clamps to 5V/GND) → C1 (0.1µF, to GND)
           → R2 (330Ω) → U1 (MCP6002, unity-gain buffer)
           → R3/R4 (divider, 0–5V → 0–3.3V, per-channel values)
           → R5 (220Ω) → C2 (4.7nF, to GND) → STM32 ADC pin
```
Four zones by purpose: **Protection** (R1/D1/C1) → **Buffer** (R2/U1) → **Scale 5V→3.3V** (R3/R4) → **ADC isolation** (R5/C2).

**Op-amp: `MCP6002`** (Microchip, dual). Three hard requirements for this role, general datasheet knowledge (not locally verified — no `MCP6002.pdf` in `pref/datasheet/` yet):
- Rail-to-rail input *and* output — single 3.3V supply, signal swings near both rails. Rules out classic parts like `LM358` (input common-mode range doesn't reach the negative rail).
- CMOS input stage (picoamp-range bias current) — needed because the op-amp is fed from a divider/filter node; a bipolar-input part's higher bias current would introduce a real error through that source impedance.
- Unity-gain stable, since it's a plain voltage follower here.
- Get the extended-temp variant (-40°C to 125°C) given the under-dash/engine-bay environment. For genuine AEC-Q100 qualification instead of just extended commercial temp range, `TLV9302-Q1` (TI) is the properly-qualified equivalent.
- Package count option: since `MCP6004` is the quad version of the same cell, 2× `MCP6004` covers all 6 channels (2 spare) using 2 packages instead of 3× dual — same electronics, fewer parts, worth considering over the 3×-dual plan.

**Why buffer before dividing, not after**: buffering the raw sensor signal first gives the divider a near-zero-impedance source, so the divider's actual ratio stays accurate regardless of the sensor's own output impedance or harness/cable resistance. It also means the *divider's* output — not an op-amp's — is what directly drives the ADC pin, so `R3`/`R4`'s own Thevenin impedance needs to stay low (same sizing consideration as the bufferless battery-voltage divider, target ~1–1.5kΩ) since nothing buffers it afterward.

**Do R1/R2/R5 affect the reading?**
- **R1 and R2: no meaningful effect.** Both sit in series with the op-amp's CMOS input (picoamp-level bias current) — voltage drop is in the nanovolt range, utterly below the ADC's ~0.8mV/LSB resolution (12-bit over 3.3V).
- **R5: a small, real, but different kind of effect than C2 addresses.** C2 solves the *dynamic* problem (does the ADC's sample-and-hold cap have time to charge through R5 during its sampling window). Separately, R5 sits in the path of the STM32 ADC pin's own small input leakage current (a spec of the silicon itself — not locally verified against the exact STM32F4 datasheet number, but typically low-hundreds-of-nA to low-µA range), which creates a small **fixed, systematic voltage offset** (`V_error = I_leak × (R5 + R_divider)`), not noise or scatter. Calibratable if it ever proves significant (same category as `batVoltCorrect`-style trims), and automotive sensor readings don't typically need sub-mV precision anyway. If it's a concern, reducing R5 (e.g. to 100Ω) trades some protection margin for less offset. This exact R+C isolation pattern at an ADC input is standard MCU-vendor-recommended practice (e.g. ST app notes) precisely because this tradeoff is well understood and generally accepted.

### Battery reference voltage divider — ratio is fixed by firmware, not freely configurable
- Checked `speeduino/speeduino/sensors.cpp:749`: `fastMap10Bit(readAnalogSensor(pinNumbers.pinBat), 0, 245)` linearly maps the ADC's full-scale reading to a **hardcoded 0–24.5V** range. `sensors.cpp:191` forces `analogReadResolution(10)` on STM32 builds specifically so this 0–1023-based math stays valid across AVR and STM32 targets.
- The only TunerStudio-exposed adjustment is `configPage4.batVoltCorrect` (`config_pages.h:428`) — a small **additive offset** trim (`int8_t`, 0.1V units), not a scale/ratio control. It can correct minor resistor-tolerance error but cannot fix a fundamentally wrong divider ratio (an offset shifts the whole curve by a constant; it can't correct a multiplicative/scale error across the range).
- Required design rule: `divider ratio = VDDA / 24.5V`.
  - Official AVR board (VDDA=5V): ratio = 0.204 — matches its actual R29(1k)/R30(3.9k) divider (`1/4.9 = 0.204`) exactly, confirming this is a deliberate firmware/hardware pairing, not a coincidence.
  - Our STM32 board (VDDA=3.3V): ratio = 3.3/24.5 = **0.1347**.
- **Revised recommended values: R_top = 9.76kΩ (E96), R_bottom = 1.5kΩ** → ratio = 0.133 (~1.3% low, trimmable via `batVoltCorrect`). Gives 12V→1.60V, 14.4V→1.92V, 24.5V (firmware's assumed ceiling)→3.26V — safely under 3.3V VDDA with margin.
- **No op-amp buffer needed for this channel** — confirmed against the official board's own schematic, which drives its battery-reference divider (R30/R29 + RC filter) straight into the AVR's ADC pin with no buffer at all. Battery voltage is slow-changing and doesn't need the accuracy/bandwidth the other analog channels (MAP/TPS/etc.) get from a buffer.
- Original 64.9kΩ/10kΩ values (same ratio, ~7x higher resistance) are **not recommended without a buffer**: their Thevenin source impedance (~8.7kΩ) is much higher than the official board's own divider (~795Ω) and risks incomplete ADC sample-and-hold settling. The revised 9.76kΩ/1.5kΩ pair brings Thevenin impedance down to ~1.3kΩ, in the same range as the proven official design, safe to sample directly.
- **Protection for this channel needs two clamps, not one** — it's a special case vs. the other analog channels (see "Protection IC selection: `SP721`" below for why), and unlike them it also can't rely on a single clamp point:
  - **Clamp 1, before the divider, on the raw `12V-SW` line**: standalone automotive TVS, **standoff ~24–26V** (not part of the SP721 group — needs a proper automotive-rated part, e.g. in the `SM6T26A`/`SMAJ26A` class). Sits above the normal 9–16V operating range so it never triggers during legitimate driving, but clamps a real load-dump event (actual clamp voltage ~35–42V at surge current, higher than the standoff spec) before it reaches `R_top`. Protects `R_top` and everything downstream from the raw, unbounded fault voltage.
  - **Clamp 2, at the tap point between `R_top` and `R_bottom`**: tight clamp referenced to **3.3V** (VDDA), using a discrete rail-referenced dual-diode part — **decided on `PRTR5V0U1T`** (NXP), a purpose-built ESD protection diode, over the electrically-equivalent `BAT54S` (a general-purpose switching diode informally repurposed for this job). `PRTR5V0U1T` is explicitly characterized/rated for ESD/transient duty (IEC 61000-4-2), matching the "prefer parts rated for the actual failure mode" approach used elsewhere in this design (same reasoning as picking `SN65HVD230` over `MCP2551` for CAN). **Caveat: not locally datasheet-verified** — no `PRTR5V0U1T.pdf` in `pref/datasheet/` yet, so exact capacitance/ESD kV/peak-current numbers are from general part-family knowledge, not confirmed the way `SP721`/`TC4423A`/`ULN2003A` are. Add the datasheet before finalizing if exact numbers matter. Wire `VCC` to 3.3V (VDDA), not 5V, despite the "5V0" part-number suffix — that just denotes the max line voltage it's rated for, not a fixed internal threshold.
  - **Why both are needed, not just one**: if Clamp 1 alone let a 40V load-dump event through at its ~35–42V clamp voltage, the tap point would still see `40V × 0.133 ≈ 5.3V` — already past the STM32 ADC's ~3.6V absolute max. `R_top`'s division alone isn't enough to bring an already-clamped-but-still-elevated voltage down into safe range. Clamp 1 handles the big/slow transient, Clamp 2 handles what still gets through, at the voltage the ADC actually lives at.
  - **Extra cheap addition, copied from the official board**: a small series resistor (`R32`-equivalent, ~470Ω) between the tap point and the actual ADC pin, after Clamp 2 — one more layer of current-limiting closest to the MCU pin, on top of the tap-point clamp.
  - Not redundant with the car board's main 12V input protection (reverse-polarity/fuse/TVS) — that protects the whole board's power rail, but this sense line can still pick up local noise/transients from other loads sharing the same `12V-SW` net along the harness run, so local protection here is still worth having.
- This divider only sets normal-range scaling — it provides no transient protection on its own; the two clamps above are what actually handle that.

### Digital inputs (crank/cam/clutch/flex)
- STM32 GPIOs are largely 5V-tolerant (FT pins), so these can connect directly without a divider — but confirm the specific pins used are marked FT in the STM32F4 datasheet (`PC13`/`PC14`/`PC15` are **not** FT; a pin also loses 5V tolerance while configured as true analog).
- Considered "just reverse the AHCT541 buffer" for these inputs — **rejected**: `SN74AHCT541` is rated for 4.5–5.5V VCC only; running it at 3.3V to get a safe 3.3V output is out of spec, and its input clamp diodes (tied to its own VCC) would back-feed the 3.3V rail when fed a 5V signal, risking damage/rail corruption.
- Since this is a standalone custom PCB (not a shield plugging into the existing 5V main board), the simplest fix is to **re-reference the pull-ups to 3.3V from the start** — removes the need for any translator entirely. Recommended default.
- If a 5V-referenced front end must be kept (e.g. reusing a 5V-only VR/Hall conditioner chip), use a proper dual-supply level translator (`74LVC1T45`, `TXS0108E`) or a sized resistor divider — not a repurposed single-supply logic buffer.
- Keep crank/cam paths minimal (direct FT-pin wire, or a low-value divider) to avoid adding delay/jitter to timing-critical trigger edges.

### Raw 12V digital switch inputs (brake switch, A/C request, etc.) — different problem from the 5V-logic inputs above
- These are a genuinely different case from crank/cam/clutch/flex above — those arrive already at 5V logic; a raw 12V-switched harness signal is a different voltage regime entirely, and logic-level translator ICs (`SN74AHCT541`, `74LVC1T45`, `TXS0108E`) don't apply — all are rated to ≤5.5V absolute max input, nowhere near 12–18V.
- A plain resistor divider is the wrong default here: automotive 12V rails swing ~9V (cranking sag) to ~16V+ (charging ceiling), and STM32's fixed logic thresholds (VIH≈0.7×VDD≈2.3V, VIL≈0.3×VDD≈1V at 3.3V) don't leave room for one ratio that reads a clean HIGH at 9V *and* stays under 3.3V at 16V+ without relying on the TVS clamp to save normal (non-fault) operation.
- **Recommended: optocoupler (`PC817`)** instead. Input side: series resistor (~1–2.2kΩ) from the 12V line into the LED, sized to keep LED current in a safe ~10–20mA range at max expected voltage. Output side: pull-up resistor (~10kΩ) from 3.3V to the STM32 GPIO. The ON threshold is set by the LED's forward turn-on voltage (~1.2–1.5V), not by STM32's VIH — so it reads cleanly ON from a few volts through 18V+, no ambiguous zone at cranking-sag voltages. Bonus: full galvanic isolation from the harness. No separate Schmitt buffer needed — STM32 GPIOs already have built-in input hysteresis.
- Still needs the TVS clamp on this line at the car board's harness connector (per the dual-layer protection plan) — the opto's LED tolerates normal automotive swing with the series resistor, but a real load-dump transient (40–60V) is still outside its rating.
- Placement: MCU board (it's a "shifter for digital input," same bucket as the other level-shifting/protection circuitry).

### Transient/ESD protection (separate concern from level shifting)
- Buffers/op-amps/level-shifters only handle *valid-range* signals — they are not rated to survive automotive transients: inductive kickback from coils/injectors/relays, load dump (ISO 7637-2, up to 40–60V), ESD (kV-level), or wiring faults (short to battery+).
- Protection must sit **upstream** of the buffer/op-amp/level-shifter, not instead of it:
  ```
  harness pin → series resistor → TVS/clamp diode (to rail/GND) → filter cap → buffer/op-amp/level-shifter → MCU pin
  ```
- Apply this to: all analog sensor inputs (reuse SP721-style array or discrete TVS, as official board does), crank/cam VR input (own clamp at the connector, independent of the conditioner chip's own protection), and the clutch/digital switch inputs (still exposed to harness even though directly wired to the MCU).
- Digital outputs driving inductive loads (injectors, relays, idle valve) need **flyback diodes across the load** (see official board's injector drivers: series diode + indicator LED) — different mechanism from TVS clamps on the logic/sensor side.
- Main 12V input needs its own reverse-polarity protection + TVS/clamp + fuse, separate from per-pin protection.

### Protection IC selection: `SP721`, verified — and why the clamp goes before the divider, not after
- **`SP721`** (Littelfuse, same part the official board uses) — datasheet fully verified (`pref/datasheet/SP721.pdf`). 6 channels/package (8-pin SOIC/PDIP), each `IN` pin clamped to a shared `V+`/`V-` pair, triggering ~1 diode-drop (VBE) above `V+` or below `V-`. Its own datasheet lists both "Analog Device Input Protection" and "Microprocessor/Logic Input Protection" as intended uses — one part family covers both signal types. Key specs: 3pF input capacitance (negligible loading on analog signals), 4kV HBM direct discharge / 15kV air discharge (IEC 61000-4-2), ±3A peak (8/20µs), ±2A (100µs), ±5A (4µs). Datasheet Note 2 (automotive guidance): add a series resistor + ≥0.01µF bypass cap on the SP721's own `V+`/`V-` supply pins if that rail is exposed to the vehicle supply.
- **Quantity/grouping — final: 2× SP721** (revised down from an earlier 3× draft once battery voltage moved to its own discrete parts, below), split by voltage domain since mixing domains isn't possible (all 6 channels in a package share one `V+`):
  - 1× for **6 of the 7 analog channels** (MAP/TPS/CLT/IAT/O2/+1 more), `V+` = 5V, tied to the same rail exciting the sensors (not an independent reference — keeps the clamp threshold tracking the sensors' own valid range if that rail drifts)
  - 1× for the digital sensor inputs, `V+` = 3.3V (6-channel capacity: crank/cam/TDC/VSS/FR = 5 used, 1 spare)
  - **Battery voltage (the 7th analog channel) is handled separately with discrete parts, not an SP721 channel** — see below and the battery-reference-divider section above for the full two-clamp writeup.
- **Why clamp before the divider, not after**: the divider resistors themselves need protecting, not just the ADC pin. If the clamp sat after the divider, the raw fault voltage (a load-dump transient, a short to battery+) would hit the divider's top resistor directly with nothing ahead of it — the resistor's own voltage rating becomes the only thing standing between the fault and everything downstream, an unbounded risk. Clamping first means the divider never sees more than the clamped ~5.7V in the first place. This also keeps protection closest to the point of entry (a general EMC/ESD principle — less unprotected circuit length for a transient to couple into), and decouples the protection design from the scaling design (change the divider ratio later without re-deriving clamp/resistor stress).
- **Battery voltage needed a different approach entirely** — unlike MAP/TPS/CLT/IAT/O2 (genuine 0–5V ratiometric sensor outputs), the battery-ref signal *is* raw vehicle voltage (9–16V normally, up to firmware's 24.5V ceiling) — clamping it with a 5V-referenced SP721 the same way as the other channels would clamp continuously during completely normal operation. Decided on **two discrete clamps** instead of one SP721 channel — full writeup, part classes, and reasoning for why two are needed (not one) is in the battery-reference-divider section above. Short version: a ~24–26V-standoff automotive TVS before `R_top` (raw line), plus a 3.3V-referenced `PRTR5V0U1T` dual-diode clamp at the tap point between `R_top`/`R_bottom` — `R_top` (9.76kΩ) does double duty as the tap-point clamp's current-limiting resistor, same trick the official board's R30 uses.

### Do the digital outputs actually need a 3.3V→5V buffer? — reassessed, mostly no, now datasheet-verified
pazi88's F407 board buffers every digital output through `SN74AHCT541` (§2), but checked whether that's actually required for our chosen driver ICs, or just extra margin his design happened to have room for. `pref/datasheet/TC4423A.pdf` (Microchip DS21998B, covers the TC4423A/TC4424A/TC4425A family — same input stage) and `pref/datasheet/uln2003a.pdf` (TI SLRS027T) were added and confirm the numbers below directly. `DRV8825` and the injector MOSFET are still general knowledge / component-selection questions, not locally verified.

| Component | Input threshold (datasheet-verified where noted) | 3.3V CMOS output OK? |
|---|---|---|
| `ULN2003A` (idle/fan/pump/tacho) | **Confirmed directly by TI's datasheet**: "The ULN2003A device has a series base resistor to each Darlington pair, thus allowing operation directly with TTL or CMOS operating at supply voltages of 5 V or **3.3 V**." (SLRS027T §7.1) | **Yes — explicitly supported, no caveat needed** |
| `DRV8825` (idle stepper) | Pololu's datasheet specs logic inputs as native 3.3–5V (not locally verified) | Yes, no shifting needed |
| Discrete MOSFET (injectors, spare outputs) | Depends entirely on the specific MOSFET chosen | Yes, *if* a true "logic-level" MOSFET spec'd for full R_DS(on) at V_GS≈2.5–3.3V is selected — a component-selection question, not a level-shifting one |
| `TC4424A` (ignition driver) | **Confirmed by Microchip datasheet DS21998B p.3**: `VIH min = 2.4V`, fixed across the entire 4.5–18V VDD range (not temperature/VDD-scaled — datasheet explicitly markets it as "TTL/CMOS compatible... 2.4V to 18V" with 300mV hysteresis) | Marginal — 3.3V gives only 0.9V margin above the confirmed 2.4V min threshold, vs. 2.6V margin at 5V |

**Conclusion:** `ULN2003A` is now fully confirmed safe to drive directly from 3.3V — no buffer needed, per the manufacturer's own datasheet. The one path still worth keeping cautious about is **ignition, through TC4424A** — its 2.4V minimum input threshold is now datasheet-confirmed (not an estimate), and 3.3V logic only clears it by 0.9V versus 2.6V at 5V. Given it's the most timing/safety-critical signal and sits in the noisiest part of the harness, I'd keep a buffer on just that one path rather than the whole digital-output bank. This still meaningfully shrinks the MCU board's digital-output section versus pazi88's blanket-buffered approach (§2/§6).

### Power
- Blackpill's onboard 3.3V regulator has limited current capability — if feeding buffers, comparators, a CAN transceiver, etc. from 3.3V, add a board-level 3.3V LDO sized for the whole board (like the reference board's LM3940IMP-3.3) rather than relying on the Blackpill's onboard reg.

### Other
- Add a proper HSE crystal near the MCU if not already suited on the chosen Blackpill module — ignition timing benefits from a real crystal over the internal RC oscillator.
- Break out SWD (SWDIO/SWCLK/GND/3V3) and Boot0 even though Blackpill has USB DFU — useful for debugging.
- Keep AGND/VDDA decoupling tight at the MCU (2.2µF + 100nF at VREF+/VDDA, as the reference board does) — STM32's ADC is more noise-sensitive than the AVR's.

## 5. WeAct Blackpill (MiniF4) V2.0 module — findings from the schematic

Rendered `pref/schematic_Black_Pill_V2.0.pdf` to images (text extraction was ambiguous on the header pin layout) and confirmed the following.

### No protection on USB VBUS
- The USB Type-C connector's VBUS pins tie **directly** to the net labeled `5V`, with no series diode.
- That same `5V` net feeds the onboard 5V→3.3V LDO (U2) and is also broken out on the P1/P2 headers — the header's `5V` pin **is** VBUS, same electrical node.
- Consequence: a diode added on our own carrier board (between an external 5V source and this pin) can only block reverse current into that external source — it cannot stop current from continuing out the module's own internal (undiodeed) trace to the USB-C connector. So if the car board is powering the unit **and** a USB cable is plugged into a host at the same time, current can flow from the car board straight out to the host's VBUS pin. A single diode does not fully solve this; see decision below.

### USB D+/D- are also broken out on the header
- `PA11` (USB D-) and `PA12` (USB D+) are wired to the onboard USB-C connector through small 10Ω series resistors (R9, R7), and are **also** available as standard GPIO pins on the P1/P2 headers.
- This means the same MCU USB peripheral can be routed to a second, physically separate USB connector via the header, without needing the onboard USB-C connector's power path at all.

### Full header pinout (confirmed from the rendered schematic)
- **P1** (pins 1–20): PB12, PB13, PB14, PB15, PA8, PA9, PA10, PA11, PA12, PA15, PB3, PB4, PB5, PB6, PB7, PB8, PB9, 5V, GND, 3.3V
- **P2** (pins 1–20): 5V, GND, 3.3V, PB10, PB2, PB1, PB0, PA7, PA6, PA5, PA4, PA3, PA2, PA1, PA0, NRST, PC15, PC14, PC13, VB
- Already spoken for: USB (`PA11`/`PA12`), SWD (`PA13`/`PA14`, not on the header at all — onboard debug only), onboard SPI1 flash (`PA4` CS, `PA5` SCK, `PA7` MOSI, `PB4` MISO), LSE crystal (`PC14`/`PC15`)
- If CAN is added later, use `PB8`/`PB9` (CAN1 alternate-function pins) to avoid clashing with USB on `PA11`/`PA12`

### CAN pin assignment + transceiver IC
- STM32's bxCAN (`CAN1`) only has three possible pin pairs, selected by an AF remap (confirmed in `speeduino/speeduino/src/STM32_CAN/STM32_CAN.cpp:53-112`): `DEF` = PA11/PA12, `ALT` = PB8/PB9, `ALT_2` = PD0/PD1.
- `DEF` (PA11/PA12) is out — already used for USB. `ALT_2` (PD0/PD1) is out — **Port D doesn't physically exist on the Blackpill's 48-pin F401/F411 package**. That leaves **`ALT`: `PB8` (CAN1_RX) / `PB9` (CAN1_TX)** as the only option, and it's free in our pin budget.
- **Firmware bug found:** `speeduino/speeduino/board_stm32_official.cpp:109` hardcodes `STM32_CAN Can0 (CAN1, ALT_2, RX_SIZE_256, TX_SIZE_16);` — written for the official STM32F407 board (100-pin, has Port D). This will not work on a Blackpill and needs to be changed to `STM32_CAN Can0 (CAN1, ALT, RX_SIZE_256, TX_SIZE_16);` for our board.
- **Transceiver IC recommendation: `SN65HVD230`** (TI), not the `MCP2551` pazi88's board used. `MCP2551` is a 5V-only part with a TXD input threshold (~0.6×VDD ≈ 3V at 5V supply) that's marginal against a 3.3V STM32 output — the same 5V/3.3V interface issue we've routed around elsewhere. `SN65HVD230` runs natively on 3.3V with 3.3V-logic TX/RX, so no level-shifting is needed at all.
- Placement: transceiver chip → **MCU board** (generic comms IC, same as Bluetooth). CANH/CANL connector + TVS/ESD clamp on the bus lines → **car board** (harness-facing protection, same pattern as the sensor inputs).

### Bluetooth (HC-05) pin assignment — decided, then dropped
- **`PA9` → HC-05 RXD, `PA10` ← HC-05 TXD** (USART1) was the original plan. Free of all the allocations above, FT-rated (5V-tolerant) for headroom against a 5V-powered HC-05 module's TX level, and confirmed against the actual firmware: `speeduino/speeduino/board_stm32_official.h:126-127` hardcodes `HardwareSerial Serial1(PA10, PA9);` for Blackpill-class targets (`SERIAL_UART_INSTANCE==2`) — Speeduino's dedicated wireless serial port, matching stock firmware with no source changes.
- **Later dropped** (§7) to reclaim `PA9`/`PA10` for the digital I/O pin crunch — tuning is USB-only now. Trade-off: no wireless road-tuning/datalogging, and no easy way to add it back later since Speeduino's USB is device-mode only (can't host a USB-Bluetooth/WiFi dongle plugged into the car-board USB port). Reclaiming wireless in the future means finding 2 more pins from scratch.
- The same file's `pinIsReserved()` (line 134-140, for `ARDUINO_BLACKPILL_F401CC`/`F411CE`) independently confirms `PA11`, `PA12`, `PC14`, `PC15` as reserved — matches the schematic findings above, and remains relevant regardless of the Bluetooth decision.
- General takeaway (still applies): STM32 pin-to-peripheral assignment is a **compile-time** decision (which physical pins back a given `Serial`/`SPI`/etc. object), not a runtime or TunerStudio-configurable setting — changing it later means editing the board file and recompiling, not just rewiring.

## 6. Two-PCB architecture: MCU board + car-specific board

Decision: split into two boards.
- **MCU board**: Blackpill + anything generic/reusable across every car project.
- **Car-specific board**: everything that depends on this particular engine/harness, plus all the heavy power switching ("big components: MOSFET, power, special signal conditioning").

Guiding principle: the car board's job is to make every signal look "standard" by the time it crosses the connector to the MCU board (either it's already a standard signal type, or the car board conditions a car-specific one into a standard shape) and to own all high-current/power switching. The MCU board should never need a redesign when moving to a new car.

### MCU board (generic, reusable) — final decision
- Blackpill (STM32F4) module
- Local 3.3V LDO fed from a clean 5V input (not raw 12V) — keeps MCU power quiet, separate from the noisier driver-board rail
- USB (onboard, dev-only) + SWD debug headers
- TVS/ESD clamp arrays on every analog and digital input line (second line of defense — see dual-layer protection note below), plus the standard analog front-end (resistor divider + buffer/op-amp per channel, 5V→3.3V) for MAP/TPS/CLT/IAT/O2/battery-ref/flex
- All level-shifting lives here: digital output level-shift buffers (3.3V→5V) for standard triggers (injector/ignition/idle/etc.), and any digital-input-side translation needed
- Generic VR/Hall trigger conditioner + pull-up jumpers for crank/cam
- CAN transceiver (`SN65HVD230`) — see §5
- SD/SPI-flash logging, RTC crystal + backup battery. ~~Bluetooth/serial header~~ — dropped (§7), tuning is USB-only

Rationale for centralizing level-shifting here: this design gets solved and verified once, on the board that's reused across every car project — a new car board never needs to re-derive divider/buffer values.

### Car-specific board (owns power + harness interface) — final decision
- 12V input protection: reverse-polarity, fuse, TVS/load-dump clamp
- Main 5V regulation, sized to this car's actual driver current draw — **5V generation lives only here**, not duplicated on the MCU board (see power-source decision below)
- All MOSFET/driver stages: injector drivers, ignition drivers (TC4424-style), idle stepper (DRV8825-style), fan/fuel-pump/tacho relay drivers, with flyback diodes across every inductive load
- Harness connectors matching this car's actual wiring/plugs
- TVS/clamp diode array at every harness-facing pin (first line of defense, closest to the fault source)
- Any non-standard sensor conditioning unique to this car (odd-range sensor, factory knock amp, trans signals, boost solenoid feedback, etc.) — conditioned down into a standard 0–5V or digital shape before crossing to the MCU board
- **Custom USB connector** (data-only, see below)

### Dual-layer protection decision
Protection is deliberately duplicated on both boards, not centralized on one: the car board clamps at the harness connector (closest to the actual fault source — inductive kickback, load dump, ESD from cable handling), and the MCU board clamps again at its own input connector (protects the MCU even if a given car board's protection is imperfect, and protects against transients picked up crossing the board-to-board connector itself). Two independent clamp stages is strictly safer than one; the only cost is the extra parts on the car board, which is worth it given the car board changes per project and is more likely to see a layout mistake than the once-designed MCU board.

### Board-to-board connector contract
- Power: 5V (car→MCU board), GND
- Standard analog sensor lines: raw 0–5V, passed through after the car board's own TVS clamp; MCU board does the scaling and clamps again
- Crank/cam: raw VR/Hall signal, conditioned on the MCU board
- Digital trigger outputs: already at 5V logic (from the MCU board's level-shift buffers) going to the car board's driver inputs
- Pre-conditioned "special" signals: car board turns its non-standard sensors into standard-looking signals before handing them across

### Power-source decision: 5V generation only on the car board
- Confirmed: without the car board attached, the MCU board can be powered and tested standalone via the Blackpill's onboard USB — no need to duplicate 5V generation on the MCU board.
- This means the MCU board has two possible 5V sources depending on mode (USB VBUS when bench-testing, car board's 5V rail when installed) but never both at once in the intended workflow.
- Still worth adding a series Schottky diode (e.g. BAT54/SS14) from the car board's 5V rail into the point where it feeds the Blackpill, to protect the car board's rail if USB and car-board power were ever both present — but see the VBUS finding above: this cannot fully protect a connected USB host in that simultaneous case, because the module's own VBUS-to-5V-pin trace is undiodeed. The real mitigation is procedural: don't connect USB-to-host and car-board power at the same time. This matches the intended workflow already (see USB routing decision below), so it's a non-issue in practice.

### USB routing decision
- The onboard USB-C connector on the Blackpill is **dev-only** (bench flashing/testing, board powered via USB, car board unplugged).
- A **separate, custom USB connector lives on the car board**, wired to the same MCU USB peripheral via the broken-out `PA11`/`PA12` header pins (not through the Blackpill's onboard USB-C circuitry). This is used for USB connectivity while the unit is powered via the car board, and sidesteps the VBUS/backfeed problem entirely — as long as:
  - The custom connector's **VBUS pin is left unconnected** (data + GND only) — it must never be tied to the board's 5V rail, or the same backfeed problem reappears one connector removed.
  - **ESD protection** (e.g. `USBLC6-2SC6`) is added at the custom connector, since it's now a user-facing, cable-inserted port on the car board — consistent with the car board's general role of harness/connector-facing protection.
  - **D+/D- are routed as a matched differential pair** (length-matched, short, ~90Ω differential) across the board-to-board connector and PCB traces.
  - **Only one USB connector is used at a time** — onboard dev port and the car-board port share the same physical USB peripheral pins; using both simultaneously would cause bus contention. The intended workflow (dev via onboard USB with car board unplugged, or connected via car board with onboard USB unused) already satisfies this.
- **MCU board also gets its own USB ESD protection**, not just the car board's connector: add a second `USBLC6-2SC6`-class array on the MCU board at the `PA11`/`PA12` breakout, ahead of where the board-to-board connector feeds them out to the car board's custom USB connector. Same dual-layer reasoning as the sensor/digital inputs (§ Dual-layer protection decision) — the car board's ESD array is the first line of defense at the actual harness-facing connector, the MCU board's is the second, protecting the MCU peripheral even if the car board's protection is imperfect or bypassed.

## 7. Digital I/O pin budget: `beat-mcu.md` (Honda Beat OEM ECU pinout) mapped to the MCU board

Decision to replicate the Honda Beat's factory ECU connector pinout (`pref/beat-mcu.md`, `pref/beat-ecu.jpg`) rather than design a new harness — the custom ECU plugs into the existing car harness. Need: 7 analog inputs, 18 digital I/O (D1–D18).

### Speeduino has no I²C GPIO expander support
Grepped the entire `speeduino/speeduino` tree for `MCP23017`/`PCF8574`/`expander` — no matches anywhere. Confirmed by reading the code directly: even Speeduino's generic Programmable I/O system (`src/controllers/progammableIO/programmableIOControl.cpp:39`) calls `digitalWrite(channel.outputPin, ...)` on a plain native pin number. **There is no hook anywhere in the firmware for an I²C-attached pin** — every input/output Speeduino touches must be a real MCU GPIO known to the Arduino core. Using an I²C expander for anything Speeduino itself controls would require writing custom firmware to bridge it, not just wiring it up.

### 3 of the originally-planned "expander" signals are actually native Speeduino features
Checked `speeduino/speeduino/src/pins/pinNumbers_t.h` for the full list of named pins Speeduino understands. Of the 8 signals initially assumed to be simple/slow enough for an expander, 3 have dedicated first-class control logic and must be native MCU pins regardless:
- D5 (fuel pump relay) → `pinFuelPump`
- D9 (A/C compressor/clutch relay) → `pinAirConComp`
- D12 (A/C switch) → `pinAirConRequest`

(Control logic lives in `idle.cpp` and `src/controllers/aircon/airconController.cpp`, both calling `digitalWrite`/`digitalRead` directly on the configured native pin.)

### The remaining 5 signals have no Speeduino equivalent at all
D4 (brake switch), D7 (O2 heater control), D8 (check engine light), D11 (starter signal), D14 (service check) — none appear anywhere in `pinNumbers_t.h`. These are OEM diagnostic/monitoring signals the aftermarket ECU firmware was never going to act on. **Recommendation: don't wire them to the MCU at all** — handle as standalone car-board circuits instead:
- O2 heater: hardwire to switched (ignition-on) 12V directly, no ECU timing needed
- Check engine light: **now assigned** — see below, freed up by dropping Bluetooth
- Brake switch / starter signal / service check: likely unnecessary for an aftermarket build; `pinLaunch` exists if brake switch is ever wanted as a launch-control trigger, a future consideration only

### Net result: no I²C expander needed
The 3 reclassified signals fit exactly into the 3 MCU pins originally set aside for an I²C bus (`B10`, `B3`, `C13` — see `mcu-board.md`). All digital signals Speeduino actually understands now fit natively — full pin table and rationale in `mcu-board.md`.

### Pin pressure reassessed — Bluetooth dropped instead of spinning a custom MCU (for now)
With all 13 free header pins on the WeAct Blackpill module committed and zero margin left for anything unplanned (the still-undecided check-engine light, or any signal discovered later while going through the rest of `beat-mcu.md`'s B21/B22 sensor-excitation rows), the next move considered was designing a custom STM32 circuit directly (raw chip on the PCB, bigger package like the `STM32F407VET6` 100-pin LQFP pazi88's reference board already uses) instead of the pre-made Blackpill module — trading the Blackpill's convenience (proven layout, onboard USB/flash/crystal, cheap to source) for full pin availability.

**Decided against, for now**: dropping the Bluetooth breakout (`PA9`/`PA10`, see §5) freed exactly 2 pins, which covers the check-engine-light gap with 1 pin (`PA10`) still spare — enough margin to keep the Blackpill module viable without a custom MCU redesign. The custom-MCU option remains on the table if a future need outstrips this margin again (e.g. wanting wireless tuning back, or more signals surfacing from `beat-mcu.md`'s B21/B22 rows) — pazi88's schematic remains the proven template to follow if that becomes necessary.

### Full audit against `beat-mcu.md` — confirmed, digital I/O fits with exactly 1 pin spare
Re-verified every D1–D18 signal (post D15-gap fix) against `mcu-board.md`'s pin table line by line, checking for duplicate pin assignments and EXTI line conflicts.

| D# | Signal | Pin |
|---|---|---|
| D1–D3 | Injectors #1–3 | `B12` / `B13` / `B14` |
| D4 | Brake switch | not wired (no Speeduino equivalent) |
| D5 | Fuel pump relay | `C13` (`pinFuelPump`) |
| D6 | EACV solenoid | `A8` |
| D7 | O2 heater | not wired (no Speeduino equivalent) |
| D8 | Check engine light | `A9` |
| D9 | A/C compressor | `B10` (`pinAirConComp`) |
| D10 | Igniter drive | `B15` |
| D11 | Starter signal | not wired (no Speeduino equivalent) |
| D12 | A/C switch | `B3` (`pinAirConRequest`) |
| D13 | VSS | `B2` (EXTI2) |
| D14 | Service check | not wired (no Speeduino equivalent) |
| D15 | Cylinder ID/cam | `B6` (EXTI6) |
| D16 | TDC | `B5` (EXTI5) |
| D17 | Crank angle | `B7` (EXTI7) |
| D18 | Alternator FR | `A15` (EXTI15) |

14 wired + 4 intentionally unwired = 18, matches `beat-mcu.md` exactly. No duplicate pin assignments found. The 5 EXTI-interrupt inputs (D13/D15/D16/D17/D18) sit on distinct pin-numbers (2/6/5/7/15) — no line-sharing conflicts.

Pin budget re-derived independently and matches `mcu-board.md`'s own claim: 40 header positions → 8 fixed (power/reset/VBAT) → 32 GPIO-capable → −2 (LSE crystal) → 30 → −2 (USB) → −4 (SPI flash) → −2 (CAN) → −7 (analog) → −14 (digital) = **1 pin spare (`PA10`)**.

**Analog count reconciled**: `beat-mcu.md`'s own "A" counter only lists 6 signals (`A1`=battery, `A2`=TPS, `A3`=MAP, `A4`=IAT, `A5`=CLT, `A6`=O2), but the MCU board provisions 7 "Analog #" slots — **confirmed intentional, one genuine spare analog channel**, not an uncounted 7th sensor.

**Mystery signal pins ignored, deliberately**: `beat-mcu.md`'s `B12` ("atmospheric pressure sensor test?") and `B16` ("test only?") have no pin counter assigned and are being left unconnected — a confirmed decision, not an oversight.

## Open items / things to verify later
- Confirm exact `SN74AHCT541` electrical specs against its datasheet directly — not yet independently re-verified, still general datasheet knowledge. ~~SP721, TC4423A/TC4424A, and ULN2003A~~ — resolved, all three datasheets added and verified.
- Confirm which specific pins on the chosen Blackpill module (F401/F411/F407 variant) are actually broken out and FT-rated before finalizing crank/cam/clutch/flex pin assignments.
- Confirm the exact STM32F4 variant on the WeAct board (`STM32F4X1CxU6` is generic in the schematic — could be F401 or F411) and its onboard LDO dropout, to size the Schottky diode drop from the car board's 5V rail with enough headroom.
- Decide whether to add a physical interlock (e.g. a switch or clearly separated connectors) to enforce "USB-to-host and car-board power never both connected," or just document it as an operating rule.
- Patch `speeduino/speeduino/board_stm32_official.cpp:109` from `STM32_CAN Can0 (CAN1, ALT_2, ...)` to `ALT` before CAN will work on our Blackpill-based board (Port D pins used by `ALT_2` don't exist on the F401/F411 48-pin package).
- Confirm `DRV8825` logic-input spec and choose/verify the specific injector/spare-output MOSFET's V_GS threshold — both still general knowledge, not datasheet-verified locally.
- Add datasheets and verify exact specs for the two newly-chosen battery-voltage-divider protection parts: `PRTR5V0U1T` (tap-point clamp) and the ~24–26V-standoff automotive TVS for the raw `12V-SW` line (specific part not yet picked, e.g. `SM6T26A`/`SMAJ26A` class) — both currently general knowledge only.
- Add a datasheet and verify `MCP6002`'s exact specs (input bias current, GBW, slew rate, abs max ratings) — currently general part-family knowledge, not locally confirmed.
- Confirm the exact STM32F4 ADC pin input-leakage-current spec (from the real datasheet electrical characteristics table) to precisely quantify the small systematic offset R5 introduces on the 6 buffered analog channels — currently an order-of-magnitude estimate, not a confirmed number.
