# MCU board


## pin mapping

| usage | left pins | right pins | usage |
|---|---|---|---|
| Free | B12 | 5V | Power input (5V rail) |
| Free | B13 | GND | Ground |
| Free | B14 | 3.3V | Power output (3.3V rail) |
| Free | B15 | B10 | Free |
| Free | A8 | B2 | Free |
| Bluetooth TX1 → HC-05 RXD | A9 | B1 | Analog #6 |
| Bluetooth RX1 ← HC-05 TXD | A10 | B0 | Analog #5 |
| USB D- (onboard dev + car-board custom port, shared) | A11 | A7 | Onboard flash SPI1 MOSI (reserved) |
| USB D+ (onboard dev + car-board custom port, shared) | A12 | A6 | Analog #7(battery voltage) |
| Free | A15 | A5 | Onboard flash SPI1 SCK (reserved) |
| Free | B3 | A4 | Onboard flash SPI1 CS (reserved) |
| Onboard flash SPI1 MISO (reserved) | B4 | A3 | Analog #4 |
| Free | B5 | A2 | Analog #3 |
| Free | B6 | A1 | Analog #2 |
| Free | B7 | A0 | Analog #1 |
| CAN1_RX (ALT mapping) | B8 | NRST | Reset input |
| CAN1_TX (ALT mapping) | B9 | C15 | Reserved — LSE 32.768kHz crystal |
| Power input (5V rail, same net) | 5V | C14 | Reserved — LSE 32.768kHz crystal |
| Ground | GND | C13 | Free (onboard blue LED also on this pin) |
| Power output (3.3V rail, same net) | 3.3V | VB | VBAT — RTC backup battery input |

Note: `PA13`/`PA14` (SWD) aren't on either header — internal-only, wired to the onboard debug connector.
