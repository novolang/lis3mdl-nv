# Changelog

## 0.1.0

The first release: a driver for ST's LIS3MDL three-axis magnetometer on an
I2C or an SPI bus.

- `Lis3mdl` (I2C) and `Lis3mdlSpi` (SPI), the types a board's device table
  builds, implementing devices-nv's `Magnetometer` and `Identifiable`, with
  their own methods for the range, the performance mode, the operating mode,
  the rate in millihertz, the self-test and register access.
- The register access as functions over `Lis3mdlPort`, a trait that a type
  holding a bus implements, and the framing of each bus as functions over
  `I2cBus` and `SpiBus`.
- Readings in nanotesla at the sensitivity of the data sheet's Table 3,
  converted with the range read in the same transaction as the sample.
- The performance mode written to OM and OMZ together, and the FAST_ODR
  rates above 80 Hz set through it.
- The self-test as a function that answers the change in nanotesla, and a
  comparison with Table 3's limits.
- The suite runs over a double of the part on both buses and covers every
  line of `src/`.

### Known

- Hardware verification: NOT VERIFIED.  No board carries a LIS3MDL.
- The device program builds for the nRF52840 DK and the nRF54LM20 DK.  Their
  board packages answered for I2C and SPI as absent hardware when the
  package was written (images of 13,448 and 14,220 bytes of flash, without
  the reading path); the TWIM and SPIM drivers, tracked by
  `features/embedded-devex/nordic-twim-i2c-driver.md`, landed on 2026-11-10,
  and the images are now 19,584 and 20,540 bytes with the reading path.  A
  reply from a LIS3MDL over those buses is NOT VERIFIED until the driver rig.
- The data sheet's INT_CFG default and the reserved bits of the register
  were not exercised: the driver does not write INT_CFG.
