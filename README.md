# ina3221-nv

A driver for the INA3221, a three-channel current and voltage monitor from
Texas Instruments with an I2C interface and four alert pins, specified in
the data sheet "INA3221 26V, Triple Channel, 13-Bit, I2C Output Current and
Voltage Monitor With Alerts", document SBOS576C, May 2012, revised
September 2026.  The driver implements the `CurrentMonitor` and
`Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv): the bus voltage,
the shunt voltage, the current and the power of each of three rails, in
integer units, and the manufacturer and die ID check.

**Hardware: NOT VERIFIED.**  No board carries an INA3221 yet.  The device
program below builds for the nRF52840 DK and the nRF54LM20 DK.  The
driver's functions run over a double of the part that follows the data
sheet's register map, its sign and step rules and its bus protocol.  A
reading from a part on a board has not been taken.

## Install

```sh
novo pkg add ina3221-nv
```

The device program is `examples/rail_monitor.nv`.  Make it the `main` of a
package that depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
```

## The device program

`examples/rail_monitor.nv` reads a breakout over I2C once a second and
prints the bus voltage in millivolts, the current in microamps and the
power in microwatts of two rails over RTT:

```
ina3221 ch=0 mV=5012 uA=41250 uW=206745
ina3221 ch=1 mV=3301 uA=12800 uW=42253
```

The breakout's SDA and SCL go to the board's I2C pins and its A0 pin to
ground, which puts the part at 0x40.  Two loads go through the breakout's
shunts, 0.1 ohm resistors, on channels 1 and 2 (numbered 0 and 1 in the
driver).  The program brings the part up, takes the two shunts into use,
and reads.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use dev_current.{ CurrentMonitor }
use ina3221.{ Ina3221 }

// The part's address with A0 at ground.
const ADDR = 0x40

@tier(embedded)
fn main() [hw]
    // Both shunts are 0.1 ohm: 100000 micro-ohms.  The third channel has none.
    let part = Ina3221.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR),
                              100000, 100000, 0)
    if not part.begin()
        hal.uart.write("ina3221: no answer\n")
        return
    // 800 mA at most: 80 mV across each shunt.
    if not part.mon_set_shunt(0, 100000, 800000) or not part.mon_set_shunt(1, 100000, 800000)
        hal.uart.write("ina3221: shunt refused\n")
        return
    loop
        match part.mon_read(0)
            Some(r) =>
                hal.uart.write("ina3221 mV=")
                hal.uart.write(str.from_int(r.bus.mv))
                hal.uart.write(" uA=")
                hal.uart.write(str.from_int(r.current.ua))
                hal.uart.write("\n")
            None => hal.uart.write("ina3221: no result\n")
        hal.timer.delay_ms(1000)
```

`begin` reads the manufacturer ID and the die ID, resets the part, and
checks that CONFIG holds its power-on value.  It answers `false` for a part
that does not answer, for one that is not an INA3221, and for one that does
not reset.  `mon_set_shunt` checks the resistance and the largest current
and enables the channel.  The full program, which also leaves the unused
third channel out of the measurement pass and waits the time `mon_start`
answers, is `examples/rail_monitor.nv`.  A script builds it for the boards
below by making it the `main` of a scratch package that depends on this one.
It also runs the emulated board's image under QEMU, where no part answers,
and checks that the program reports it.

```sh
bash tests/device_build.sh
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 17,672 B | 6,606 B | 3,252 B |
| `nrf54lm20-dk` | 18,656 B | 6,778 B | 3,236 B |

The numbers are from the build's memory report.  Flash also holds the
runtime, the compiler's support library and the program's tables.  The
arena, 65,536 B on the Nordic boards, is the runtime's heap and is reserved
whether or not the program uses it.

The Nordic DKs drive I2C with a TWIM (the nRF52840 DK's TWIM0 on SDA P0.26
and SCL P0.27, the nRF54LM20 DK's TWIM22 on SDA P1.03 and SCL P1.04), so
both rows are the image with the driver's reading path in it.  With no part
on the bus the program prints `ina3221: no answer`.  A reply from the part
over a DK's bus is NOT VERIFIED until the driver rig carries one.

### The effects

Every function of the driver that reaches the bus or the Critical pin
declares `[hw]`, and nothing else.  The register conversions, the limit
arithmetic and the tables are pure.  The driver performs no input or output
of its own and builds no list of its own.  The bus functions of devices-nv
build one short frame per transaction inside a `system` block and release it
when the call returns.  The package checks at the embedded tier, so a
program of any tier may call it.  The program above declares `[hw]` for the
same reason.  The one module that declares more, `ina3221_async`, is below.

## What it is

A current monitor sits on a supply rail.  A shunt resistor of a fraction of
an ohm is in series with the load, and the monitor measures the small
voltage across it, which Ohm's law turns into a current.  It also measures
the voltage at its IN- pin, the bus voltage.  The INA3221 does this for three
rails with one ADC, one channel after another.

Unlike the INA226, the INA3221 has no calibration register and computes
neither a current nor a power: it reports two voltages per channel.  The
driver multiplies by the resistances it was given at `attach` to answer the
trait's current and power.  A shunt resistance that is not positive marks a
channel the board does not use, and `mon_read` answers `None` for it.

| Register | Address | Step | What it holds |
|---|---|---|---|
| CONFIG | 0x00 | | reset, channel enables, averaging, conversion times, mode; power-on value 0x7127 |
| SHUNT 1 to 3 | 0x01, 0x03, 0x05 | 40 µV | signed, 13 bits in bits 15 to 3; full scale 163.8 mV |
| BUS 1 to 3 | 0x02, 0x04, 0x06 | 8 mV | the same layout; full scale 32.76 V, the pins take 26 V |
| CRITICAL limit 1 to 3 | 0x07, 0x09, 0x0B | 40 µV | compared with each shunt conversion |
| WARNING limit 1 to 3 | 0x08, 0x0A, 0x0C | 40 µV | compared with the averaged shunt voltage |
| SUM | 0x0D | 40 µV | the shunt voltages of the channels the alert configuration picks, signed, bits 15 to 1 |
| SUM LIMIT | 0x0E | 40 µV | compared with SUM; power-on 0x7FFE |
| MASK/ENABLE | 0x0F | | summation channels, latch bits, the alert flags; power-on 0x0002 |
| PV UPPER, LOWER | 0x10, 0x11 | 8 mV | the power-valid limits, 10 V and 9 V at power-on |
| MANUFACTURER ID | 0xFE | | 0x5449 |
| DIE ID | 0xFF | | 0x3220 |

The sheet's example of a shunt voltage is -80 mV, 2000 steps of 40 µV, which
is 0xC180.  A 5 V rail through a 0.1 ohm shunt carrying 50 mA drops 5 mV,
125 steps, and the SHUNT register holds 125 << 3 = 0x03E8; the BUS register
holds 625 << 3 = 0x1388; the driver reads 5000 µV, 5000 mV, 50000 µA and
250000 µW.

CONFIG holds five fields.

| Field | Bits | Values |
|---|---|---|
| channel enables | 14, 13, 12 | channel 1, 2 and 3 |
| AVG | 11:9 | 1, 4, 16, 64, 128, 256, 512 or 1024 averages |
| VBUSCT | 8:6 | 140, 204, 332, 588, 1100, 2116, 4156 or 8244 µs |
| VSHCT | 5:3 | the same times |
| MODE | 2:0 | 0 and 4 power-down; 1 shunt, 2 bus, 3 both, triggered; 5, 6 and 7 the same, continuous |

A pass over the enabled channels takes the channels times the averages
times the conversion times of the signals the mode measures: 6.6 ms at the
power-on values.  A write to CONFIG halts a conversion in progress and
starts one from the new settings.  A triggered mode converts every enabled
channel once and then powers the part down; the registers keep the last
results.

The part has four alert pins.  Critical goes active when one shunt
conversion exceeds its channel's critical limit or the sum exceeds the sum
limit; Warning when an averaged shunt voltage exceeds its warning limit;
Power Valid (PV) goes high when every enabled bus voltage is above the upper
limit and low again when one falls below the lower limit; Timing Control
(TC) goes low if channel 2 does not rise above 1.2 V within a fixed time
after channel 1 does, at power-up.  The pins are open-drain.

## What the package contains

| Module | What is in it |
|---|---|
| `ina3221` | The types `Ina3221`, `Ina3221Config`, `Ina3221Flags` and `Ina3221Alert`, the trait `Ina3221Port`, the bus framing, the registers, the identity, the configuration, the readings, the alerts and the bring-up |
| `ina3221_async` | `Ina3221Timed`, the awaitable form of the measurement, for a host scheduler |

## How to choose an entry point

A program on a board uses `Ina3221`, built by `attach` with the board's bus,
its GPIO, the wiring and the shunt resistance of each channel in
micro-ohms.  The resistances are facts about the board, as the address is,
and the driver keeps them.  The wiring's `irq` is the Critical pin.  The
type implements `CurrentMonitor` and `Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w, r0, r1, r2)` | the part at `w.addr`; nothing is sent |
| `begin()` | makes the bus ready, checks the IDs, resets, checks CONFIG |
| `mon_set_shunt(ch, shunt, max_ua)` | `CurrentMonitor`: enables the channel; refuses a shunt other than the board's, a largest current that drops more than 163.8 mV, and a channel other than 0 to 2 |
| `mon_read(ch)` | the bus voltage, shunt voltage, current and power of an enabled channel |
| `mon_bus_voltage(ch)` | the bus voltage alone, one transaction |
| `mon_start()` | 0 in a continuous mode; otherwise a triggered conversion of the enabled channels and its time in microseconds; −1 when no channel is enabled or the part did not answer |
| `device_id()` | `Identifiable`: the die ID, 0x3220, no revision |
| `shunt_voltage(ch)`, `sum()` | the shunt voltage of a channel; the sum register |
| `flags()`, `ready()` | the MASK/ENABLE flags; the Conversion Ready flag. Both clear it |
| `config()`, `set_config(c)`, `set_channel(ch, on)`, `set_averaging(n)`, `set_conversion_times(b, s)`, `set_mode(n)` | the CONFIG fields; each write is read back |
| `set_critical(ch, uv)`, `set_warning(ch, uv)`, `critical(ch)`, `warning(ch)` | the alert limits in microvolts across the shunt |
| `set_sum_limit(uv)`, `set_alert(a)` | the sum limit; the summed channels and the latch bits |
| `set_power_valid(upper_mv, lower_mv)`, `power_valid_upper()`, `power_valid_lower()` | the power-valid limits in millivolts |
| `alert_pin()` | the level of the Critical pin, −1 for a wiring with none |
| `reg(r)` | any register |

A program that holds its own bus, or a test with a double of the part,
implements `Ina3221Port` over its bus and calls the module's functions,
which take the port: `ina3221.reading_of(port, ch, shunt)`,
`ina3221.critical_write(port, ch, uv)`.  The framing is `i2c_read_into` and
`i2c_write`, generic over `I2cBus`.  The conversions are `shunt_uv_of`,
`bus_mv_of`, `sum_uv_of` and `flags_of`, which need no bus.

### The awaitable form

`ina3221_async` has `Ina3221Timed`, which implements `CurrentMonitorAsync`:
`mon_measure(ch)` starts a triggered conversion, parks the task for its
length with `core.rt.sleep_ms`, and answers the channel's measurement.  It
holds a port and the three resistances, so it is built over any
`Ina3221Port`, `Ina3221Timed { port: ..., shunt0: ..., shunt1: ..., shunt2: ... }`.
An `async fn` cannot take a `@value` struct (E2015), so the type is a plain
struct and one arena cell; it is for a host scheduler and its method is
`@tier(app)`.  A device program does not call it: `async fn` is refused at
the embedded tier, and the program waits the time `mon_start` answers.  The
part has no pin that signals a finished conversion, so there is no
`AsyncDeviceReady` form either.

## The rules a user needs

1. Call `begin` before the first `mon_read`.  `begin` resets the part, which
   enables all three channels and sets continuous conversion of the shunt
   and bus voltages.
2. `mon_set_shunt` takes the resistance the board was attached with.  A
   different one would make every current and power wrong, and is refused.
   The part has no calibration register, so the call writes nothing but the
   channel's enable bit.
3. `mon_read` answers `None` for a disabled channel, since a disabled
   channel is not measured and its registers hold the result of its last
   conversion.  `mon_bus_voltage` does not check, so it answers that old
   value.
4. A reading is three transactions, one register each: CONFIG for the
   enable, the SHUNT register and the BUS register.  The sheet defines a
   read as the pointer write and one register, and promises no read that
   runs on into the next.  In a continuous mode a conversion can end between
   two of them.
5. The part reports a saturated shunt ADC as a register at its positive or
   negative full scale, 0x7FF8 or 0x8000, which cannot be told from a
   voltage past it.  The reading sets `overflow` and answers a current and a
   power of 0.
6. A current is positive from the supply side of the shunt to the load side.
   A bus voltage is positive.  A negative shunt voltage, a battery that is
   charging, is signed correctly.
7. Reading MASK/ENABLE clears the critical, summation and warning flags and
   Conversion Ready.  Writing it does not.  The power-valid and
   timing-control flags follow their pins.  `flags()` returns the flags it
   read; a program that latches alerts reads it, and `ready()` clears the
   flags too.
8. The critical and warning limits, the sum limit and the power-valid limits
   are values the register can hold: a limit outside its range is refused
   without a transaction.  A limit is in the register's steps rounded toward
   zero.
9. The warning alert compares the averaged shunt voltage and the critical
   alert compares each conversion.  Choose the averaging for the first and
   read the second as a fast over-current trip.
10. The sum alert adds the shunt voltages of the channels `set_alert` picks.
    The sheet says the sum is meaningful only when the shunts have the same
    resistance.
11. A triggered conversion's time is the enabled channels times the averages
    times the conversion times.  With no channel enabled `mon_start` answers
    −1.
12. The power-valid alert needs the bus voltages measured, so a mode that
    includes the bus voltage, and it watches every enabled channel.  If one
    channel is unused the sheet says to tie its IN- pin to a used channel
    for the PV function.  A software reset restores 10 V and 9 V.
13. The Timing Control function runs at power-up and after a reset only; a
    write to CONFIG before it finishes disables it until the next reset.
14. The alert pins need a pull-up on the board.

## What is not included

- The SMBus alert response and the high-speed (3.4 MHz) mode of the bus.
  NOT IMPLEMENTED.
- The Timing Control and Power Valid pins as awaitable edges.  The wiring
  carries one pin, the Critical pin; a program that uses the others waits for
  them with the board's `GpioInAsync`.  NOT IMPLEMENTED.
- A current limit in microamps: the limits are in microvolts across the
  shunt, since the part compares voltages.  A program converts with the
  resistance.
- An averaged wait: a program that wants to wait for a result polls
  `ready()` or waits `mon_start`'s time.
- The INA219 and the INA226, which are other parts: `i2c-devices-nv` drives
  the first, `ina226-nv` the
  second.

## Reference drivers read

Each of these was read for its fixes, and none was ported.  The fix is named
at its site in `src/ina3221.nv`.

| Driver | What was taken from it |
|---|---|
| Zephyr `drivers/sensor/ti/ina3221` | the triggered conversion's time as the enabled channels times the averages times the conversion time, and the Conversion Ready poll; the register is shifted as an unsigned number, so a negative shunt voltage comes out positive, which this driver does not do; it never waits after the reset |
| `ina3221` crate 0.4.5 | the register addresses and the 40 µV and 8 mV steps; its `set_channels_enabled` masks CONFIG with 0xFFF8, which powers the part down and drops the averaging and the conversion times, which this driver does not do; its alert flag constants number the warning, critical and summation channels the other way from the sheet's Table 8-33 |

Embassy and esp-hal hold no INA3221 driver.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the traits
  this driver implements, the wiring record and the register helpers it
  uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus` and `GpioOut` traits and the board handles.
- `ina226-nv` drives the one-channel
  INA226, with a calibration register.

## Tests

`novo test` runs the module over a double of the part that answers on an
I2C bus at 0x40, with the power-on values, the read-only registers, limit
registers that keep the bits the sheet gives them, a MASK/ENABLE register
whose flags clear when it is read, and a read that answers one register and
nothing longer.  It checks the framing of every entry point, the
configuration, the readings of each channel, the alerts, the identity and
the bring-up with its refusals, and the awaitable form.  The suite covers
every line of `src/`; `tests/coverage.sh` measures it.

The conversions are checked against the data sheet's example (-80 mV is
0xC180) and against a worked rail: 5 V through 0.1 ohm at 50 mA.  The data
sheet gives no numeric example of an alert; the limit values in the suite
are computed from the register steps.

**Hardware: NOT VERIFIED.**  See the top of this page.

## Licence

Apache-2.0.
