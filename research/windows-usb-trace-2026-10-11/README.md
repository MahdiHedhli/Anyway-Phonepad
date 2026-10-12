# PhonePad USB trace and feature/string correlation

Session: 2026-10-11 evening America/New_York (2026-10-12 UTC).
Target: owner's battery-equipped Anyware PhonePad, USB VID:PID **0525:B4B1**, connected to Windows 11 on MHGPDW2M.

Related baseline: [Windows HID investigation](../windows-hid-2026-10-11/README.md).

## Result

Two distinct observations materially narrow the investigation:

1. The existing USB ETW trace records **3,725 successful 32-byte interrupt-IN completions** attributed to this PhonePad after reconnecting. The interface was not silent at the USB-stack level in this capture. These are transfer completions, not 3,725 verified touch events: their payload contents are not available in the trace.
2. A new bounded, read-only experiment shows that **GET_FEATURE report 2 returns the header of the immediately preceding product/manufacturer string descriptor**, rather than its advertised Contact Count Maximum report. Alternating the string request changes the next feature result from `16 03` to `4E 03` and back, reproducibly.

This establishes a data-dependent invalid feature response through the Windows HID API. Stale USB control-transfer buffer reuse is a strong explanation, likely in the device's request handling, but the exact implementation and causal role in Windows touch failure are not yet proven. No driver fix, working Windows touch, or screen-blanking fix was demonstrated.

## Scope and privacy

This pass analyzed an existing ETL and made only standard read-only HID product/manufacturer-string and advertised feature-report queries. No additional user touches were requested. No new capture, driver installation, reboot, firmware write, output report, touch injection or display-setting change occurred.

The original ETL and its complete XML conversions cover the shared USB stack and remain on the owner's computer. This document intentionally publishes only target-specific findings and verification metadata, not unrelated device traffic, kernel addresses, account data or the full shared trace.

## 1. Original capture integrity and decoding

The owner recorded a 60-second USB ETW session while reconnecting the PhonePad and testing touch.

| Property | Result |
| --- | --- |
| Session | `PhonePadUSB-6c1163a2d114` |
| Original ETL | `usb_000001.etl` |
| Size | 5,898,240 bytes |
| Capture start | `2026-10-12T01:28:40.7179540Z` |
| Start in owner timezone | `2026-10-11 21:28:40.717954 -04:00` |
| Events processed | 14,644, across the shared USB stack |
| Events lost | 0 reported by the trace summary |
| Providers | USBXHCI, UCX, USBHUB3 |
| Configured keywords | Default + PartialDataBusTrace (`0x81`) |

The current original ETL SHA-256 **matches the checksums.json record produced when the capture stopped**:

```text
ae5ca186263e2cbd4ab91ccb722af1d340da075c2fb26016516e7274e134ee90
```

The first strict XML conversion reported a schema warning. Re-conversion with `tracerpt -of XML -lr` completed without that warning. Its `events-loose.xml` is 37,019,711 bytes, SHA-256:

```text
03c23c1df049a6fe7ebf8b53d3005bc95a10ce0999fa74d9fba9dd652d85a1e7
```

A two-pass local summarizer first resolves the PhonePad identity from its VID/PID and USB interface registration, then filters events by the resulting device handle. It also maps the pipe handles using UCX Endpoint Create events. This avoids treating traffic from other USB devices as PhonePad data. A provisional metadata summary was superseded by the verified version-2 summary.

## 2. Input transfers are present below the earlier HID reader

The trace identifies interrupt-IN endpoint **0x81**, 32-byte maximum packet size, and interrupt-OUT endpoint **0x02**, also 32 bytes. The presence of an OUT endpoint does not contradict the earlier absence of advertised HID Output reports, and does not establish a vendor command or firmware-update interface.

| PhonePad input transfer check | Result |
| --- | --- |
| UCX submit events | 3,727 |
| UCX completion events | 3,725 |
| Completed transfer length | 32 bytes for every completion |
| Completed USBD status | `0x00000000` for every completion |
| Completed NT status | `0x00000000` for every completion |
| Completions matched to a submit by pipe/IRP/URB | 3,725 |
| Unmatched completions | 0 |
| Outstanding submissions at trace end | 2 |
| First completion | `2026-10-11T21:28:51.533019400-04:00` |
| Last completion | `2026-10-11T21:29:40.787370700-04:00` |
| Median completion gap | Approximately 14.008 ms |
| Largest completion gap | Approximately 4,173.985 ms |

Microsoft documents that the completed URB's TransferBufferLength is the number of bytes transferred. This is stronger evidence than merely seeing polls or pending reads.

However, the contents might be valid contacts, idle reports, malformed reports or something else. The earlier user-space ReadFile capture and this ETW capture were not simultaneous. Their different outcomes must not be presented as a proven same-packet loss point inside Windows.

## 3. Initialization sequence

The trace records the following target-specific initialization activity:

- SET_CONFIGURATION to configuration 1 completes successfully.
- HID SET_IDLE (`21 0A 00 00 00 00 00 00`) fails with USBD status `0xC0000004` (STALL_PID) and NT status `0xC0000001`. Other initialization and input transfers continue. The stall is not, by itself, the demonstrated cause of missing Windows touch.
- The HID report descriptor request (`81 06 00 22 00 00 FF 01`) requests 511 bytes and completes successfully with **447 bytes**. The descriptor bytes themselves are not present in this capture's decoded transfer data.
- A product string request for index 2, language `0x0409`, completes at `21:28:51.528277500-04:00` with **22 bytes**.
- Immediately afterward Windows submits GET_REPORT for feature report 2: **`A1 01 02 03 00 00 02 00`**. It completes at `21:28:51.529123800-04:00`, status success, length 2.

The feature transaction in this trace confirms Windows requests the problematic capability during initialization. It does **not** reveal the returned two bytes. The invalid bytes were obtained through the separate HID API tests described below and in the baseline investigation.

A Prepare Hardware event has a suspended flag at `21:28:51.511929600-04:00`, followed immediately by successful D0 Entry at approximately `21:28:51.512030300-04:00`. This is a startup state transition, **not evidence that Windows caused the owner's later short-interval screen blanking**. No timestamped visual observation connects a later screen blank to a USB power event.

## 4. Native ETL audit: payloads are not merely hidden by XML conversion

The existing ETL was also read directly using Windows OpenTraceW / ProcessTrace with EVENT_RECORD callbacks. The native reader processed all 14,644 records, returned success and recorded no callback errors.

For PhonePad UCX events:

| Event | Count | UserData length |
| --- | ---: | ---: |
| Control transfer EX start, ID 23 v0 | 9 | 176 |
| Control transfer EX completion, ID 24 v1 | 9 | 180 |
| Descriptor-from-device request, ID 34 v0 | 4 | 176 |
| Control completion, ID 21 v1 | 7 | 180 |
| Class-interface request, ID 53 v0 | 2 | 176 |
| Descriptor-from-interface request, ID 35 v0 | 1 | 176 |
| Interrupt submission, ID 26 v0 | 3,727 | 168 |
| Interrupt completion, ID 27 v1 | 3,725 | 172 |

The observed lengths account for the fixed device/pipe/request handles, URB structure and completion status. No 32-byte input payload is appended. Target records have a 22-byte ETW extended-data item of type 12; examination did not uncover HID report payloads there either. The recorded TransferBuffer values are addresses, not copies of the buffer contents.

Consequently this existing trace cannot provide coordinate decoding or an on-wire reproduction of the invalid maximum-contact response. Further parsing of the same metadata is not a substitute for a payload-bearing capture.

## 5. New read-only feature/string experiment

Recorded at **`2026-10-12T01:45:19.905701+00:00`**.

A shared handle with desired access zero was opened to the previously identified PhonePad HID collection. The only queries were:

- `HidD_GetFeature(handle, [0x02, 0xA5], 2)` for the already advertised Contact Count Maximum feature.
- `HidD_GetManufacturerString`.
- `HidD_GetProductString`.

No serial number was requested. All nine query calls returned success; none of the five feature results matched report ID 2.

| Sequence | String returned, when applicable | Next feature-report buffer |
| --- | --- | --- |
| Baseline feature read | Not applicable | `16 03` |
| Read manufacturer, then feature 2 | `Linux 3.10.24 with 18038000.usb_otg_hs` | `4E 03` |
| Read product, then feature 2 | `HID Gadget` | `16 03` |
| Repeat manufacturer, then feature 2 | Same manufacturer string | `4E 03` |
| Repeat product, then feature 2 | Same product string | `16 03` |

A USB string descriptor starts with its byte length and descriptor type 3. The manufacturer string has a 78-byte descriptor (`0x4E`), while the product string has a 22-byte descriptor (`0x16`). Thus **each feature response matches the preceding string descriptor's first two bytes**, reproducibly.

### Interpretation and boundaries

This is substantially stronger than observing one fixed wrong report ID. It demonstrates cross-request dependence consistent with stale control-transfer response data. In particular, the second byte `03` is not a verified maximum of three touches: it matches the string descriptor type instead.

The leading hypothesis is faulty firmware-side GET_REPORT handling returning stale endpoint-zero buffer contents. This remains an inference from behavior; there is no matched PhonePad firmware source, debugger session or USB payload capture locating the exact faulty code path. The upstream Linux v3.10 f_hid.c GET_REPORT implementation clears its buffer, so it must not be cited as already containing the observed stale-string behavior. A vendor fork or another layer must be investigated rather than assumed.

The manufacturer's USB string directly reports **Linux 3.10.24** and controller name **18038000.usb_otg_hs** on this PhonePad. This is no longer solely a similarity inferred from PhoneBook research. It is still a device-supplied string, not an authenticated kernel-version attestation, exact SoC identification, firmware backup or proof of a usable general-purpose tablet OS.

## 6. What is and is not established

**Established:** local touch works according to the owner; this USB capture contains successful 32-byte PhonePad input transfers; GET_FEATURE 2 returns invalid bytes dependent on the preceding string read; Windows issues feature 2 during initialization; the PhonePad reports the Linux/controller string above.

**Not established:** valid touch-coordinate payloads, the raw HID report descriptor, actual maximum contact count, the exact point where usable input is lost, the physical touch-controller chip, a working Windows fix, or the cause of screen blanking.

A maximum-contact compatibility fix is now a concrete investigation target, but changing that response alone has not been shown to restore touch.

## 7. Next discriminating step

Capture USB transfer payloads and the raw HID descriptor from this unit. A signed USBPcap filter-driver installation is a candidate on Windows; installation and any restart require coordination and were not performed. The project's official page documents a restart requirement and older supported Windows versions; compatibility with this exact Windows 11 build must be checked before installation. A Linux comparison with USB monitoring is another route and also requires coordination.

The next payload capture should first prove that actual data bytes are being collected before asking the owner to repeat touches. Collect feature initialization and a short awake touch sequence, avoiding the local on-screen menu. Do not install random touch drivers, disable signing enforcement or flash PhoneBook firmware by analogy.

## 8. Evidence retained locally

Capture directory:

```text
C:\Users\mhedh\Documents\PhonePad-research-20261011\diagnostics\usb-trace\PhonePadUSB-6c1163a2d114\
```

New local analysis files:

- `events-loose.xml`, `trace-summary-loose.txt`
- `phonepad-target-summary.json` (verified schema_version 2)
- `phonepad-control-transfers.json`
- `native-etl-record-audit.json`
- `native-etl-extensions-audit.json`
- `summarize_existing_trace.py` (offline two-pass identity filter and transfer summary)

The read-only correlation experiment is stored in the parent diagnostics directory as `feature-string-correlation-20261012T014519Z.json`, SHA-256:

```text
4e9f33b1acbcefc94c3211a0bc068411287338a704042ad9f201a2b026f44af3
```

This commit preserves the target-specific findings and reproducible query sequence. It does not publish or claim an independent backup of the shared ETL, full XML or original raw diagnostic files.

## Primary references

- [Microsoft: USB_STRING_DESCRIPTOR](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/usbspec/ns-usbspec-_usb_string_descriptor): length, type and Unicode string layout.
- [Microsoft: URB_BULK_OR_INTERRUPT_TRANSFER](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/usb/ns-usb-_urb_bulk_or_interrupt_transfer): IN direction and completed transfer length.
- [Microsoft: HidD_GetFeature](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/hidsdi/nf-hidsdi-hidd_getfeature): report-ID buffer contract.
- [Microsoft: touchscreen required HID collections](https://learn.microsoft.com/en-us/windows-hardware/design/component-guidelines/touchscreen-required-hid-top-level-collections): maximum-contact capability.
- [Microsoft: USB ETW capture](https://learn.microsoft.com/en-us/windows-hardware/drivers/usbcon/how-to-capture-a-usb-event-trace).
- [Microsoft: EVENT_RECORD](https://learn.microsoft.com/en-us/windows/win32/api/evntcons/ns-evntcons-event_record) and [EVENT_TRACE_LOGFILEW](https://learn.microsoft.com/en-us/windows/win32/api/evntrace/ns-evntrace-event_trace_logfilew): native offline trace audit.
- [Linux v3.10 f_hid.c](https://github.com/torvalds/linux/blob/v3.10/drivers/usb/gadget/f_hid.c): reference only, not matched PhonePad firmware source.
- [USBPcap official site](https://desowin.org/usbpcap/): signed installer and restart requirement.
