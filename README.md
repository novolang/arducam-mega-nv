# arducam-mega-nv

A driver for Arducam's Mega SPI camera modules, the 3 MP and the 5 MP, on an SPI bus. The modules are specified in Arducam's "Mega SPI Camera Series Application Note" (September 2023) and in Arducam's open C SDK, `ArduCAM/Arducam_Mega`, commit `146e058b1b5e` (SDK version 3.0.0, 2025-06-20). A module takes a picture in JPEG, RGB565 or YUV and holds it in an 8 MB FIFO that the host reads over SPI. The driver sets the format and the size, takes a picture, hands the FIFO to the caller in chunks the caller sizes, finds the start and the end of a JPEG in the chunks, and sets the image controls the SDK has. The 2 MP module that the SDK also lists is covered by the same code. A program on a computer finds the same markers in a log and writes the picture to a file with `mega_file`.

**Hardware verification: NOT VERIFIED.** No board on the bench carries an Arducam Mega module, so the driver has run only against a double of the module, in the host suite, which plays a JPEG made on the host and not a recording of a camera. The device program below builds for the nRF52840 DK and the nRF54LM20 DK and has not been flashed. Neither Nordic board package drives its SPI peripheral yet, so a program on either board reaches the module only after that driver exists (see "The device program").

## The device program

`examples/capture.nv` waits half a second for the module, brings it up, sets JPEG at 640x480, and once a second takes a picture, reads it in chunks of 64 bytes while it looks for the JPEG's markers, and prints over the board's UART:

```text
model 133
fifo 11264
jpeg 11250 bytes
```

`model` is the sensor ID byte in decimal (133 is 0x85). `fifo` is the length the module reports and `jpeg` the length between the JPEG's start and end markers, which can be shorter.

```novo norun:needs-pkg
use embedded_hal.{ BoardSpi, BoardGpio, BoardTimer }
use dev_bus
use arducam_mega
use arducam_mega.{ Arducam, MegaChunk }
use mega_jpeg

// The chip-select pin, the bytes read in one burst, and the most bursts a
// picture can take: the FIFO is 8 MB, which is 131,072 bursts of 64 bytes.
const CS_PIN = 28
const CHUNK = 64
const MAX_CHUNKS = 131072

// Takes one picture and reports it.
@tier(embedded)
fn take(var cam: Arducam) [hw]
    if not cam.capture()
        hal.uart.write("capture failed\n")
        return
    var scan = mega_jpeg.scan_new()
    var chunk: MegaChunk = Vec.new()
    for _i in 0..MAX_CHUNKS
        let n = cam.read_chunk(CHUNK, var chunk)
        if n == 0
            break
        mega_jpeg.scan_feed(var scan, chunk, n)
    let _ = cam.finish()
    if mega_jpeg.scan_done(scan)
        hal.uart.write("jpeg ")
        // ... the length, mega_jpeg.scan_length(scan), written a digit at a time
        hal.uart.write(" bytes\n")
    else
        hal.uart.write("no end marker\n")

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    var cam = Arducam.attach(BoardSpi { id: 0 }, BoardTimer { id: 0 }, BoardGpio { id: 0 },
                             dev_bus.wiring_spi(0, CS_PIN))
    hal.timer.delay_ms(500)
    if not cam.begin()
        hal.uart.write("no answer from the camera\n")
        return
    if not cam.configure(arducam_mega.FMT_JPEG, arducam_mega.RES_640X480)
        hal.uart.write("format refused\n")
        return
    while true
        take(var cam)
        hal.timer.delay_ms(1000)
```

The complete program is `examples/capture.nv`. `tests/device_build.sh` builds it from a scratch package that depends on this one, for the two boards below and for the emulated board, where it runs the image and checks that the program reports that the camera does not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM at start (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 18,420 B | 6,164 B | 2,660 B |
| `nrf54lm20-dk` | 19,292 B | 6,288 B | 2,644 B |

The numbers are from the build's memory report. Flash also holds the runtime, the compiler's support library and the program's tables. The arena, 65,536 B on both boards, is the runtime's heap and is reserved whether or not the program uses it; the driver allocates nothing in it. The driver's state is an `Arducam` value of 152 bytes and the chunk buffer a `MegaChunk` of 264 bytes, both in the program's frame.

Neither Nordic board package names a driver for its SPI peripheral (`@hal(spi)`), so `BoardSpi` is absent hardware on both: every transfer is refused or answers nothing, and the program prints `no answer from the camera`. The feature `features/embedded-devex/nordic-twim-i2c-driver.md` in the novo-lang repository tracks the missing Nordic bus drivers. The wiring when a bus exists: SCK, MOSI and MISO to the board's first SPI bus, CS to a free GPIO pin, which the wiring record names, and the module's supply to 3.3 V.

### The effects

Every function of the driver that reaches the bus, a pin or the timer declares `[hw]`, and nothing else. The driver performs no input or output of its own and allocates nothing of its own. The SPI helpers of devices-nv build one short frame per transaction in a buffer of the caller's frame. The package checks at the `embedded` tier, and so at every tier, except the module `mega_file`, which is for a computer and is checked at the `app` tier. A capture spins the core, polling the module, until the module finishes or two seconds pass.

## What the module is

An Arducam Mega is a camera module with an image sensor, a logic chip that holds a picture in an 8 MB memory (the FIFO) and an SPI interface. The host never reads the sensor directly. It writes the format, the size and the controls to registers, starts a capture, waits for a flag, reads the length of the picture from three registers and then reads the picture out of the FIFO.

The SPI interface is a slave in mode 0 (CPOL 0, CPHA 0). Arducam recommends 8 MHz, and chip select stays low for the whole transaction. There are four bus cycles (Application Note section 3):

| Cycle | Bytes after chip select falls |
|---|---|
| Single read | the register (bit 7 clear), a dummy byte, then one byte clocked out by the host while the register value comes back |
| Single write | the register with bit 7 set, then the value |
| First burst read | `0x3C`, one dummy byte, then the FIFO bytes |
| Nth burst read (N is 2 or more) | `0x3C`, then the FIFO bytes |

The FIFO read pointer carries on where the last burst stopped, so a picture is read in as many bursts as the caller likes. Only the first burst of a capture has the dummy byte. The SDK keeps that fact in a flag, `burstFirstFlag`. The method of a trait receives its value and cannot change it, so the driver keeps the same fact in `MegaState`, which the caller holds and passes `var`.

The module has a second bus inside it, to the image sensor. A write to a control register is passed on over it, and the SDK waits after each write for the register at `0x44` to report the bus idle (bits 1:0 equal to `0b10`). The driver does the same.

### The capture

The sequence is the SDK's `cameraSetCapture`:

| Step | Register | What it does |
|---|---|---|
| clear the flag | `0x04` bit 0 | clears the capture-done flag |
| start | `0x04` bit 1 | starts one capture |
| wait | `0x44` bit 2 | set when the picture is in the FIFO; polled every 2 ms, at most 1000 times |
| length | `0x45`, `0x46`, `0x47` | the FIFO length, 24 bits, low byte first |
| read | burst command `0x3C` | the picture, in chunks |
| end | `0x04` bit 0 | clears the flag again |

For JPEG the FIFO length is the most the picture can be. The Rust crate for the module found trailing zero bytes after the picture, so the picture ends at the first `FF D9` after the first `FF D8`. `mega_jpeg` finds both in the chunks as they arrive. It keeps the last byte between chunks, so a marker split across two chunks is found, and says for each chunk which of its bytes belong to the picture.

### The modules

The sensor ID byte at `0x40` names the module. The SDK has two families. The three modules of the new family (`0x85` to `0x87`) use the SDK's own size enumeration as the size code and have the exposure freezes. The four older modules (`0x81` to `0x84`) use the size codes of the Application Note and have, according to the module, autofocus or a sharpness register.

| ID | Module | Family | Sizes | Largest size | Also has | Exposure | Gain |
|---|---|---|---|---|---|---|---|
| `0x81` | 5 MP | legacy | 9 | 2592x1944 | autofocus | 1 to 30000 | 1 to 1023 |
| `0x82` | 3 MP | legacy | 9 | 2048x1536 | sharpness | 1 to 30000 | 1 to 31 (the OV3640 table) |
| `0x83` | 5 MP | legacy | 9 | 2592x1944 (2592x1936 in Zephyr) | autofocus | 1 to 65535 | 1 to 65535 |
| `0x84` | 3 MP | legacy | 9 | 2048x1536 | sharpness | 1 to 65535 | 1 to 65535 |
| `0x85` | 5 MP | new | 14 | 2592x1944 | AE and AWB freeze | 1 to 10000 | 0 to 240 |
| `0x86` | 3 MP | new | 13 | 2048x1536 | AE and AWB freeze | 1 to 10000 | 0 to 240 |
| `0x87` | 2 MP | new | 11 | 1600x1200 | AE and AWB freeze | 1 to 10000 | 0 to 240 |

The nine sizes of a legacy module are 96x96, 128x128, 320x240, 320x320, 640x480, 1280x720, 1600x1200, 1920x1080 and the largest. The new family takes in addition 160x120, 800x600, 1024x768 and 1280x1024, and up to 1600x1200 (2 MP), 2048x1536 (3 MP) or 2592x1944 (5 MP).

The differences between the 3 MP and the 5 MP, as the SDK and the Application Note give them:

| | 3 MP | 5 MP |
|---|---|---|
| Largest size (Application Note, register `0x21`) | 2048x1536 (code 8 on a legacy module) | 2592x1944 (code 9 on a legacy module) |
| Special effects | codes 0 to 6, and 9 (light yellow) | codes 0 to 6 |
| Legacy extras | sharpness at `0x28` | autofocus at `0x29` |
| Sizes of the new family | 13 | 14 |
| Pitfall | the OV3640 gain table on ID `0x82` | the Rust crate reports a green cast with the automatic white balance on |

## Install

```sh
novo pkg add arducam-mega-nv
```

## What the package contains

| Module | What is in it |
|---|---|
| `arducam_mega` | The types `Arducam` (the module on a board), `MegaLink` (its bus, timer and pin), `MegaState`, the trait `MegaPort`, the SPI framing, the bring-up, the format and size, the capture, the chunk read, the image controls, the low-power mode, and the tables of the seven modules |
| `mega_jpeg` | The start and end marker scan over the chunks of a capture |
| `mega_file` | For a computer: trimming a captured JPEG to its markers and writing it to a file |

## How to choose an entry point

A program on a board uses `Arducam`, built by `attach` with the board's SPI bus, timer and GPIO and the wiring (the chip select is `w.cs`). A program keeps it in a `var`: the bring-up, the format and the chunk read change the state it holds.

| Method | What it does |
|---|---|
| `attach(bus, timer, pins, w)` | the module with chip select `w.cs`; nothing is sent |
| `begin()` | resets the module, waits for it, checks the sensor ID against the seven and sets the sensor's I2C address; whether the module answered |
| `chip_id()`, `firmware()`, `model()` | the sensor ID byte, the firmware date and number, the ID the bring-up read |
| `configure(fmt, res)` | the format (`FMT_JPEG`, `FMT_RGB565`, `FMT_YUV`) and the size (a `RES_` constant the module takes) |
| `capture()` | takes one picture and reads its length; whether it finished within 2 s |
| `frame_len()`, `frame_left()` | the FIFO length, and the bytes not yet read |
| `read_chunk(n, var dst)` | up to `n` bytes (at most 256) into a `MegaChunk`; how many were read |
| `finish()` | clears the capture-done flag |
| `reset()` | resets the module and forgets the format and the size |
| `sleep()`, `wake()` | the low-power mode |
| `set_brightness(step)`, `set_contrast(step)`, `set_saturation(step)`, `set_ev(step)` | steps from -4 to +4 (brightness) and -3 to +3 |
| `set_white_balance(mode)`, `set_effect(fx)`, `set_quality(q)`, `set_rotation(turned)` | one of the named constants |
| `set_sharpness(level)`, `set_focus(mode)` | on the legacy modules that have them |
| `set_auto(which, on)`, `set_exposure(v)`, `set_gain(v)` | automatic or manual gain, exposure and white balance |
| `freeze_ae(frozen)`, `freeze_awb(frozen)` | on the new family |

The constants carry the SDK's names in upper case: `FMT_` for the formats, `RES_` for the sizes, `FX_` for the effects (none, blueish, redish, black and white, sepia, negative, greenish, over exposure, solarize, light yellow), `WB_` for the white balance (auto, daylight, office, cloudy, indoor), `QUALITY_` and `AUTO_`.

A program that holds its own bus, or a test with a double of the module, implements `MegaPort` over its bus and calls the module's functions, which take the port and the state: `arducam_mega.bring_up(port, var st)`, `arducam_mega.frame_capture(port, var st)`, `arducam_mega.chunk_read(port, var st, n, var dst)`. The framing of the bus is `spi_reg_read`, `spi_reg_write` and `spi_burst`, generic over `SpiBus` and `GpioOut`.

### The trait this would implement

devices-nv has no trait for a camera. Its planning page proposes `ImageSensor`, with a configure method for a format, a width, a height and a rate, a start and a stop, one frame at a time into a buffer the caller owns, a frame-complete event and a frame status, and exposure and gain as optional methods. This package declares its own functions in that shape, and will implement the trait when it exists:

| The proposed method | Here |
|---|---|
| configure | `format_set` with a format and a `RES_` size; the module has no rate setting |
| start | `frame_capture`, which waits for the frame to complete |
| frame complete, frame status | the answer of `frame_capture`; `false` for a capture that did not finish or left no length |
| read into the caller's buffer | `chunk_read`, with the caller's `MegaChunk` and count |
| stop | `frame_end` |
| exposure, gain | `exposure_set`, `gain_set` and `auto_set` |

## The rules a user needs

1. Power the module and wait before `begin`. The sources read give no figure for the wait. The device program waits 500 ms, which is an assumption.
2. The SPI clock is at most 8 MHz in mode 0 (Application Note section 2). `spi_setup` drives chip select high before it makes the pin an output, so the module is never selected by accident.
3. Every wait ends. The SDK waits for the sensor bus and for the capture without a limit and hangs on a module that is absent. The driver polls every 2 ms: 100 times after a control write, 600 times after a reset and 1000 times for a capture.
4. A capture is refused, with nothing sent, before `begin`, before `configure` and while the module sleeps. The SDK's capture loop gives up at once while its low-power flag is set and then reads a length before the picture exists.
5. A format change rewrites the size. The SDK does so on the new family, and the driver does so on every module. A call that repeats the format and the size sends nothing.
6. Read a picture from its first chunk on, and read all of it before the next capture. The first read after a capture clocks the dummy byte and the later ones do not, and a new capture starts again with the dummy byte.
7. A burst that fails leaves the first-burst flag and the count as they were, so the call can be repeated.
8. For JPEG, use the markers and not the length. The length can include bytes after `FF D9`.
9. The brightness, contrast, saturation and EV registers take a code that alternates: 0 is the default, 1 is +1, 2 is -1, 3 is +2, and so on. `level_code` turns a step into the code, and a step beyond the limit is cut to it.
10. A control the module does not have is refused and sends nothing: sharpness on a module without it, an effect the module's mask lacks, a gain outside the range, and the gain index of ID `0x82` outside 1 to 31, which the SDK indexes without a check.
11. Without a module to try them on, the low-power registers follow the SDK, and Zephyr's driver differs for the modules `0x83` and `0x84`, where it inverts the value of register `0x02`. NOT VERIFIED.
12. The automatic white balance gives a strong green cast on the 5 MP in the Rust crate's report. This driver does not check that.

## What is not included

NOT IMPLEMENTED:

- Preview and video mode (bit 7 of register `0x21`), the callback the SDK offers for it, and taking several pictures in one capture (register `0x01`).
- Manual focus and the autofocus status, which go through the image sensor's own registers, and the debug-register write other than the low-power sequence.
- Flushing the FIFO's cache (register `0x07`, bit 7), which the SDK has commented out.
- A conversion of RGB565 or YUV frames. The module writes them to the FIFO and `chunk_read` hands them over unchanged. The byte order of the YUV frame is not confirmed (a forum thread on the 5 MP is not conclusive, and Zephyr names it YUYV).
- The async form of the traits, which an `async fn` cannot take as a value type.
- A trait of devices-nv. None exists for a camera yet.
- Parallel-bus image sensors. This package is for the SPI modules.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the wiring record and the buffers the SPI helpers use.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names the `SpiBus`, `GpioOut` and `Timer` traits and the board handles.
- [vl53l0x-nv](https://novo-lang.org/packages/vl53l0x-nv) drives a time-of-flight sensor in the same style, with the driver state held by the caller.

## Tests

`novo test` runs the module over a double of the module: a register file, a FIFO that holds a 304-byte JPEG made on the host (`tests/data/capture_16x16.jpg`), and an SPI slave that follows the four bus cycles of the Application Note, including a dummy byte on the first burst only. The suite compares every SPI window byte for byte: the register read and write, the burst with and without the dummy byte, the bring-up of each of the seven IDs and the refusals of an unknown ID and of an absent module, the format and size writes in order for the two families, a capture with its polls and its 24-bit length, a capture that never ends, the whole picture read back in chunks of 1 to 1000 bytes and compared with the file, a burst that fails and is repeated, the markers found for any chunk size with and without trailing zeros, the codes of every control and the refusal of every setting a module lacks, the three low-power sequences, the module tables, the type over the host's board handles, and the host helper writing a file. A register or a transfer that the port refuses is tried at each step of the bring-up, the capture and the low-power sequences, and the call must answer that it failed.

The suite covers every line of `src/`:

```sh
NOVO=bin/novo bash tests/coverage.sh
```

**Hardware: NOT VERIFIED.** See the device program section.

## Sources

The sources read for this driver, for their fixes and not ported, are Arducam's Application Note (<https://blog.arducam.com/downloads/datasheet/Arducam_MEGA_SPI_Camera_Application_Note.pdf>), the SDK at the commit above (MIT licence; `src/Arducam/ArducamCamera.c` and `ArducamCamera.h`), Zephyr's `drivers/video/video_arducam_mega.c` (Apache-2.0, `main` at 85681eef80aa on 2026-11-10) and the `arducam-mega` Rust crate of unticks (version 0.4.0, commit 7927fae557f0). Embassy and esp-hal hold no driver for the module. The pitfalls taken from them are named in `src/arducam_mega.nv` at the sites that avoid them:

- The SDK waits for the sensor bus and for the capture without a limit; here every wait ends.
- The SDK's capture loop breaks out while its low-power flag is set; here a capture is refused while the module sleeps. Zephyr stops waiting after 200 tries and reads the length anyway; here a capture that did not finish is a failure.
- The SDK indexes the OV3640 gain table with the caller's number; here the number is checked.
- The SDK's mask of special effects has a bit for "yellowish" and no code for it; the code, 9, is from the Application Note.
- Zephyr accepts only the IDs `0x81` to `0x84`, and the SDK has no check; here the seven IDs are accepted and nothing else.
- The Rust crate reads the FIFO in chunks of 63 bytes with a dummy byte in each. The Application Note and the SDK put the dummy byte in the first burst only, and the chunk size is the caller's.
- The FIFO length can exceed the picture (the Rust crate); here the picture ends at `FF D9`.
- The Application Note's size codes are the legacy codes; the SDK's own enumeration is a different numbering, mapped by `legacy_res_code`.

## Licence

Apache-2.0.
