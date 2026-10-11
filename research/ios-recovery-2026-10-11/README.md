# Anyware iOS recovery leads

Date: 2026-10-11

## Status

**No IPA recovered, verified, installed, or backed up during this investigation.** This document preserves source-derived download leads, not an iOS software backup. Current App Store availability, bundle identifier, release version, signing identity, and runtime compatibility remain unverified.

## Evidence from supplied manual

`archive/manuals/AnyWare PhonePad PhoneBook Manual Manuel Tablette.pdf`, PDF page 3, explicitly directs users to download Anyware by scanning the on-screen QR or searching App Store/Google Play. Page 5 shows an iOS App Store screenshot. This establishes historical iOS app distribution, not present availability.

## Original QR landing page recovered

Source: http://dl.anywaretek.com/anyware/

The HTML was retrieved as text on the owner's ColPanicM2 using Python urllib with an iPhone User-Agent. The response was printed by Remote Desktop Commander and inspected statically. Its JavaScript was not executed and no provisioning profile was installed. This retrieval used plain HTTP: the contents are a valuable lead, not cryptographically authenticated vendor evidence. Earlier web-tool retrieval failures must not be taken to mean this landing page is permanently offline.

The response contains the following exact declarations (excerpt, not a full HTML snapshot):

```javascript
//var ipaUrl = "https://gamestore.fr.sageres.com:8332/gamectrllist/download/app/ios";
var ipaUrl = "https://apps.apple.com/cn/app/anyware/id1490849644";

//var apkUrl = "https://dl.res.sageres.com/gamectrl.dl/android/GameCtrl.apk";
//var apkUrl = "./AnywareAndroid_1.0.3136.apk";
var apkUrl = "http://svc2.onebitplay.com/onebit-list/file-records/anyware.apk";
```

### iOS leads

- Active iOS target in the recovered page: https://apps.apple.com/cn/app/anyware/id1490849644
- App Store numeric ID: `1490849644`.
- US storefront URL tested separately: https://apps.apple.com/us/app/anyware/id1490849644
- Commented-out legacy endpoint: https://gamestore.fr.sageres.com:8332/gamectrllist/download/app/ios
- The legacy endpoint's name does not establish that it serves a raw IPA. It could return a redirect, manifest, landing page, or error.
- The landing page also references `./catrust.mobileprovision` and exposes a trust-related link for iOS. This file was not retrieved or inspected. Do not install an unknown profile merely to follow this lead.
- Other commented numeric App Store IDs occur in generic/legacy code. They are not verified Anyware identifiers and must not be substituted for the active ID above.

## Retrieval limitations

- Web retrieval of the CN and US App Store links returned inaccessible responses; this does NOT establish delisting or country restrictions.
- An attempted follow-up remote batch for Apple lookup, legacy endpoint, and archive-index checks was blocked by the tool safety layer before execution. No lookup results or archive-index results were obtained from that batch, and it was not retried via a workaround.
- Public searches did not yield an identifiable downloadable IPA for this PhonePad/PhoneBook app. Namesake applications are not substitutes.

## Android lead discovered incidentally

The active APK target above was missed during the earlier APK search. It is worth comparing against the preserved APK in a separately authorized retrieval/static-verification pass. No newer APK or stronger publisher provenance has been established by discovering this URL alone. Preserve the current APK regardless; do not replace it based on filename or download origin.

## Next recovery evidence needed

1. Check whether App Store ID 1490849644 opens on the owner's device/account, including previously downloaded apps.
2. Look for an existing owner-held IPA or old app-library archive. An installed app is useful evidence, but does not guarantee an exportable/reinstallable IPA.
3. For any recovered IPA, record original source, SHA-256, file size, Info.plist bundle/version identifiers, signing/provisioning metadata, and executable encryption status. Distinguish archival integrity from authenticity, installability, and safety.
4. Keep account-specific receipts or other personal metadata out of public research commits.
