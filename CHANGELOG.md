# Changelog

## 0.2.1 — 2026-11-27

The README example and `examples/angle_read.nv` call `part.zero_here` without a binding.  No signature of the package changed.

## 0.2.0 — 2026-11-24

The register reads and writes, and the bus set-up, are devices-nv's `RegPort` and `wiring_*` functions, and `As5600Port` is smaller by them.  The release depends on devices-nv 0.4.1.

### Breaking

- **`As5600Port` has the supertrait `RegPort` and no longer declares `read_regs` and `write_reg`.**  A type that implements `As5600Port` for a bus of its own moves those two methods, unchanged, into an `impl RegPort`.  *Migration:* with `use dev_bus.{ RegPort }`,

  ```novo
  impl RegPort for MyPort
      fn read_regs(self, reg: Int, n: Int, var dst: DevBytes) -> Bool [hw]
          ...
      fn write_reg(self, reg: Int, v: Int) -> Bool [hw]
          ...

  impl As5600Port for MyPort
      ...                      // the other methods stay where they are
  ```

  The package's own types are migrated.  A caller that holds one of them is unaffected.

### Deprecated

- `reg_read`: each is a one-line function over the method or the `wiring_*` function that replaces it, warns W1001 at a call with the replacement named, and is removed in the next release.  `As5600Port` itself is the driver's trait and stays.

### Changed

- The register helpers the module carried (`reg_read`, `reg_write`, `reg_update` over its port) are devices-nv's `RegPort` methods (`read_reg`, `write_reg_verified`, `update_reg_verified` and the others); the module calls them as `p.read_reg(reg)`.
- The bus set-up and the interrupt-level read are devices-nv's `wiring_i2c_setup`, `wiring_spi_setup` and `wiring_irq_level`, called with the part's rate and mode.
- `i2c_setup` stays a function of the module and is not deprecated: it drives the DIR pin low and makes it an output before the bus starts, which `wiring_i2c_setup` does not.  `word12_read`, `word12_write` and the other word helpers stay: they pack 12-bit values.
- Image size on nrf52840-dk, `.text` plus read-only data, before and after: 14,960 bytes before and 14,928 after on the read-out (nrf52-qemu: 10,468 and 10,436).

## 0.1.1 — 2026-11-23

The package asks for embedded-hal-nv ^0.4.0 (was ^0.3.2) and devices-nv ^0.4.0 (was ^0.3.3), the minor of both trait packages that the other drivers and the board packages ask for, so one version of each serves a program that takes this package beside any of them.  Under the older ranges, a program that named this package and a driver on 0.4 stopped with "no version of 'embedded-hal-nv' satisfies every requirement".

No signature of the package changed; the suite passes without an edit.

The device program builds for the boards its README names, flash nrf52840-dk 15,060 B before and 15,068 B after; nrf54lm20-dk 16,156 B before and 16,160 B after; frdm-mcxn947 17,000 B before and 17,032 B after; nrf52-qemu 10,476 B before and 10,480 B after.

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
