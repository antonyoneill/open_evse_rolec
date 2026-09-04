# AGENTS.md — open_evse_rolec

Guidance for AI agents (and humans) working in this repo.

## What this repo is

Fork of [OpenEVSE/open_evse](https://github.com/OpenEVSE/open_evse) (8.2.3
baseline, BarnyD Rolec compat) customised for the **Rolec ACSE0021** EVSE:

- ATmega328P @ 16MHz, no bootloader, flashed via USBasp ISP (5V)
- Rolec pin remap already present in the baseline (`open_evse.h`: CP=ADC1,
  PP=ADC0, RL1=PC2, RL2=PD5, LED red PB0 / green PD7 / blue PC3)
- Controlled over RAPI serial (9600 8N1) via an Elfin EW10A MQTT bridge into
  Home Assistant

## Build

```bash
uvx --from platformio pio run -e openevse_eu
# Output: .pio/build/openevse_eu/firmware.hex
```

**Only `openevse_eu` is supported.** Other upstream environments
(`openevse`, `emonevse`, ...) are not built or released from this fork —
the Rolec flag set lives only in that env's `build_src_flags`.

Flash (no fuse changes needed; EESAVE=1 means `-e` chip-erase wipes EEPROM):

```bash
sudo avrdude -c usbasp -p m328p -e -U flash:w:firmware.hex
```

## CI / releases

- **PR validation:** `.github/workflows/build.yml` — builds `openevse_eu`,
  uploads firmware.hex/.elf as an artifact.
- **Releases:** push a semver tag `v*.*.*` → `.github/workflows/release.yml`
  builds and attaches
  `firmware-openevse_eu-ACSE0021-v<X.Y.Z>.hex` (+ `.elf` + sha256) to a
  GitHub Release.

## Versioning (SemVer)

- `MAJOR` — incompatible behaviour change (pin remap changes, RAPI command
  removal, flash layout/EEPROM format change)
- `MINOR` — new RAPI command or new observable behaviour
  (e.g. `$LN` LED override, PP plug-detect while disabled)
- `PATCH` — bugfix with no new surface (colour tweaks, timing fixes)

Tags are created manually; the release workflow does the rest. First
release: `v1.0.0`.

## Branch & PR conventions

- Default branch: `main`
- One PR per logical feature, stacked where dependent (PR bodies state what
  they stack on)
- Never commit directly to `main`

## Key files

| Path | Purpose |
|---|---|
| `platformio.ini` | Build flags per env. Rolec-specific flags are documented inline — OEV6/RELAY_PWM/MENNEKES_LOCK/BOOTLOCK are intentionally omitted, read the comments before re-enabling |
| `firmware/open_evse/open_evse.h` | Pin map, feature defines (`KWH_RECORDING`, `VOLTMETER`, `AMMETER` are on) |
| `firmware/open_evse/rapi_proc.cpp` | RAPI command handling (`$GS`, `$GG`, `$GE`, `$GU`, `$GV`, `$LN`, ...) |
| `firmware/open_evse/J1772EvseController.cpp` | State machine (`Update()`), pilot/PP sampling |
| `firmware/open_evse/EnergyMeter.cpp` | Session/lifetime kWh (EEPROM-backed) |

## RAPI quick reference (this fork)

```
$GS  -> "%02x %ld %02x %04x"  state, elapsed_min, pilot, VOLATILE vflags
$GG  -> "%ld %ld"             charging current (mA), voltage (mV)  # NOT ground check
$GE  -> "%d %04x"             current capacity (A), persistent flags
$GU  -> "%lu %lu"             session watt-seconds, lifetime Wh
$GN  -> "%u %u %u"            LED override mode, rgb bits, period_ms (see $LN)
$GV  -> "<ver> <rapi_ver>"    version
$FD / $FE                   disable / enable
$LN 0|1|2|3 [r g b [ms]]    LED override (0=normal, 1=flash b-g, 2=solid blue, 3=manual RGB)
```

Volatile vflag `ECVF_EV_CONNECTED` (0x0100) stays live while DISABLED —
derived from passive PP pin sampling, not the pilot (pilot sits at -12V).

## Gotchas

- `EvseController::Update()` early-returns while DISABLED; anything relying
  on per-loop pilot/PP sampling must be placed before that return.
- LED green shares PD7 with the RL2 driver — only one colour at a time.
- `$LN` override is intentionally skipped during EVSE_STATE_C (charging) so
  the user always sees real charging state.
- The Elfin bridge sends raw `$CMD\r\n` (no checksum); the firmware parser
  accepts it, replies carry XOR checksums.
- PP plug-detect also runs during -12V *fault* states (e.g. GFI trips), not
  just DISABLED: `ReadPilot()`'s N12 branch applies to any -12V pilot, so
  `$GS` reports live plug presence during faults too. Considered desirable
  (rescue crews / HA can see if the car is still attached). Sampling is
  rate-limited to 2 Hz in `ReadPilot()` — do not "optimise" it back to
  every-pass; `AdcPin::read()` busy-waits ~112µs/sample (10x avg = ~1.1ms).
- `AdcPin::read()` mutates global ADMUX/ADCSRB with no interrupt protection.
  Safe today because no ISR reads the ADC — if you ever add one, guard the
  reads with cli()/sei() or the mux channel will corrupt mid-conversion.
