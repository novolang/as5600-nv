# Changelog

## 0.1.0

The first release: a driver for ams's AS5600 12-bit magnetic rotary
position sensor on an I2C bus.

- `As5600`, the type a board's device table builds, implementing
  devices-nv's `AngleSensor`, with its own methods for the raw and scaled
  angle, the magnet status, the gain and magnitude, the zero and the range,
  and the CONF fields.  `attach_with_dir` takes a GPIO for the DIR pin.
- The register access as functions over `As5600Port`, a trait that a type
  holding a bus implements, and the framing as functions over `I2cBus`.
- The angle as 12 bits with the magnet state, from ANGLE, so that the zero
  and the range the program set are applied; a weak or strong magnet keeps
  its angle, a missing one has the angle 0.
- ZPOS, MPOS and MANG with the 18-degree minimum of the data sheet, each
  read back, and `zero_here` for the sheet's Option A step 2 and 3.
- CONF as seven fields (power mode, hysteresis, output stage, PWM
  frequency, slow filter, fast filter threshold, watchdog) changed one at a
  time by read, change and write, keeping the factory's reserved bits.
- The times of the data sheet as constants and tables: `POWER_UP_US`,
  `SETTLE_US`, `SAMPLE_US` and `settling_us`.
- A device program, `examples/angle_read.nv`, and `tests/device_build.sh`,
  which builds it for the nRF52840 DK, the nRF54LM20 DK and the
  FRDM-MCXN947 and runs the emulated board's image under QEMU.
- The suite runs over a double of the part and covers every line of `src/`;
  `tests/coverage.sh` measures it.

### Known

- Hardware verification: NOT VERIFIED.  No board carries an AS5600.
- The BURN commands are NOT IMPLEMENTED, on purpose: they write the part's
  one-time-programmable memory and cannot be undone.  `burns()` reads ZMCO.
- The copy of the data sheet read is version [v1-06] 2018-Jun-20 and names
  no document number.  It gives no numeric examples, so the suite's values
  are computed from its register map and formulas.
- The status and the angle are two transactions, because of the sheet's
  pointer rule for ANGLE, RAW ANGLE and MAGNITUDE.  The suite's double
  models that rule as this driver reads it; no part has been read.
- The OUT pin, the AS5600L (a different default address) and an async form
  are not implemented.
- The device program builds for the nRF52840 DK and the nRF54LM20 DK, which
  drive I2C with TWIM0 and TWIM22, so their images carry the reading path:
  19,484 and 20,460 bytes of flash, where before the Nordic buses landed they
  were 15,296 and 16,140 bytes.  A reply from the part over either bus is
  NOT VERIFIED.
- `devices-nv` is the registry range `^0.3.3`, the first release with
  `AngleSensor`.
