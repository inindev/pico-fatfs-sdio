# pico-fatfs-sd

SD card driver (SDIO and SPI) with FatFs for RP2040 and RP2350 (Raspberry Pi Pico).

This is a trimmed fork of carlk3's [no-OS-FatFS-SD-SDIO-SPI-RPi-Pico](https://github.com/carlk3/no-OS-FatFS-SD-SDIO-SPI-RPi-Pico). The following have been removed: C++ wrappers, examples, and vendored third-party code (ZuluSCSI, mbed-os, SdFat).

## What's here

- **FatFs R0.16** (elm-chan's Generic FAT/exFAT filesystem module) in `lib/fatfs/`
- **4-bit SDIO driver** using PIO, derived from ZuluSCSI-firmware, in `src/sd_driver/SDIO/`
- **SPI driver** using DMA, in `src/sd_driver/SPI/`
- **Glue code** (DMA interrupts, CRC, RTC, debug) in `src/`
- **Custom ffconf.h** in `src/include/` with LFN, exFAT, UTF-8, and other options enabled

## Changes from upstream

1. **Trimmed** — C++ wrappers, examples, and stubs removed
2. **FatFs R0.16** — upgraded from R0.15
3. **RP2350B extended GPIO support** — five patches to the SDIO driver for GPIOs 32-47:
   - CLK pin calculation accounts for `pio_get_gpio_base()` offset
   - `pio_set_gpio_base()` called during PIO init for GPIO window > 31
   - `input_sync_bypass` uses PIO-relative pin numbers
   - `sd_sdio_isBusy()` uses `gpio_get()` instead of raw `sio_hw->gpio_in`
   - `gpio_conf()` derives PIO function from the configured PIO instance

These patches are backward-compatible with RP2040.

## Directory structure

```
lib/fatfs/              FatFs R0.16 (source + docs)
src/
  CMakeLists.txt        INTERFACE library target
  include/
    ffconf.h            Customized FatFs configuration
    hw_config.h         Hardware config API (you implement this)
    ...
  sd_driver/
    sd_card.h           Main driver API
    sd_card.c           Driver init (dispatches to SPI or SDIO)
    SDIO/
      rp2040_sdio.pio   PIO program for 4-bit SDIO
      rp2040_sdio.c     PIO/DMA transfer engine
      sd_card_sdio.c    SDIO card protocol (CMD, ACMD, init, R/W)
      SdioCard.h        SDIO function prototypes
    SPI/
      my_spi.c/h        DMA-based SPI transport layer
      sd_spi.c/h        SD-over-SPI helpers (select, frequency)
      sd_card_spi.c/h   SPI card protocol (CMD, init, R/W)
    ...
```

## Usage

Include as a subdirectory in your CMake project:

```cmake
add_subdirectory(pico-fatfs-sd/src build_fatfs)
target_link_libraries(your_target pico-fatfs-sd)
```

You must provide a `hw_config.c` that implements `sd_get_num()` and `sd_get_by_num()`. Set `.type` to `SD_IF_SDIO` or `SD_IF_SPI` depending on your hardware.

### SDIO example (Adafruit Fruit Jam — RP2350B, SDIO on GPIO 34-39)

```c
#include "hw_config.h"
#include "sd_card.h"

static sd_sdio_if_t sdio_if = {
    .CMD_gpio = 35,
    .D0_gpio = 36,
    .SDIO_PIO = pio1,
    .DMA_IRQ_num = DMA_IRQ_1,
};

static sd_card_t sd_card = {
    .type = SD_IF_SDIO,
    .sdio_if_p = &sdio_if,
    .use_card_detect = true,
    .card_detect_gpio = 33,
    .card_detected_true = 0,
    .card_detect_use_pull = true,
    .card_detect_pull_hi = true,
};

size_t sd_get_num(void) { return 1; }
sd_card_t *sd_get_by_num(size_t num) {
    return (num == 0) ? &sd_card : NULL;
}
```

### SPI example

```c
#include "hw_config.h"
#include "sd_card.h"

static spi_t spi = {
    .hw_inst = spi0,
    .miso_gpio = 4,
    .mosi_gpio = 3,
    .sck_gpio = 2,
    .baud_rate = 12500000,  // 12.5 MHz
};

static sd_spi_if_t spi_if = {
    .spi = &spi,
    .ss_gpio = 7,
};

static sd_card_t sd_card = {
    .type = SD_IF_SPI,
    .spi_if_p = &spi_if,
};

size_t sd_get_num(void) { return 1; }
sd_card_t *sd_get_by_num(size_t num) {
    return (num == 0) ? &sd_card : NULL;
}
```

## Hardware resources

**SDIO:** One PIO instance, one state machine, and two DMA channels. The PIO program is loaded into instruction memory at init time. On RP2350B with GPIOs >= 32, the driver calls `pio_set_gpio_base()` to shift the PIO GPIO window.

**SPI:** One SPI peripheral (spi0 or spi1) and two DMA channels for full-duplex transfers.

## License

Apache 2.0 (upstream), FatFs license (see `lib/fatfs/LICENSE.txt`).
