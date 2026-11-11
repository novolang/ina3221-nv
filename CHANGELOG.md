# Changelog

## 0.1.0

The first release: a driver for Texas Instruments' INA3221 three-channel
current and voltage monitor on an I2C bus.

- `Ina3221`, the type a board's device table builds with the shunt
  resistance of each of the three channels, implementing devices-nv's
  `CurrentMonitor` for three channels and `Identifiable`, with its own
  methods for the configuration, the flags, the alert limits and register
  access.
- The register access as functions over `Ina3221Port`, a trait that a type
  holding a bus implements, and the framing as functions over `I2cBus`.
  Each register is its own transaction.
- The manufacturer and die ID check: 0x5449 and 0x3220.  The bring-up resets
  the part and checks CONFIG.
- CONFIG as five fields (channel enables, averaging, bus and shunt
  conversion times, mode), each changed with the others kept, the time of a
  pass over the enabled channels, and a triggered conversion that answers
  its time and does nothing in a continuous mode.
- The shunt voltage (40 µV a step) and bus voltage (8 mV a step) of each
  channel, sign-extended from the upper 13 bits, and the current and power
  from the resistance given at `attach`; a saturated shunt register sets
  `overflow`.
- The critical and warning limits of each channel, the shunt-voltage sum and
  its limit, the summed channels and the latch bits, the alert flags with
  the channel numbering of the sheet's Table 8-33, and the power-valid
  limits.
- `ina3221_async.Ina3221Timed`, an implementation of `CurrentMonitorAsync`
  for a host scheduler.
- A device program, `examples/rail_monitor.nv`, and `tests/device_build.sh`,
  which builds it for the nRF52840 DK and the nRF54LM20 DK and runs the
  emulated board's image under QEMU.
- The suite runs over a double of the part and covers every line of `src/`;
  `tests/coverage.sh` measures it.

### Known

- Hardware verification: NOT VERIFIED.  No board carries an INA3221.
- The data sheet read is SBOS576C, May 2012, revised September 2026.  It
  gives one numeric example, a shunt voltage of -80 mV as 0xC180, which the
  suite uses, and none of the alerts.
- A reading is three transactions because the data sheet defines no read
  across registers.  A conversion can end between two of them in a
  continuous mode.
- `mon_set_shunt` accepts only the resistance the board was attached with,
  and writes nothing but the channel's enable bit: the part has no
  calibration.  `mon_read` answers `None` for a disabled channel.
- The SMBus alert response, the high-speed mode and the Power Valid and
  Timing Control pins as awaitable edges are not implemented.
- `Ina3221Timed` is a plain struct because an `async fn` cannot take a
  `@value` struct (E2015); an `async fn` is refused at the embedded tier, so
  it is for a host scheduler.
- The device program's images are 17,672 and 18,656 bytes of flash for the
  nRF52840 DK and the nRF54LM20 DK.  A reply from the part over either bus is
  NOT VERIFIED.
