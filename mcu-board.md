# MCU board


## pin mapping

| usage | left pins | right pins | usage |
|---|---|---|---|
| Injector #1 output (D1, ISR-timed) | B12 | 5V | Power input (5V rail) |
| Injector #2 output (D2, ISR-timed) | B13 | GND | Ground |
| Injector #3 output (D3, ISR-timed) | B14 | 3.3V | Power output (3.3V rail) |
| Igniter drive output (D10, ISR-timed) | B15 | B10 | A/C compressor output (D9, `pinAirConComp`) |
| EACV solenoid PWM output (D6, idle) | A8 | B2 | VSS input (D13, EXTI2) |
| Check engine light output (D8, via Programmable Output) | A9 | B1 | Analog #6 |
| Free (Bluetooth breakout dropped — tune via USB only) | A10 | B0 | Analog #5 |
| USB D- (onboard dev + car-board custom port, shared) | A11 | A7 | Onboard flash SPI1 MOSI (reserved) |
| USB D+ (onboard dev + car-board custom port, shared) | A12 | A6 | Analog #7(battery voltage) |
| Alternator FR signal input (D18, duty-cycle, EXTI15) | A15 | A5 | Onboard flash SPI1 SCK (reserved) |
| A/C request input (D12, `pinAirConRequest`) | B3 | A4 | Onboard flash SPI1 CS (reserved) |
| Onboard flash SPI1 MISO (reserved) | B4 | A3 | Analog #4 |
| TDC sensor input (D16, EXTI5) | B5 | A2 | Analog #3 |
| Cylinder ID / cam sensor input (D15, EXTI6) | B6 | A1 | Analog #2 |
| Crank angle sensor input (D17, EXTI7) | B7 | A0 | Analog #1 |
| CAN1_RX (ALT mapping) | B8 | NRST | Reset input |
| CAN1_TX (ALT mapping) | B9 | C15 | Reserved — LSE 32.768kHz crystal |
| Power input (5V rail, same net) | 5V | C14 | Reserved — LSE 32.768kHz crystal |
| Ground | GND | C13 | Fuel pump output (D5, `pinFuelPump`) — onboard blue LED also on this pin, harmless for a driven output |
| Power output (3.3V rail, same net) | 3.3V | VB | VBAT — RTC backup battery input |

Note: `PA13`/`PA14` (SWD) aren't on either header — internal-only, wired to the onboard debug connector.

## Bluetooth breakout dropped
Removed the HC-05 header (`PA9`/`PA10`, previously USART1) — tuning is done via USB only now (onboard USB for bench dev, custom car-board USB port while installed). This trades away wireless road-tuning/datalogging with no easy path to add it back later (Speeduino's USB is device-mode only, not host-mode, so a USB-Bluetooth/WiFi dongle can't be plugged into the car-board USB port to regain it — reclaiming wireless later would mean finding 2 more pins from scratch). Freed pins used below.

14 of 15 free MCU pins are now assigned (`PA9`/`PA10` became free once Bluetooth was dropped, on top of the original 13); `PA10` remains the last spare GPIO.

## Digital I/O split rationale (from `pref/beat-mcu.md`)

18 digital signals (D1–D18) needed vs. only 13 free MCU pins after USB/CAN/SPI-flash/analog/power (before the Bluetooth removal above freed 2 more). Originally planned a car-board I²C GPIO expander (e.g. `MCP23017`) for the 8 slow/non-timing-critical signals — **reconsidered and dropped** after checking the actual firmware (see `research.md` for the full writeup):

- **Speeduino has no I²C GPIO expander support anywhere in its codebase.** Every pin it touches, including its generic Programmable I/O system, goes through a plain `digitalWrite`/`digitalRead` on a native MCU pin — an expander would need custom firmware to bridge it, not just wiring.
- Of the original 8 "slow" signals, **3 turned out to be first-class Speeduino features requiring a native pin regardless of expander plans**: D5 (fuel pump → `pinFuelPump`), D9 (A/C compressor → `pinAirConComp`), D12 (A/C request → `pinAirConRequest`). These are now assigned directly (table above), using the 3 pins originally set aside for the I²C bus.
- The remaining 4 (D4 brake switch, D7 O2 heater, D11 starter signal, D14 service check) have **no equivalent anywhere in Speeduino's firmware** — they're OEM diagnostic/monitoring signals the ECU was never going to act on. Recommendation: don't wire them to the MCU at all. Handle as standalone car-board circuits instead (e.g. O2 heater hardwired to switched 12V; brake/starter/service-check likely unnecessary for an aftermarket build).
- **D8 (check engine light) is now assigned** (table above, `PA9`) using one of the two pins freed by dropping Bluetooth — driven via Speeduino's Programmable Output rule system, which needs a native pin but no core firmware change.

**Result: no expander needed.** All 14 digital signals Speeduino actually cares about (10 originally-direct + 3 reclassified native features + check engine light) fit natively in the pins freed up after dropping Bluetooth, with 1 pin (`PA10`) still spare.

**Direct (14 signals total)** — ISR-timed outputs (injectors, igniter, EACV PWM), EXTI-interrupt inputs (crank, cam/cylinder-ID, TDC, VSS, FR — FR confirmed duty-cycle-based), the 3 native Speeduino features (fuel pump, A/C compressor, A/C request), and check engine light, all listed in the table above. The interrupt inputs were deliberately assigned to distinct pin-numbers (2, 5, 6, 7, 15) since STM32 EXTI lines are shared by pin-number across ports — two inputs can't share a pin-number's EXTI line simultaneously.
