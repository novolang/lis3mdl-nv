# lis3mdl-nv

A driver for ST's LIS3MDL, a three-axis magnetometer with an I2C and an
SPI interface, specified in ST's data sheet "LIS3MDL: Digital output
magnetic sensor: ultra-low-power, high-performance 3-axis magnetometer",
DocID024204 Rev 4, May 2015.  The driver implements the `Magnetometer`
and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv) and reads the
field in nanotesla.

## Install

```sh
novo pkg add lis3mdl-nv
```

## Example

A device program reads the part over I2C.  The breakout's SDA and SCL go
to the board's I2C pins and its SDO/SA1 pin to ground, which puts the
part at 0x1C.  The program brings the part up at ±4 gauss in ultra-high
performance at 10 Hz and prints one line per sample over RTT.  The
Earth's field at the surface is between about 25 000 and 65 000 nT, so
the three axes together read about that much.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use dev_mag.{ MagNanoT, Magnetometer }
use lis3mdl
use lis3mdl.{ Lis3mdl }

// The part's address with SDO/SA1 low.
const ADDR = 0x1C

// A label and a number, written without building a string.
fn field(label: Str, v: Int) [hw]
    hal.uart.write(label)
    hal.uart.write(str.from_int(v))

@tier(embedded)
fn main() [hw]
    let part = Lis3mdl.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR))
    if not part.begin()
        hal.uart.write("lis3mdl: no answer\n")
        return
    loop
        match part.mag_read_new()
            Some(m) =>
                hal.uart.write("lis3mdl")
                field(" x=", m.x)
                field(" y=", m.y)
                field(" z=", m.z)
                hal.uart.write("\n")
            None =>
                hal.timer.delay_ms(5)
```

The program is `tests/device_read.nv`.  Build it for a board with:

```sh
novo build --target=nrf52840-dk tests/device_read.nv
novo build --target=nrf54lm20-dk tests/device_read.nv
novo build --target=frdm-mcxn947 tests/device_read.nv
```

| Board | Flash | Of which the program | RAM at start |
|---|---|---|---|
| nRF52840 DK | 19,584 bytes | 4,552 bytes | 3,252 bytes |
| nRF54LM20 DK | 20,540 bytes | 4,716 bytes | 3,236 bytes |
| FRDM-MCXN947 | 21,484 bytes | 6,182 bytes | 2,748 bytes |

All three boards drive I2C and SPI, and each row is the image with the
driver's reading path in it.  The nRF52840 DK's bus is TWIM0 on SDA P0.26
and SCL P0.27, and the nRF54LM20 DK's is TWIM22 on SDA P1.03 and SCL
P1.04 (`docs/hal/README.md` in the novo-lang repository, "The Nordic DKs'
buses").  Before the Nordic buses landed on 2026-11-10 the two DK board
packages answered for I2C and SPI as absent hardware, the build removed the
code that follows the first bus read, and the rows were 13,448 bytes of
flash and 2,660 bytes of RAM (nRF52840 DK) and 14,220 bytes and 2,644 bytes
(nRF54LM20 DK).  A reply from a LIS3MDL over those buses is NOT VERIFIED
until the driver rig.  The rest of each image is the runtime, the
compiler's support library and tables.  The driver's own code allocates
nothing: every buffer is a fixed-size `Vec` in the caller's frame.

**Hardware: NOT VERIFIED.**  No board carries a LIS3MDL yet.  The program
builds, and the driver's functions run over a double of the part that
follows the data sheet's register map and bus protocols.  A reading from
a part on a board has not been taken.

### The effects

Every function of the driver that reaches the bus declares `[hw]`, and
nothing else.  The driver performs no input or output of its own and
allocates nothing of its own.  The bus functions of devices-nv build one
short frame per transaction inside a `system` block and release it when
the call returns.  The module checks at every tier, `embedded` included.
The program above declares `[hw]` for the same reason.

## What it is

A magnetometer measures the magnetic field along three axes.  The
Earth's field at the surface is between about 25 and 65 microtesla, and
iron or current near the part adds its own.  A program that needs a
heading calibrates the part where it is mounted.

Each axis is a 16-bit two's complement count.  The part measures within
a full scale of ±4, ±8, ±12 or ±16 gauss, and one gauss is 100 000
nanotesla (nT).  The driver answers nT.

| Full scale | Counts per gauss (Table 3) | Step |
|---|---|---|
| ±4 gauss | 6842 | 14.6 nT |
| ±8 gauss | 3421 | 29.2 nT |
| ±12 gauss | 2281 | 43.8 nT |
| ±16 gauss | 1711 | 58.4 nT |

A count is converted as `count * 100000 / counts_per_gauss`, so 6842
counts at ±4 gauss are exactly 100 000 nT.  The driver reads the range
in the same transaction as the sample, so a reading is converted with the
range it was taken at.

The part has four performance modes (Table 21).  A mode sets how many
internal averages a sample takes, and with FAST_ODR also the output data
rate.

| Mode | Rate with FAST_ODR |
|---|---|
| Low power | 1000 Hz |
| Medium performance | 560 Hz |
| High performance | 300 Hz |
| Ultra-high performance | 155 Hz |

Without FAST_ODR the output data rate is one of 0.625, 1.25, 2.5, 5, 10,
20, 40 and 80 Hz (Table 22).  The X and Y axes take their mode from OM in
CTRL_REG1 and the Z axis from OMZ in CTRL_REG4.  The two must agree, and
the driver writes them together.

A new sample sets ZYXDA in STATUS_REG.  The part also has a DRDY pin of
its own, which goes high when a new set of data is ready and low when the
output registers are read.  It needs no configuration.  The INT pin is a
separate threshold interrupt, which the driver leaves off.

The part has two bus interfaces on the same pins.  Chip select high
selects I2C.

| | I2C | SPI |
|---|---|---|
| Address | 0x1C, or 0x1E with SDO/SA1 high | chip select low |
| Several registers in one read | bit 7 of the sub-address set | bit 6 of the command byte set (MS) |
| Read | sub-address, repeated start, data | command byte with bit 7 set (RW), then data |
| Clock | 400 kHz | 4 MHz, mode 3 (up to 10 MHz) |

The two buses put the increment bit in different places.  On SPI, bit 7
is RW, so a driver that copies the I2C sub-address onto SPI reads and
writes the wrong way round.

## What the package contains

| Module | What is in it |
|---|---|
| `lis3mdl` | The types `Lis3mdl` (I2C) and `Lis3mdlSpi` (SPI), `Lis3mdlRange`, `Lis3mdlPerf` and `Lis3mdlPower`, the trait `Lis3mdlPort`, the bus framing, the register access, the conversions, the self-test and the bring-up |

## How to choose an entry point

A program on a board uses `Lis3mdl` for a part on I2C and `Lis3mdlSpi`
for a part on SPI.  Each is built by `attach` with the board's bus, its
GPIO and the wiring, as a board's device table builds it.  Both
implement `Magnetometer` and `Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` (I2C) or with the chip select `w.cs` (SPI); nothing is sent |
| `begin()` | brings the part up, see below |
| `mag_read()`, `mag_ready()`, `mag_read_new()` | `Magnetometer`: the latest sample in nT, the data-ready flag, a sample not read before |
| `mag_rate_hz()` | `Magnetometer`: the rate in hertz, rounded up, so 0.625 Hz is 1 |
| `device_id()` | `Identifiable`: WHO_AM_I, 0x3D |
| `range()`, `set_range(r)` | the full scale, changed without touching the other bits of CTRL_REG2 |
| `perf()`, `set_perf(m)` | the performance mode of X, Y and Z |
| `power()`, `set_power(m)` | continuous, single conversion or power-down |
| `rate_mhz()`, `set_rate_mhz(mhz)` | the rate in millihertz, and the slowest rate at or above `mhz` |
| `self_test()` | the change in nT that the part's self-test makes |
| `irq_level()` | the level of the DRDY pin, -1 for a wiring with none |
| `reg(r)`, `set_reg(r, v)` | one register, and a write that is read back |

`begin` checks WHO_AM_I, puts the part in power-down and the registers in
a known state because the part has no reset pin and keeps them while
powered, then sets ±4 gauss, ultra-high performance, block data update
and 10 Hz, starts continuous conversion, and reads the output registers
once so that DRDY starts low.  It answers `false` for a part that does
not answer 0x3D and for a register that does not hold what was written.

A program that holds its own bus, or a test with a double of the part,
implements `Lis3mdlPort` over its bus and calls the module's functions,
which take the port: `lis3mdl.sample_read(port)`, `lis3mdl.start(port)`.
The framing of each bus is `i2c_read_into`, `i2c_write`, `spi_read_into`
and `spi_write`, generic over `I2cBus` and `SpiBus`.

## The rules a user needs

1. `begin` has to run before a reading: the part is in power-down at
   power-on (CTRL_REG3 MD is 11).
2. A rate above 80 Hz is set by the performance mode, with FAST_ODR.
   `set_rate_mhz` sets the mode with the rate: 155, 300, 560 or 1000 Hz
   for ultra-high, high, medium and low-power.  A rate up to 80 Hz
   clears FAST_ODR and LP and keeps the mode.
3. `set_rate_mhz` answers the rate taken, which is rarely the rate asked
   for.  It answers 0 for a part that did not take it.
4. Single-conversion mode (`set_power`) is for rates from 0.625 to 80 Hz.
   The part converts once and goes to power-down.
5. DRDY stays high until the output registers are read.  A program that
   waits for its rising edge reads one sample first, as `begin` does.
6. The self-test runs the part at ±12 gauss, ultra-high performance and
   80 Hz, and answers the second reading minus the first, in nT.
   `self_test_ok` checks it against Table 3: 1 to 3 gauss on X and Y and
   0.1 to 1 gauss on Z, in magnitude.  Each reading is converted before
   the subtraction, so the change can differ by 1 nT from the count
   difference converted.
7. `device_id` answers `None` for a bus that no part drives: an SPI bus
   reads 0xFF, or 0x00 when held low.
8. Iron or current near the part adds to the field, and a heading needs a
   calibration where the part is mounted.

## What is not included

- The threshold interrupt on the INT pin (INT_CFG, INT_SRC, INT_THS):
  no trait in devices-nv names it.  `reg` and `set_reg` reach it.
- The temperature sensor (TEMP_EN, TEMP_OUT).  The sheet gives 8 counts
  per degree and no reference temperature.
- The software reset and the reboot of the memory content (CTRL_REG2,
  SOFT_RST and REBOOT), whose settling time the sheet does not give.
- The big-endian output (BLE), FAST_READ, and the SPI three-wire mode
  (SIM).
- A FIFO: the part has none.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  traits this driver implements, the wiring record and the register
  helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus`, `SpiBus` and `GpioOut` traits and the board handles.
- [lsm303agr-nv](https://novo-lang.org/packages/lsm303agr-nv) drives the
  accelerometer and the magnetometer of the LSM303AGR, a different
  magnetometer at a fixed 150 nT step.
- lis3dh-nv drives ST's LIS3DH accelerometer, whose registers follow the
  same conventions.

## Tests

`novo test` runs the module over a double of the part that answers on an
I2C bus and on an SPI bus: the sub-address and the command byte, the
increment, WHO_AM_I, the conversion at every range, the rate bits and the
FAST_ODR rates, the range, performance and power changes, the bring-up
and its refusals, the self-test and the two types over the host's board
handles.  The log of bytes sent is compared for each entry point.  The
suite covers every line of `src/`.

**Hardware: NOT VERIFIED.**  See the example section.

The sources read for this driver, for their fixes and not ported, are
the ST data sheet above, Zephyr's `drivers/sensor/st/lis3mdl` and the
`lis3mdl-driver` crate 0.1.0.  Each fix kept is named in `src/lis3mdl.nv`
where it is kept.

## Licence

Apache-2.0.
