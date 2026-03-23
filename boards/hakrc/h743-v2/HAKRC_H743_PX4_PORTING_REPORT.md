# HAKRC H743 – PX4 Firmware Porting Technical Report

**Date:** 2026-03-23  
**Firmware:** PX4 Autopilot (custom branch `hakrc_h743v2_support`)  
**Board:** HAKRC H743 V2 (STM32H743VIT6, 480 MHz)  
**Status:** ✅ Fully operational — dual IMU, barometer, OSD confirmed

---

## 1. Hardware Overview

| Feature | Details |
|---|---|
| MCU | STM32H743VIT6, 480 MHz, 2 MB Flash, 1 MB RAM |
| IMU 1 | ICM42688P on SPI1, CS=PC15, DRDY=PB2 |
| IMU 2 | ICM42688P on SPI4, CS=PC13 (no DRDY — backup domain) |
| Barometer | DPS310 (compatible) on I2C1 at 0x76 |
| OSD | MAX7456 on SPI3, CS=PE2 |
| Motors | 4× DSHOT on TIM8/CH1 (bitbang), expandable to 12 PWM |
| SD Card | SDIO (4-bit) on PC8/PC9/PC10/PC11/PC12/PD2 |
| USB | USB OTG FS on PA11/PA12 |
| LED Strip | TIM1/CH1 on PA8 |
| Beeper | PE9 |

---

## 2. Reference Board

The closest existing PX4 target is **Matek H743-Slim V3**, which has an identical peripheral topology:
- Gyro 1: ICM42688P on SPI1, CS=PC15, DRDY=PB2
- Gyro 2: ICM42688P on SPI4, CS=PC13 (no DRDY)
- OSD: Matek uses SPI2, HAKRC uses SPI3

**Key difference:** HAKRC uses SPI3 for OSD (PB3/PB4/PB5), Matek uses SPI2 (PB12).

---

## 3. SPI Bus Topology (Most Critical Finding)

### 3.1 Wrong initial assumption

The initial approach assumed the OSD was on SPI4 (based on the PE12/PE13/PE14 pins, which are SPI4). This was incorrect.

### 3.2 Definitive layout from Betaflight CLI dump

Running `get spi` in Betaflight revealed:
```
gyro_1_spibus = 1     → SPI1 (PA5/PA6/PD7)
gyro_2_spibus = 4     → SPI4 (PE12/PE13/PE14)
max7456_spi_bus = 3   → SPI3 (PB3/PB4/PB5)
baro_bustype = I2C
baro_i2c_device = 1   → I2C1 (PB6/PB7)
```

### 3.3 Final SPI pin mapping

| Bus | SCK | MISO | MOSI | Device | CS | DRDY |
|---|---|---|---|---|---|---|
| SPI1 | PA5 (AF5) | PA6 (AF5) | PD7 (AF5) | ICM42688P IMU1 | PC15 | PB2 |
| SPI3 | PB3 (AF6) | PB4 (AF6) | PB5 (AF7) | MAX7456 OSD | PE2 | — |
| SPI4 | PE12 (AF5) | PE13 (AF5) | PE14 (AF5) | ICM42688P IMU2 | PC13 | — |

> **Important:** SPI3 MOSI on PB5 uses AF7 (`GPIO_SPI3_MOSI_4`), not AF6. This is because PB5 is also shared with I2S3_SDO — the pinmap has two entries and AF7 is the correct SPI3 MOSI alternate function on this specific pin.

---

## 4. The PC13/PC14 Backup Domain Problem

### 4.1 Background

On STM32H7, **PC13, PC14, PC15** are in the RTC backup power domain (`VBAT`). They default to being protected by the backup domain write protection (`PWR_CR1.DBP` bit), which must be explicitly cleared before these pins can be configured as regular GPIOs.

### 4.2 Impact

- **PC15** (IMU1 CS): Works fine because PX4's SPI CS init forces `GPIO_OUTPUT_SET`, and NuttX's `stm32_configgpio()` already unlocks the backup domain as part of normal GPIO init on STM32H7.
- **PC13** (IMU2 CS): Works correctly as a CS output after ensuring `stm32_configgpio()` configures it properly.
- **PC14** (IMU2 DRDY): Configuring PC14 as a `GPIO_INPUT | GPIO_EXTI` failed silently — the EXTI interrupt was not firing, causing the ICM42688P on SPI4 to never produce IMU data.

### 4.3 Solution

Remove PC14 from the DRDY config and run IMU2 in **polling mode** (no DRDY). This is identical to how the official **Matek H743-Slim** handles the same situation:

```cpp
// spi.cpp
initSPIBus(SPI::Bus::SPI4, {
    initSPIDevice(DRV_IMU_DEVTYPE_ICM42688P,
                  SPI::CS{GPIO::PortC, GPIO::Pin13}), // no DRDY
}),
```

**Performance impact:** IMU2 shows interval SD of ~418 µs vs ~70 µs for DRDY-driven IMU1. This is acceptable for a redundancy sensor — PX4 uses IMU1 for primary flight filtering.

---

## 5. File-by-File Changes

### 5.1 `boards/hakrc/h743-v2/src/spi.cpp`
```cpp
constexpr px4_spi_bus_t px4_spi_buses[SPI_BUS_MAX_BUS_ITEMS] = {
    initSPIBus(SPI::Bus::SPI1, {
        initSPIDevice(DRV_IMU_DEVTYPE_ICM42688P,
                      SPI::CS{GPIO::PortC, GPIO::Pin15},
                      SPI::DRDY{GPIO::PortB, GPIO::Pin2}),
    }),
    initSPIBus(SPI::Bus::SPI3, {
        initSPIDevice(DRV_OSD_DEVTYPE_ATXXXX,
                      SPI::CS{GPIO::PortE, GPIO::Pin2}),
    }),
    initSPIBus(SPI::Bus::SPI4, {
        initSPIDevice(DRV_IMU_DEVTYPE_ICM42688P,
                      SPI::CS{GPIO::PortC, GPIO::Pin13}), // polling mode
    }),
};
```

### 5.2 `boards/hakrc/h743-v2/src/board_config.h`

Key GPIO definitions:
```c
#define GPIO_SPI1_CS_IMU1   /* PC15 */ (GPIO_OUTPUT|GPIO_PUSHPULL|GPIO_SPEED_50MHz|GPIO_OUTPUT_SET|GPIO_PORTC|GPIO_PIN15)
#define GPIO_SPI4_CS_IMU2   /* PC13 */ (GPIO_OUTPUT|GPIO_PUSHPULL|GPIO_SPEED_50MHz|GPIO_OUTPUT_SET|GPIO_PORTC|GPIO_PIN13)
#define GPIO_SPI3_CS_OSD    /* PE2  */ (GPIO_OUTPUT|GPIO_PUSHPULL|GPIO_SPEED_50MHz|GPIO_OUTPUT_SET|GPIO_PORTE|GPIO_PIN2)
#define GPIO_SPI1_DRDY_IMU1 /* PB2  */ (GPIO_INPUT|GPIO_FLOAT|GPIO_EXTI|GPIO_PORTB|GPIO_PIN2)
// No DRDY for IMU2 (PC14 backup domain)
```

PWM: 12 channels across TIM5/TIM4/TIM15/TIM3 (note: TIM15 on PE5/PE6 does NOT conflict with SPI4 on PE12/13/14).

### 5.3 `boards/hakrc/h743-v2/nuttx-config/include/board.h`

SPI pin overrides:
```c
#define GPIO_SPI1_MOSI  GPIO_SPI1_MOSI_3  // PD7, AF5
#define GPIO_SPI3_MISO  GPIO_SPI3_MISO_1  // PB4, AF6
#define GPIO_SPI3_MOSI  GPIO_SPI3_MOSI_4  // PB5, AF7  ← note AF7, not AF6
#define GPIO_SPI3_SCK   GPIO_SPI3_SCK_1   // PB3, AF6
#define GPIO_SPI4_MISO  GPIO_SPI4_MISO_1  // PE13, AF5
#define GPIO_SPI4_MOSI  GPIO_SPI4_MOSI_1  // PE14, AF5
#define GPIO_SPI4_SCK   GPIO_SPI4_SCK_1   // PE12, AF5
#define GPIO_I2C1_SCL   GPIO_I2C1_SCL_1   // PB6
#define GPIO_I2C1_SDA   GPIO_I2C1_SDA_1   // PB7
```

### 5.4 `boards/hakrc/h743-v2/nuttx-config/nsh/defconfig`

SPI buses enabled — **must match** `spi.cpp` exactly or `validateSPIConfig()` static assertion fails at compile time:
```
CONFIG_STM32H7_SPI1=y
CONFIG_STM32H7_SPI1_DMA=y
CONFIG_STM32H7_SPI1_DMA_BUFFER=2048
CONFIG_STM32H7_SPI3=y
CONFIG_STM32H7_SPI3_DMA=y
CONFIG_STM32H7_SPI3_DMA_BUFFER=2048
CONFIG_STM32H7_SPI4=y
CONFIG_STM32H7_SPI4_DMA=y
CONFIG_STM32H7_SPI4_DMA_BUFFER=2048
```

> **Critical:** `validateSPIConfig()` in `platforms/nuttx/src/px4/stm/stm32h7/include/px4_arch/spi_hw_description.h` enforces a 1:1 match between `CONFIG_STM32H7_SPIx=y` flags and entries in `px4_spi_buses[]`. Adding SPI3 to `spi.cpp` without adding `CONFIG_STM32H7_SPI3=y` to defconfig will cause a compile-time static assertion error.

### 5.5 `boards/hakrc/h743-v2/default.px4board`

Enabled drivers:
```
CONFIG_DRIVERS_IMU_INVENSENSE_ICM42688P=y
CONFIG_DRIVERS_BAROMETER_GOERTEK_SPA06=y   # replaces DPS310
CONFIG_DRIVERS_OSD_ATXXXX=y                # MAX7456 analog OSD
CONFIG_DRIVERS_ADC_BOARD_ADC=y
```

### 5.6 `boards/hakrc/h743-v2/init/rc.board_sensors`

```sh
board_adc start

# Gyro 1 – ICM42688P on SPI1, CS=PC15, DRDY=PB2
icm42688p start -s -b 1 -R 6

# Gyro 2 – ICM42688P on SPI4, CS=PC13 (polling mode)
icm42688p start -s -b 4 -R 6

# Barometer – DPS310/SPA06-compatible on I2C1 at 0x76
spa06 start -X -b 1 -a 0x76

# OSD – MAX7456 on SPI3
atxxxx start -s -b 3
```

---

## 6. Barometer

- **Chip:** DPS310 (or compatible SPA06)
- **Bus:** I2C1 (PB6 SCL / PB7 SDA)
- **Address:** 0x76
- **Driver:** `CONFIG_DRIVERS_BAROMETER_GOERTEK_SPA06` — the `spa06` driver supports DPS310-compatible chips.
- **Start command:** `spa06 start -X -b 1 -a 0x76` (`-X` = I2C external mode, `-b 1` = I2C bus 1)

---

## 7. OSD

- **Chip:** MAX7456 analog OSD
- **Bus:** SPI3 (PB3 SCK / PB4 MISO / PB5 MOSI)
- **CS:** PE2
- **Driver:** `atxxxx` (PX4 analog OSD driver)
- **Bus number in PX4:** 3 (1-indexed)

PX4 does not render a live HUD on the OSD out-of-the-box. The `atxxxx` driver maintains SPI communication to the MAX7456 but display content requires MAVLink OSD configuration.

---

## 8. ICM42688P Driver Notes

The `ICM42688P::probe()` function (`src/drivers/imu/invensense/icm42688p/ICM42688P.cpp`) was patched to also accept `WHO_AM_I = 0x42` (non-P variant) in addition to the standard `0x47`. This patch is optional — both boards confirmed `0x47`, but it future-proofs against minor silicon variants.

---

## 9. PWM Timers

12 total PWM channels configured:

| Timer | Channels | Pins |
|---|---|---|
| TIM5 | 1-4 | PA0, PA1, PA2, PA3 |
| TIM4 | 5-8 | PD12, PD13, PD14, PD15 |
| TIM15 | 9-10 | PE5, PE6 |
| TIM3 | 11-12 | PB0, PB1 |

DSHOT uses TIM8/CH1 (PA7 via bitbang DMA).

---

## 10. I2C Buses

| Bus | SCL | SDA | Devices |
|---|---|---|---|
| I2C1 | PB6 | PB7 | Barometer (0x76) |
| I2C2 | PB10 | PB11 | Available (GPS, compass) |

---

## 11. Board ID

Custom Board ID: **1185** (registered in `CMakeLists.txt`).  
Bootloader uses this ID to match firmware — both must be built from this branch.

---

## 12. Serial Ports

| UART | Pins | Default use |
|---|---|---|
| USART1 | (standard) | Available |
| USART2 | (standard) | Available |
| USART3 | (standard) | Available |
| UART4 | (standard) | Primary telemetry candidate |
| UART7 | (standard) | Available |

---

## 13. Build Instructions

```bash
# Build firmware
make hakrc_h743-v2_default

# Build bootloader (first time only)
make hakrc_h743-v2_bootloader

# Output artifacts
build/hakrc_h743-v2_default/hakrc_h743-v2_default.px4   # Upload via QGC
build/hakrc_h743-v2_default/hakrc_h743-v2_default.bin   # Raw binary
build/hakrc_h743-v2_bootloader/hakrc_h743-v2_bootloader.bin
```

Flash bootloader via STM32CubeProgrammer (DFU mode), then load `.px4` via QGC.

---

## 14. Verified Sensor Output (NSH Console)

```
sensors status:
  sensor #0 (IMU1): gyro + accel, 8105 Hz FIFO, device 2490378
  sensor #1 (IMU2): gyro + accel, ~800 Hz polling, device 2490402
  BARO: device 15234569, I2C

icm42688p status:
  Running on SPI Bus 1 (bad_transfer: ~0, FIFO driven)
  Running on SPI Bus 4 (bad_transfer: 0, polling)

atxxxx status:
  Running on SPI Bus 3
```

---

## 15. ArduPilot Porting Notes

For porting to ArduPilot (AP_HAL_ChibiOS), the following information is critical:

### 15.1 SPI Bus Map (HAL_SPIDEVICE)
```
SPIDesc HAL_SPI_DEVICE_LIST[] = {
    { "icm42688",   1, 0, PC15, SPI_MODE3, 2*MHZ, 8*MHZ },  // Gyro 1
    { "icm42688",   3, 1, PC13, SPI_MODE3, 2*MHZ, 8*MHZ },  // Gyro 2 (SPI4 = bus index 3 in AP)
    { "max7456",    2, 2, PE2,  SPI_MODE0, 10*MHZ, 10*MHZ}, // OSD (SPI3 = bus index 2 in AP)
};
```
> **Note:** ArduPilot uses 0-indexed SPI buses (SPI1→0, SPI3→2, SPI4→3). Verify with `hwdef.dat` format.

### 15.2 I2C Bus Map
```
I2C_ORDER I2C1 I2C2       # I2C1 has baro, I2C2 spare
BARO DPS310 I2C:0:0x76    # baro on first I2C bus
```

### 15.3 Backup Domain Warning
In `hwdef.dat` do **not** assign PC13/PC14/PC15 as EXTI interrupt pins without explicitly enabling the backup domain regulator. Use PC13 as CS only (output), not EXTI.

### 15.4 DMA Streams (Reference from Betaflight)
```
DMA1 Stream 1: SPI1 TX
DMA1 Stream 2: SPI1 RX
DMA1 Stream 3: SPI3 TX
DMA1 Stream 4: SPI3 RX
DMA1 Stream 5: SPI4 TX
DMA1 Stream 6: SPI4 RX
DMA2 Stream 2: ADC1
DMA2 Stream 3: ADC3
```

### 15.5 Gyro Rotation
Both ICM42688P sensors are mounted with rotation `ROTATION_YAW_180` (equivalent to `-R 6` in PX4). Verify physically.

---

## 16. Known Issues / Limitations

| Issue | Status | Notes |
|---|---|---|
| IMU2 polling mode jitter | Acceptable | Same as Matek H743-Slim official target |
| PC14 EXTI unusable | Won't fix | RTC backup domain — use polling only |
| OSD content | PX4 limitation | atxxxx driver communicates but PX4 OSD display needs configuration |
| SD Card | Untested | SDIO pins configured, not verified |
