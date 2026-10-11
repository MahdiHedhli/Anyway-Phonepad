# PhonePad Windows HID investigation, 2026-10-11

## Result and scope

Read-only investigation of the owner's PhonePad attached by USB-C to MHGPDW2M, with Bluetooth already paired. **A repeatable Contact Count Maximum feature-report mismatch was observed through the Windows HID API.** This is a concrete compatibility defect in the currently exposed interface, but is NOT yet proven to be the cause of missing touchscreen input. No successful physical touch input or screen-blanking fix was demonstrated.

The tests used the existing Microsoft HID stack, Python's standard-library ctypes, and Windows HidP/HidD APIs. No packages or drivers were installed, no firmware or settings were changed, and no input was injected. All HidP_SetUsageValue/SetUsages calls constructed test reports in LOCAL MEMORY ONLY. The only device report queries were four GET_FEATURE calls for the already advertised Contact Count Maximum report. A separate shared ReadFile input capture was bounded to 10 seconds.

## Direct observations from this PhonePad

The device remained enumerated as USB Input Device / HID-compliant touch screen, VID:PID **0525:B4B1**, status OK. Earlier device-property inspection recorded Microsoft input.inf / HidUsb and a bus description of HID Gadget. Chip identity and internal touch-controller topology remain unknown.

### Windows-parsed metadata

Captured 2026-10-11 22:55:07 UTC using a zero-access shared metadata handle and HidD_GetPreparsedData / HidP_GetCaps / HidP_GetValueCaps / HidP_GetButtonCaps / HidP_GetLinkCollectionNodes.

| Field | Observed value |
| --- | --- |
| Top-level collection | Digitizer / Touch Screen, page 0x0D / usage 0x04 |
| Input report | ID 1, 32 bytes including report ID |
| Finger collections | Six logical Finger collections, each containing a physical child collection for X/Y |
| Per-finger fields | Tip Switch, In Range, 4-bit Contact Identifier, 16-bit X, 16-bit Y |
| Contact IDs | Logical range 0..15 |
| X and Y | Absolute, logical range 0..4096 |
| Contact Count | Report 1, top-level, 8 bits, logical range 0..4096 |
| Feature report | ID 2, 2 bytes including report ID |
| Feature usage | Contact Count Maximum, page 0x0D / usage 0x55, 8 bits, logical range 0..10 |
| Feature main-item flags | 0x0003, including the Constant bit |
| Output reports | None advertised |
| Input capability counts | 12 button caps, 19 value caps, 31 data indices |
| Collections | 13 total: top-level + six Finger + six physical children |

This is **parsed metadata, not a dump of the raw HID report descriptor**. The declarations do not prove that usable contact reports are being generated.

### Reproduced feature-report failure

Four calls to `HidD_GetFeature` requested feature report ID **0x02** with the required **2-byte** buffer. The data byte was initialized differently to distinguish a returned value from untouched memory.

| Input buffer | API returned success | Returned buffer |
| --- | --- | --- |
| 02 A5 | true | 16 03 |
| 02 5A | true | 16 03 |
| 02 00 | true | 16 03 |
| 02 FF | true | 16 03 |

The first query completed in 0.078 seconds. Repeats occurred at approximately 22:56:43 UTC. The screen's awake/asleep state was not physically observed during these queries.

Windows' own `HidP_GetUsageValue(HidP_Feature, 0x0D, 0, 0x55, ...)` rejected the returned `16 03` buffer with **0xC011000A / HIDP_STATUS_INCOMPATIBLE_REPORT_ID**. The same parser, same preparsed data, and same usage accepted LOCAL synthetic `02 0A` as value 10 and `02 00` as value 0. Those synthetic buffers were never sent to the device.

Therefore **3 is not a verified maximum-contact count**: the byte occurs in an incompatible report. Likewise the descriptor's logical maximum of 10 is not a successfully retrieved device capability.

This establishes what the Windows API returned; there is no USB bus capture yet to locate the fault conclusively in the device firmware versus another layer of the active USB/HID path. A separate C# cross-check produced no result and was terminated; it is not counted as independent confirmation.

### Other metadata concerns

1. **Contact Count has an unrepresentable advertised maximum:** 4096 cannot fit in an unsigned 8-bit field (maximum 255). This could reflect a global-item initialization mistake, but raw descriptor bytes are needed to establish how it was encoded. Windows' generic parser still allowed construction/extraction of a small valid count in local tests, so this inconsistency alone does not prove that the touchscreen stack rejects the device.
2. **X/Y physical metadata needs review:** all parsed axes have PhysicalMin=0, PhysicalMax=0, Units=0x33 and UnitExponent nibble 0xE (-2). This does not supply a meaningful measured panel size. Do not assume that zero physical limits alone prove an error: HID default/fallback semantics must be checked against the raw descriptor. The unusual units and inherited units on count fields are additional review targets.
3. **The maximum-contact feature is marked Constant.** Record and compare this with a conformant implementation; no causal claim is made from that flag alone.
4. **Six contact slots is not proof of a six-touch hardware limit.** The report could be part of a hybrid multi-report scheme. The actual maximum and frame behavior remain untested.
5. Scan Time, Input Mode, and Latency Mode usages were not present in the parsed collection. Scan Time is optional in Microsoft's cited touchscreen requirements. No known standard standby/backlight control was found in this USB touch collection; that says nothing conclusive about separate Bluetooth/vendor interfaces.

## Input layout and decoder verification

The layout below was derived using Windows' parser to construct local synthetic reports, rather than by guessing from a VID/PID.

```text
Byte 0:       Report ID = 0x01
Bytes 1..30:  Six contact records, five bytes each
  record +0:  bit 0 Tip Switch, bit 1 In Range,
              bits 2..3 reserved, bits 4..7 Contact ID
  record +1:  X low byte
  record +2:  X high byte
  record +3:  Y low byte
  record +4:  Y high byte
Byte 31:      Contact Count
```

An offline decoder was tested against a synthetic six-contact report built with HidP_InitializeReportForID / HidP_SetUsageValue / HidP_SetUsages. **All 31 field round-trip comparisons passed** (five decoded fields per finger plus Contact Count). Two malformed-input rejection checks also passed. This verifies the decoder against Windows' parsed interpretation, not physical touchscreen behavior.

Synthetic test vector, never transmitted:

```text
01136500cf0023cc003c01333301a901439a0116025301028302636802f00206
```

## Live capture readiness

A shared GENERIC_READ overlapped handle could be opened successfully and closed without changing the driver. A bounded ReadFile capture then ran for **10.015 seconds**, with **zero errors and zero input reports**. No one was asked to wake or touch the screen; the screen state was not observed. This is **not a failed touch test**, and does not prove that reports never arrive.

The collector cancels pending reads at its deadline and closes its handles. It reads only the PhonePad VID/PID interface, not other keyboards/touchscreens.

## Why Linux might behave differently: source-based hypothesis

Lurker00's PhoneBook README at commit c577b70e90205e680359ac8ad376d9080c07a8c6 reports the same VID/PID with nonworking Windows touch and working Ubuntu 18.04/20.04 touch. This is related-hardware evidence, not a PhonePad result.

Linux upstream `hid-multitouch.c` in both v4.15 and v5.4 contains a CONTACTMAX fallback: if the parsed feature value is zero, it may use the descriptor's logical maximum instead. This provides a concrete possible explanation for Linux tolerating a faulty maximum-contact response. It is not proof of the exact Ubuntu build, active quirks, or behavior of this PhonePad under Linux.

## Next discriminating tests

1. Run the existing bounded collector while the owner wakes the screen with the wheel and taps/drags immediately. Repeat wake as needed if it blanks. Record whether local on-screen PhonePad controls respond while awake. Do not equate absence of touch-wake with a dead sensor.
2. If real report-1 coordinates arrive, assess a reversible user-space input bridge before considering a kernel driver. No such bridge has been implemented or proven here.
3. If no reports arrive during confirmed awake touch, compare this same unit on Linux and capture enumeration/control/interrupt traffic. Obtain raw descriptors and the on-wire report-2 exchange before changing drivers.
4. If firmware changes become justified, obtain a recoverable firmware backup first. Do not flash related PhoneBook firmware by analogy.

Screen blanking remains a separate unresolved symptom. Earlier Windows timeout checks were minutes, not seconds; the owner reports separate power and explicit Type-C mode were already tried. This pass did not identify or alter the backlight/standby mechanism.

## Evidence location and preservation status

Raw evidence and scripts are currently on **MHGPDW2M**:

```text
C:\Users\mhedh\Documents\PhonePad-research-20261011\diagnostics\
```

Key files:
- `hid-fields-20261011T225507Z.json`
- `contact-max-feature-20261011T225558Z.json`
- `feature-repeat-and-parser-20261011T225643Z.json`
- `decoder-selftest.json`
- `read-handle-check.json`
- `capture-20261011T230101Z.json`
- `inspect_hid_fields.py`, `read_contact_max.py`, `validate_feature_and_layout.py`
- `check_read_handle.py`, `capture_touch.py`, `test_decoder_windows.py`

A local `analysis-evidence.zip` was created, containing the evidence, scripts and per-file SHA256SUMS. ZIP size: **14,632 bytes**; SHA-256:

```text
d921f665d64883bc1ddc177604fe8d43935fe125306f901b66824ec934dd3dcd
```

**This commit preserves the written findings only.** Transfer of the raw evidence bundle encountered a tool safety block and was not completed or worked around. Do not claim the scripts/ZIP/raw JSON files are independently backed up in this repository yet. The older manuals/APK archive is unaffected.

The scripts as first executed target the local research directory and baseline interface path; they are investigation tooling, not a general-purpose driver package. Do not run `validate_feature_and_layout.py` casually: although it does not write device reports, it repeats the documented GET_FEATURE query.

## Sources

- Microsoft: [Required HID top-level collections](https://learn.microsoft.com/windows-hardware/design/component-guidelines/touchscreen-required-hid-top-level-collections). Mandatory contact information and maximum-contact feature; physical-axis metadata; optional Scan Time.
- Microsoft: [HidD_GetFeature](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/hidsdi/nf-hidsdi-hidd_getfeature). Buffer length and report-ID initialization.
- Microsoft: [HidP_GetUsageValue](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/hidpi/nf-hidpi-hidp_getusagevalue) and [Troubleshooting HID reports](https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/troubleshooting-hid-reports). Meaning of incompatible report IDs.
- Microsoft: [HIDP_VALUE_CAPS](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/hidpi/ns-hidpi-_hidp_value_caps), [HIDP_BUTTON_CAPS](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/hidpi/ns-hidpi-_hidp_button_caps), [HIDP_LINK_COLLECTION_NODE](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/hidpi/ns-hidpi-_hidp_link_collection_node). Structure definitions used for parsing.
- [Linux v4.15 hid-multitouch.c](https://github.com/torvalds/linux/blob/v4.15/drivers/hid/hid-multitouch.c#L400-L409) and [Linux v5.4 hid-multitouch.c](https://github.com/torvalds/linux/blob/v5.4/drivers/hid/hid-multitouch.c#L426-L434). Maximum-contact fallback.
- [Related PhoneBook research, pinned](https://github.com/Lurker00/PhoneBook/blob/c577b70e90205e680359ac8ad376d9080c07a8c6/README.md).
- Archived Anyware Products User Manual, PDF p.27: historically states PC touch was unavailable. Historical documentation is not substituted for these live results.
