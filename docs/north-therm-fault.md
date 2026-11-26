# The north_therm hardware fault

## Summary

This report describes one Pixel 8 Pro (`husky`) in August and September 2026. Times are local.

The `north_therm` sensor path read too cold at rest. In the recorded events, it also gave false readings of 55 to 58 °C for less than 7 seconds. The thermal software treated these readings as real heat. This caused "phone is too warm" notices, short stutters, and thermal shutdowns. The [patched build](../modules/norththermrewire.py) replaces `north_therm` with `cam_therm` in the affected formulas. It does not repair the sensor.

## Symptoms

- From 15 September, System UI showed the `high_temp` notice ("phone is too warm") about 50 times per day. The phone was idle and cool.
- Sometimes a short stutter came with the notice. It never became a freeze.
- On 15 September the phone ran GrapheneOS build 2026091001, security patch level 2026-09-01. No root.
- Thermal shutdowns started on 16 September. The phone recorded five on 21 September.

## The sensor on the original build

| Item | Value |
|---|---|
| Name | `north_therm` |
| Kernel zone | `thermal_zone16` |
| Hardware | an NTC thermistor on the ADC of the S2MPG15 sub-PMIC (platform device `s2mpg15-spmic-thermal`) |
| Location | top of the phone, camera-bar region (inference, see below) |
| Kernel trips | one passive trip at 125 °C. No critical trip. |

**Location evidence.** In the stock thermal model examined, `north_therm` had the largest weight (0.14) in `VIRTUAL-BTS-WINDOW-PARTIAL`. This formula covers the body-temperature IR sensor in the rear camera bar. `north_therm` had almost no weight in the speaker formulas.

**Possible placement.** The Google Service BOM for the Pixel 8 Pro (V4.2) lists an ANT1 sub-PCB (part G949-00706-01). It also lists a ZIF cable from ANT1 to the logic board ("DIECUT, KAPTON, ANT1PCB ZIF"). The iFixit logic-board guide shows a press connector in the top-left corner. These clues suggest the ANT1 sub-PCB as a possible location. The investigation did not verify the physical location.

## How a false reading causes a notice or a shutdown

In the stock build used for the investigation, `/vendor/etc/thermal_info_config.json` defined these formulas:

```
VIRTUAL-SKIN-SUB-4 = 0.9 × north_therm + 0.05 × soc_therm + 0.01 × neutral_therm + 0.73
VIRTUAL-SKIN       = MAX(VIRTUAL-SKIN-SUB-0 … SUB-10)
```

The formulas reproduced the logged values. In one capture, `north_therm` reached 54.94 °C and `VIRTUAL-SKIN` reached 52.56 °C. `VIRTUAL-SKIN` had the type SKIN. The investigation linked these thresholds to the observed thermal status:

| VIRTUAL-SKIN | Status | Effect |
|---|---|---|
| 39 °C | 1 | light throttling |
| 52 °C | 5, EMERGENCY | "too warm" notice. CPU clusters 1 and 2 limited to step 14, GPU to 11, TPU to 7, charging `chg_mdis` 26 |
| 55 °C | 6, SHUTDOWN | orderly shutdown by the framework |

A spike of `north_therm` to about 55 °C pushed `VIRTUAL-SKIN` to about 53 °C: status 5, the notice, and a short throttle. At about 57 °C or more, `VIRTUAL-SKIN` passed 55 °C and the framework shut the phone down. The board was at about 33 to 35 °C during these events.

## Evidence

**Captures on 15 September.**
- A 1-second poll of `dumpsys thermalservice` showed `north_therm` at 54.9 °C. All other sensors were at their cool baseline: soc 36.8, cam 32.1, usb 32.8, battery 32.1 °C.
- The other 8 thermistors were steady. Their largest jump between readings was 1.6 °C. The jump of `north_therm` was 37 °C.
- The logs showed no kernel or ADC read errors. The readings were well formed but inconsistent with the other sensors.
- Three of nine recorded spikes occurred with the screen off.
- The HAL zone notice preceded the PowerUI event by about 70 ms. The logs recorded about eight notices per hour during that capture. This count does not include spikes that did not cause a notice.

**Software clues.** The GrapheneOS release notes reviewed at the time did not list changes to the thermal configuration, HAL, or sensor driver. The investigation found no matching reports on the tracker or forum. These observations favor a hardware fault but do not rule out every software cause.

**Heat and spike rate: warm-up test (15 September, 22:40 to 22:50).**
- An 8-thread load ran for 5 minutes. `soc_therm` reached 51 to 53 °C.
- There were zero spikes during the test and 30 minutes with no spike overall.
- This short test did not reproduce a spike. It did not establish how heat or workload affects spike frequency.
- `north_therm` rose from 15.5 to 19.9 °C as the board warmed. The sensor path still responded to heat.

**Heat and spike height.** The spike height increased with the resting temperature of the sensor:

| Rest reading of north_therm | Spike |
|---|---|
| 16.2 °C | 54.9 °C |
| 17.5 °C | 55.2 °C |
| about 19.7 °C | about 56.4 °C (VIRTUAL-SKIN 53.95 °C at 22:56 on 15 September, about 1 °C below shutdown) |

The sample was small, but higher baseline readings coincided with higher spikes.

**Later readings.**
- On 18 September the board was warm (soc 47, battery 38 °C, charging). `north_therm` was stable at about 26 °C and tracked the board, but read about 11 °C below the adjacent `cam_therm` (37 °C). There was no shutdown that day.
- By 21 September the cold offset had grown to about 16 °C.

**Withdrawn claims.**
- "All spikes go straight to status 5, so the fault is bistable." Withdrawn: PowerUI does not log statuses 1 to 4. A real status-1 event produced no log line.
- "The USB drops on the evening of 15 September show a bad port." Withdrawn: the USB configuration could cut the link when the phone locked. On 21 September, USB also disconnected when the phone moved while attached to a cable.
- "Humidity triggers the spikes." Demoted: 15 September was cold and dry, and the humid day (Sunday, 13 September) had no spikes.

## Shutdown history

| Date | Events |
|---|---|
| 15 September | 4 restarts, none thermal |
| 16 September | first thermal shutdown. The framework shut down at 08:50:49, and the bootloader recorded `shutdown,thermal` at 08:51:24. The battery read 35.2 °C 78 seconds before. A second thermal shutdown occurred at about 21:15. |
| 17 September | 1 thermal shutdown at about 16:50 |
| 18 September | no shutdown |
| 21 September | 5 thermal shutdowns, including 17:43, 17:47 and 19:04 |

The 16 September event was an orderly framework shutdown of a phone at about 35 °C. Implied `north_therm` spike: 57.7 °C or more. After the reboot, the phone showed the notice "Phone turned off due to heat".

## Repair history and suspected cause

**History.** On 7 August the phone fell into a sink for about 5 seconds. It was then charged with the USB-C port still wet. On each boot it showed an overheating message and shut down at the lock screen. A shop repaired the charging circuit with board-level micro-soldering. The nearby sensors (`usb_pwr_therm` and `charge_therm`) gave stable readings during the later investigation.

**Working hypothesis (21 September): electrochemical migration.** The failure after charging with a wet port makes corrosion or contamination a plausible cause. Added series resistance can explain the cold offset. Intermittent leakage can explain the hot spikes. A contaminated connection can produce both effects. The investigation did not locate or inspect the fault.

The readings still tracked changes in temperature. This indicates that the sensor and converter could still respond. A physical inspection can help distinguish corrosion, contamination, and other connection faults.

## Soak test after the patch

On 21 September 2026, `north_therm` read 54.82 °C for two samples. `VIRTUAL-SKIN` stayed at 36.2 °C with status 0. A similar spike on the stock build had produced status 5. The test showed that the patched formula did not turn this false spike into a heat warning.

## Remaining risks after the patch

- The patched `VIRTUAL-SKIN-SUB-4` relies more on `cam_therm`. An inaccurate reading from that sensor can cause a false warning or hide real heat.
- `cam_therm` measures a different point. The soak test did not show that the new formula detects every real hot spot.
- The stock kernel configuration examined for this report gave `quiet_therm` a critical trip at 57 °C. The vendor JSON patch does not change kernel trips.

## How to diagnose it again

Do not assume that a new heat warning is false. If the phone feels hot, stop using and charging it before you investigate.

If the phone is cool and the symptoms recur:

1. Connect the phone over ADB.
2. Poll `adb shell dumpsys thermalservice` about once per second. Read the section "Current temperatures from HAL".
3. Compare `north_therm`, `cam_therm`, and `quiet_therm` with the other sensors.
4. Read the boot reasons with `adb shell getprop persist.sys.boot.reason.history`. Look for `shutdown,thermal`.

On the build used for the investigation, `logcat` showed `pixel-thermal` raw data during throttling. SELinux blocked unprivileged reads of thermal sysfs files and `dmesg`.
