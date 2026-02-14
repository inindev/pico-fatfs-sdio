add_library(pico-fatfs-sdio INTERFACE)

pico_generate_pio_header(pico-fatfs-sdio ${CMAKE_CURRENT_LIST_DIR}/sd_driver/SDIO/rp2040_sdio.pio)

target_compile_definitions(pico-fatfs-sdio INTERFACE
    PICO_MAX_SHARED_IRQ_HANDLERS=8u
)
# target_compile_options(pico-fatfs-sdio INTERFACE -ffile-prefix-map=${CMAKE_CURRENT_LIST_DIR}=)

target_sources(pico-fatfs-sdio INTERFACE
    ${CMAKE_CURRENT_LIST_DIR}/../lib/fatfs/source/ff.c
    ${CMAKE_CURRENT_LIST_DIR}/../lib/fatfs/source/ffsystem.c
    ${CMAKE_CURRENT_LIST_DIR}/../lib/fatfs/source/ffunicode.c
    ${CMAKE_CURRENT_LIST_DIR}/sd_driver/dma_interrupts.c
    ${CMAKE_CURRENT_LIST_DIR}/sd_driver/sd_card.c
    ${CMAKE_CURRENT_LIST_DIR}/sd_driver/sd_timeouts.c
    ${CMAKE_CURRENT_LIST_DIR}/sd_driver/SDIO/rp2040_sdio.c
    ${CMAKE_CURRENT_LIST_DIR}/sd_driver/SDIO/sd_card_sdio.c
    ${CMAKE_CURRENT_LIST_DIR}/src/crash.c
    ${CMAKE_CURRENT_LIST_DIR}/src/crc.c
    ${CMAKE_CURRENT_LIST_DIR}/src/f_util.c
    ${CMAKE_CURRENT_LIST_DIR}/src/glue.c
    ${CMAKE_CURRENT_LIST_DIR}/src/my_debug.c
    ${CMAKE_CURRENT_LIST_DIR}/src/my_rtc.c
    ${CMAKE_CURRENT_LIST_DIR}/src/util.c
)
target_include_directories(pico-fatfs-sdio INTERFACE
    include
    ../lib/fatfs/source
    sd_driver
)

if(PICO_RISCV)
    set(HWDEP_LIBS hardware_watchdog)
else()
    set(HWDEP_LIBS cmsis_core)
endif()

target_link_libraries(pico-fatfs-sdio INTERFACE
    hardware_dma
    hardware_pio
    hardware_sync
    pico_aon_timer
    pico_stdlib
    ${HWDEP_LIBS}
)
