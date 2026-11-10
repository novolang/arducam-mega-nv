# Changelog

## 0.1.0

The first release: a driver for Arducam's Mega SPI camera modules, the 3 MP
and the 5 MP, and the 2 MP the SDK lists with them.

- The SPI protocol of the Application Note (September 2023): mode 0 at 8 MHz,
  the register read with its dummy byte, the register write with bit 7 set,
  and the burst read of the FIFO with a dummy byte in the first burst of a
  capture only. `spi_reg_read`, `spi_reg_write`, `spi_burst` and `spi_setup`
  are generic over `SpiBus` and `GpioOut`.
- `MegaPort`, the trait a type that holds a bus implements, and `MegaState`,
  the state the SDK keeps in its camera record (the sensor ID, the format and
  size last written, the first-burst flag, the length and the bytes left,
  the sleep flag), which the caller holds and passes `var`.
- `Arducam`, the type a board's device table builds with `attach`, and the
  same functions without the type: `bring_up`, `format_set`,
  `frame_capture`, `chunk_read`, `frame_end`, `module_reset`, `power_set`
  and the image controls.
- The bring-up: the module's logic reset, the wait for the sensor bus, the
  sensor ID checked against the seven the SDK lists (`0x81` to `0x87`), the
  sensor's I2C address written to the debug register. `firmware_version`
  reads the firmware date and number.
- The capture sequence of the SDK (clear the flag, start, poll, read the
  24-bit length), the formats JPEG, RGB565 and YUV, and the sizes of each
  module: 14 on the new 5 MP, 13 on the new 3 MP, 11 on the 2 MP and 9 on
  each of the four older modules, with the older size codes of the
  Application Note.
- The FIFO read in chunks the caller sizes, up to 256 bytes, into a
  `MegaChunk`.
- `mega_jpeg`: the start (`FF D8`) and end (`FF D9`) markers of a JPEG found
  across the chunks, with the bytes of each chunk that belong to the
  picture.
- The image controls the SDK exposes: brightness, contrast, saturation and
  exposure compensation as steps, the white-balance modes, the special
  effects as named constants, the JPEG quality, rotation, sharpness and
  autofocus where a module has them, the automatic and the manual gain,
  exposure and white balance, and the freezes of the automatic exposure and
  white balance.
- The low-power mode, in the three forms the SDK has.
- `mega_file`, for a computer: `jpeg_trim`, `jpeg_write`, `jpeg_put` and
  `jpeg_save` trim a captured JPEG to its markers and write it to a file,
  replacing the file.
- A device program, `examples/capture.nv`, built for the nRF52840 DK and the
  nRF54LM20 DK by `tests/device_build.sh`, and a coverage script,
  `tests/coverage.sh`.
- The suite runs over a double of the module that plays a JPEG made on the
  host, and covers every line of `src/`.

### Known

- Hardware verification: NOT VERIFIED. No board carries an Arducam Mega
  module.
- Preview and video mode, taking several pictures in one capture, manual
  focus and the autofocus status, and the FIFO cache flush are NOT
  IMPLEMENTED.
- The low-power registers of the modules `0x83` and `0x84` follow the SDK,
  and Zephyr's driver writes them differently. NOT VERIFIED.
- The documents read are the Application Note of September 2023 and the SDK
  at commit `146e058b1b5e` (version 3.0.0).
- The device program builds for the nRF52840 DK and the nRF54LM20 DK, whose
  board packages answer for SPI as absent hardware: on those images `begin`
  answers `false` until the boards drive the bus. The gap is tracked by
  `features/embedded-devex/nordic-twim-i2c-driver.md`.
