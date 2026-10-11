# Anyware PhonePad preservation and research

Operation Declutter project. Two battery-equipped PhonePads owned; owner reports working external-display operation. HardwareHacking index: https://github.com/MahdiHedhli/HardwareHacking/tree/main/projects/anyware-phonepad

## Source documents received

- `AnyWare PhonePad PhoneBook Manual Manuel Tablette.pdf`: 31-page Anyware Products User Manual, uploaded by owner. Page references below refer to PDF page numbers, not its table-of-contents numbering.
- `PhonePad.pdf`: two-page PhonePad leaflet supplied by owner following this FCC reference: https://fccid.io/2AYTA-S2/User-Manual/User-manual-5100441 . Exact hardware-revision correspondence remains unverified.
- `Anyware_1.1.6145_APKPure.apk`: owner-supplied Android application. Filename version is not independently verified package metadata. Treat as untrusted until static inspection and signature verification.

**Preservation status:** this commit contains notes only. The PDFs and APK have NOT been copied into this repository. Conversation attachments are not an independent permanent backup. Firmware binaries have not been recovered.

## Confirmed from the 31-page manual

- Page 3: distinguishes standard and battery PhonePads. Separate USB-C power and input ports. Battery model turns on by holding the menu wheel. Manual specifies original 5V/3A adapter or a 30W quick-charging adapter; it does not document negotiation details.
- Pages 6-9: phone connects to a USB-A port with its charging/data cable; iOS Trust prompt and Android app workflow. iOS supports single-point touch; Android is recommended for multitouch. USB keyboards/mice attach to the appliance; Bluetooth peripherals can pair directly with the phone.
- Page 10: computers, consoles and Raspberry Pi connect through HDMI or USB-C input. The manual calls its phone USB-C section 'MHL-enabled phones like Samsung DeX'; retain that terminology as a source claim, not independent verification of the electrical protocol.
- Pages 14-15: Ctrl+Esc switches work/gaming modes. Menu wheel and companion app control brightness, contrast and volume. App offers output-resolution setting.
- Page 17: documented firmware update workflow requires USB-A phone connection AND Bluetooth pairing, then Personal > Firmware Update > Update. Screenshot names Download the Firmware and Transmit the Firmware stages. Update endpoint, transport details, image format, signature enforcement and rollback are not specified.
- Page 20: connection indicators represent smartphone Bluetooth, keyboard and mouse. Manual provides an MTK compatibility mode.
- Pages 26-27: troubleshooting requires matching iOS/Android Bluetooth mode. iOS instructions use AssistiveTouch and minimum pointer tracking speed. Page 27 explicitly says PC touchscreen support is not available yet. This is a historical documented limitation, not a tested assertion about every later firmware.
- Page 29: Smooth mode trades resolution for lower latency, Game mode uses medium resolution/low latency, Theater uses high resolution/high latency.
- Page 30: gaming unavailable for iOS 13.4 onward; some Android phones do not send audio over USB. No present-day compatibility testing is implied.

## Owner-observed hardware

HDMI, USB-C input, separate USB-C power connector, headphone jack, three USB-A ports marked SuperSpeed, menu wheel, battery, speakers and touchscreen. On-screen input choices: USB / HDMI / Type-C. Observed Android-mode version: `3.5275.577.44.44`. Device-name prefix: `Anywtk-`.

## Research leads, not confirmed compatibility

- Original product archive: https://web.archive.org/web/20210302144236/https://anywaretek.com/products/phonepad
- Related PhoneBook research: https://github.com/Lurker00/PhoneBook
- Related Android touch component: https://github.com/Lurker00/PhoneBook-sagevt
- Previously decoded leaflet QR destination: http://dl.anywaretek.com/anyware . Prior retrieval failed; that does not establish that the server is permanently offline.

PhoneBook findings do not establish PhonePad chipset identity or firmware interchangeability. An on-screen menu and update workflow do not prove the display runs a general-purpose Android installation.

## Next investigation, deferred

1. Independently archive original PDFs/APK with SHA-256, source provenance and signature metadata where applicable.
2. Inspect APK statically for firmware-update endpoints, model identifiers, package format and integrity checks. Do not install on a trusted phone or trigger an update merely to observe it.
3. Characterize USB and Bluetooth separately in each input/mode; do not assume touch is standard USB HID, especially given the manual's PC limitation.
4. Identify hardware and obtain recoverable backups before firmware changes. Preserve one unit unchanged.
5. Evaluate native custom userspace versus an added SBC only after chipset, memory, power, display and touch paths are known.
