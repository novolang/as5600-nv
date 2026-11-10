# as5600-nv

A driver for the AS5600, a 12-bit magnetic rotary position sensor from ams
(now ams OSRAM) with an I2C interface, specified in the data sheet "AS5600,
12-Bit Programmable Contactless Potentiometer", version [v1-06]
2018-Jun-20.  The copy read names no document number; the version and the
date are its footer's.  The driver implements the `AngleSensor` trait of
[devices-nv](https://novo-lang.org/packages/devices-nv): a 12-bit angle
and the state of the magnet, read together with the part's zero and range
applied.

**Hardware: NOT VERIFIED.**  No board carries an AS5600 yet.  The device
program below builds for the nRF52840 DK and the nRF54LM20 DK.  The
driver's functions run over a double of the part that follows the data
sheet's register map and bus protocol.  A reading from a part on a board
has not been taken.

## Install

```sh
novo pkg add as5600-nv
```

The device program is `examples/angle_read.nv`.  Make it the `main` of a package that
depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
novo build --target=frdm-mcxn947 src/main.nv
```

## The device program

`examples/angle_read.nv` reads a breakout over I2C ten times a second and
prints the angle in counts and in millidegrees, and the state of the
magnet, over RTT:

```
as5600 counts=1024 mdeg=90000 magnet=ok
```

The breakout's SDA and SCL go to the board's I2C pins and its DIR pin to
ground, which makes the angle increase with clockwise rotation.  A
diametral magnet turns above the chip.  The program waits the 10 ms the
part takes after power-up, makes the position the shaft is at the zero,
and reads.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use dev_angle
use dev_angle.{ AngleSensor }
use as5600.{ As5600 }

// The part's fixed address.
const ADDR = 0x36

@tier(embedded)
fn main() [hw]
    let part = As5600.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR))
    hal.timer.delay_us(as5600.POWER_UP_US)
    if not part.begin()
        hal.uart.write("as5600: no answer\n")
        return
    let _ = part.zero_here()
    loop
        match part.angle_read()
            Some(r) =>
                hal.uart.write("as5600 counts=")
                hal.uart.write(str.from_int(r.counts))
                hal.uart.write(" mdeg=")
                hal.uart.write(str.from_int(dev_angle.angle_milli_deg(r.counts)))
                hal.uart.write("\n")
            None => hal.uart.write("as5600: no result\n")
        hal.timer.delay_ms(100)
```

The full program, which also prints the state of the magnet, is
`examples/angle_read.nv`.  A script builds it for the boards below by making
it the `main` of a scratch package that depends on this one.  It also runs
the emulated board's image under QEMU, where no part answers, and checks
that the program reports it.

```sh
bash tests/device_build.sh
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 15,296 B | 2,860 B | 2,660 B |
| `nrf54lm20-dk` | 16,140 B | 2,916 B | 2,644 B |
| `frdm-mcxn947` | 21,784 B | 6,178 B | 2,748 B |

The numbers are from the build's memory report.  Flash also holds the
runtime, the compiler's support library and the program's tables.  The
arena, 65,536 B on the Nordic boards, is the runtime's heap and is reserved
whether or not the program uses it.

The two Nordic board packages answer for I2C as absent hardware, so the
build removes the code that follows the first bus read, and their rows are
the image without the driver's reading path.  The novo-lang
repository tracks the missing Nordic I2C driver as a feature.  The FRDM-MCXN947 board
package drives I2C, and its row is the image with the driver's code in it.
On the Nordic boards the program prints `as5600: no answer`.

### The effects

Every function of the driver that reaches the bus or the DIR pin declares
`[hw]`, and nothing else.  The tables, the CONF packing and the magnet
status are pure.  The driver performs no input or output of its own and
builds no list of its own.  The bus functions of devices-nv build one short
frame per transaction inside a `system` block and release it when the call
returns.  The package checks at the embedded tier, so a program of any tier
may call it.  The program above declares `[hw]` for the same reason.

## What it is

A magnetic rotary encoder has a small magnet glued to the end of a shaft
and a chip a millimetre or two away.  The chip senses the direction of the
magnet's field and converts it to an angle over one full turn, with no
contact and no wear.  The AS5600 gives the angle as 12 bits, 0 to 4095, so
one count is 360/4096 of a degree, 0.088 degrees.  The count wraps from
4095 to 0 as the shaft passes the zero; `dev_angle.angle_diff` is the
signed step between two counts that accounts for the wrap.

The reading is only as good as the magnet's position.  The part reports in
STATUS whether it detects a magnet (MD), and whether the field is too
strong (MH) or too weak (ML).  Too far or too weak, the angle is noisy;
too close or too strong, the part saturates; missing, there is no angle.
The AGC register shows the gain the part needed: for the most robust
reading it sits in the middle of its range, 0 to 255 at a 5 V supply and 0
to 128 at 3.3 V.

| Register | Address | What it holds |
|---|---|---|
| ZMCO | 0x00 | the number of times ZPOS and MPOS were burned, 2 bits, read only |
| ZPOS | 0x01, 0x02 | the zero position, 12 bits |
| MPOS | 0x03, 0x04 | the stop position, 12 bits |
| MANG | 0x05, 0x06 | the maximum angle, 12 bits |
| CONF | 0x07, 0x08 | the configuration, 14 bits; bits 15 and 14 are the factory's |
| STATUS | 0x0B | MH (bit 3), ML (bit 4), MD (bit 5) |
| RAW ANGLE | 0x0C, 0x0D | the angle, unscaled and unmodified |
| ANGLE | 0x0E, 0x0F | the angle after ZPOS, MPOS and MANG |
| AGC | 0x1A | the automatic gain |
| MAGNITUDE | 0x1B, 0x1C | the magnitude of the internal CORDIC |
| BURN | 0xFF | the one-time-programmable commands; not implemented |

A 12-bit value is two bytes, the high byte at the lower address, and its
upper nibble is not part of the value.

The range is the part of the turn the output spans.  ZPOS is its start.
Its end is MPOS, or its size is MANG; the sheet says to use one of them.
The range must be greater than 18 degrees, which is 205 counts.  When the
range is narrower than a turn the 4096 counts of ANGLE span the range and
not 360 degrees: the resolution is not kept, it is scaled.

CONF holds seven fields.

| Field | Bits | Values |
|---|---|---|
| PM, power mode | 1:0 | 0 NOM, 1 LPM1, 2 LPM2, 3 LPM3, which poll every 5, 20 and 100 ms |
| HYST, hysteresis | 3:2 | 0 off, 1, 2 or 3 LSBs |
| OUTS, output stage | 5:4 | 0 analog over the full supply range, 1 analog 10% to 90%, 2 PWM |
| PWMF, PWM frequency | 7:6 | 115, 230, 460 or 920 Hz |
| SF, slow filter | 9:8 | 16x, 8x, 4x or 2x |
| FTH, fast filter threshold | 12:10 | 0 for the slow filter only, then 6, 7, 9, 18, 21, 24 and 10 LSBs |
| WD, watchdog | 13 | 0 off, 1 on: the part enters LPM3 when the angle stays within 4 LSBs for a minute |

The DIR pin sets the direction: tied to ground the angle increases with
clockwise rotation, tied to the supply with counterclockwise.  It must not
float.

## What the package contains

| Module | What is in it |
|---|---|
| `as5600` | The types `As5600`, `As5600Conf`, `As5600Gain` and `As5600Settings`, the trait `As5600Port`, the bus framing, the registers, the angle, the zero and the range, the CONF fields and the bring-up |

## How to choose an entry point

A program on a board uses `As5600`, built by `attach` with the board's bus,
its GPIO and the wiring, as a board's device table builds it.  A board that
wires DIR to a GPIO uses `attach_with_dir` and gives the pin.  The type
implements `AngleSensor`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)`, `attach_with_dir(bus, pins, w, pin)` | the part at `w.addr`, 0x36; nothing is sent |
| `begin()` | makes the bus ready, drives a DIR pin low, reads STATUS: whether the part answered |
| `angle_read()` | `AngleSensor`: the angle after the zero and range, 0 to 4095, with the magnet state; the angle is 0 with no magnet detected |
| `angle_set_zero(count)` | `AngleSensor`: ZPOS = a RAW ANGLE count, 0 to 4095; volatile until the part is reset |
| `zero_here()` | ZPOS = the RAW ANGLE now |
| `raw_angle()`, `angle()`, `status()` | the registers as numbers |
| `gain()` | AGC and MAGNITUDE in one transaction |
| `set_zpos(n)`, `set_mpos(n)`, `set_mang(n)`, `set_range(zpos, mpos)` | ZPOS, MPOS and MANG; each is read back; MANG is 0 or at least 205 and a range is at least 205 counts |
| `settings()`, `burns()` | ZMCO, ZPOS, MPOS, MANG and CONF in one transaction; ZMCO alone |
| `conf()`, `set_conf(c)` | the CONF fields; a write keeps the factory's reserved bits and is read back |
| `set_power_mode(n)`, `set_hysteresis(n)`, `set_output_stage(n)`, `set_pwm_freq(n)`, `set_slow_filter(n)`, `set_fast_filter(n)`, `set_watchdog(on)` | one field of CONF, by read, change and write |
| `set_clockwise(yes)` | drives the DIR pin; `false` when the board gave none |

A program that holds its own bus, or a test with a double of the part,
implements `As5600Port` over its bus and calls the module's functions,
which take the port: `as5600.angle_read_of(port)`,
`as5600.range_write(port, zpos, mpos)`.  The framing is `i2c_read_into` and
`i2c_write`, generic over `I2cBus`.  The tables are `pwm_hz`,
`slow_filter_factor`, `fast_threshold_lsb` and `settling_us`, and the
packing is `conf_of` and `conf_bits`.

## The rules a user needs

1. Wait `POWER_UP_US`, 10 ms, after power-on before `begin`.
2. After a write to ZPOS, MPOS, MANG or CONF the new setting reaches the
   angle at least `SETTLE_US`, 1 ms, later.  The slow filter adds its
   settling time, 2.2, 1.1, 0.55 or 0.286 ms for 16x, 8x, 4x and 2x
   (`settling_us`).  The part computes a new angle every 150 µs
   (`SAMPLE_US`).
3. `angle_set_zero` takes a RAW ANGLE count, the frame `raw_angle()`
   reads, and not a count of the scaled angle.  To make the position the
   shaft is at the zero, call `zero_here()`.
4. A zero set this way is volatile: it holds until the part is reset or
   powered down.  A program that needs it again sets it at start.
5. The status and the angle are two transactions, in that order.  The
   magnet can move between them; a reading with the state `ok` is a
   reading of a magnet that was in range when the status was read.
6. A weak or strong magnet gives an angle that is less accurate and still
   an angle.  The reading carries both; a program that wants only good
   readings tests `r.magnet`.
7. A change to CONF reads the register first.  Bits 15 and 14 are the
   factory's, and a write that cleared them would change the part.
8. A narrowed range scales the output: with MANG = 2048 half a turn of the
   shaft spans all 4096 counts, and `dev_angle.angle_milli_deg` then
   overstates the angle by that factor.
9. `ANGLE` has a 10-LSB hysteresis at the limit of the 360-degree range, so
   the output does not toggle between 4095 and 0 at the wrap.

## What is not implemented

- **The BURN commands, on purpose.**  BURN (0xFF) writes the part's
  one-time-programmable memory.  0x80 burns ZPOS and MPOS, which the sheet
  allows three times and counts in ZMCO; 0x40 burns MANG and CONF once, and
  MANG only while ZMCO is 0.  A burn cannot be undone, the sheet asks for a
  10 µF capacitor on VDD3V3 and a magnet in view while it runs, and a
  program that sets the wrong zero and burns it has a part that keeps it.
  `burns()` reads ZMCO, and a program that must burn follows the sheet's
  Figure 24 and Figure 26 by hand.
- The OUT pin.  `set_output_stage` selects what the pin drives, analog or
  PWM; reading it is the program's, with an ADC or a capture timer.
- The AS5600L, which has the same registers at a programmable I2C address,
  0x40 by default.
- An async form: the part converts continuously, and `angle_read` does not
  wait.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the trait
  this driver implements, the wiring record and the register helpers it
  uses, and `angle_milli_deg` and `angle_diff` for the count.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus` and `GpioOut` traits and the board handles.

## Tests

`novo test` runs the module over a double of the part that answers on an
I2C bus at 0x36, with the registers the data sheet makes read only, the
12-bit values whose upper nibble is not part of the value, the CONF
register whose two top bits are the factory's, an ANGLE register that
follows the zero and the range, and an address pointer that goes back to
the high byte after the low byte of ANGLE, RAW ANGLE and MAGNITUDE.  It
checks the framing of every entry point, the CONF fields one at a time, the
refusals, and the readings through the trait.  The suite covers every line
of `src/`; `tests/coverage.sh` measures it.

The data sheet gives no numeric examples, and the suite's values are
computed from its register map and formulas.  The double's pointer rule is
this driver's reading of one paragraph of the sheet ("Automatic Increment
of the Address Pointer for ANGLE, RAW ANGLE and MAGNITUDE Registers"); it
has not been compared with a part.

**Hardware: NOT VERIFIED.**  See the top of this page.

The sources read for this driver, for their fixes and not ported, are the
data sheet above, the AS5600L data sheet v1-12 of 2020-May-14 for what the
two parts share (the watchdog's behaviour, which the v1-06 sheet gives only
as a detection time), and Zephyr's `drivers/sensor/ams/ams_as5600`.  Embassy and
esp-hal hold no angle-sensor driver.  Each fix kept is named in `src/as5600.nv` where it is kept:

- The status and the angle are read in two transactions, and each 12-bit
  value alone, because of the pointer rule.  The devices-nv reference
  double reads 0x0B to 0x0F in one, which on the pointer rule reads RAW
  ANGLE twice.
- Zephyr reads RAW ANGLE, which ignores the zero and the range a program
  set, and refuses a reading while the magnet is weak or strong.  This
  driver reads ANGLE and keeps a weak or strong magnet's angle with its
  state.
- The sheet says a new setting reaches the angle 1 ms after a register
  command, and a program that sets the zero and reads at once reads the
  old one.  Zephyr's driver writes no register and has no such wait;
  `SETTLE_US` names it here.
- The AS5600 sheet allows ZPOS and MPOS to be burned three times, and the
  AS5600L sheet two.  Neither matters here, since nothing burns.

## Licence

Apache-2.0.
