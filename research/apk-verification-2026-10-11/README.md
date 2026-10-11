# Anyware APK baseline verification and replacement search

Date: 2026-10-11. This is a static verification report, not a security certification.

## Result

No newer Anyware release or demonstrably more trustworthy replacement APK was recovered in this pass. Retain the existing archived original. No replacement binary was downloaded or installed.

Both APKPure and AppBrain catalog version 1.1.6145 as the last/latest release. APKPure lists older versions 1.1.5778, 1.1.5625 and 1.0.5251; these are potential regression/reverse-engineering references, not demonstrated upgrades. Catalog dates differ: AppBrain reports January 20, 2021 as last update; APKPure displays March 3, 2021. Neither date was independently confirmed as a build date.

Sources consulted:
- https://apkpure.net/anyware/com.anyware.appctrl
- https://www.appbrain.com/app/anyware/com.anyware.appctrl
- https://developer.android.com/tools/apksigner

## Verified directly from the archived binary

- Package: `com.anyware.appctrl`
- Version: `1.1.6145`, versionCode `35`
- Minimum Android API: 23; target API: 29
- SHA-256: `ccb746976d5947837fc655813cffcbe48caee178bce22225022f9f428febc9e9`
- Signer certificate SHA-256: `e45233c203615e64c28ee189e9bb44fc31755b31f56d2a0c71197981f875bb32`
- Signer certificate SHA-1: `49ecb42c0f43b55585273b47dcb044de240c9bae`
- Certificate subject: `O=GD, L=SZ, C=CN`; self-asserted subject is not vendor identity proof.
- Android apksigner reports **Verifies**, exit 0, one signer, v1 JAR signing only. Later signing schemes and SourceStamp are not verified.
- Verifier warns about unsigned AndroidX version metadata under META-INF. These warnings are not themselves evidence of malware, but do limit what the signature protects. Keep the original bytes unchanged.

Raw tool output and structured results are in this directory. Signature validity verifies the signed contents against the included certificate, not that the certificate belongs to Anyware, and not that the application is safe.

## Permissions and compatibility

The manifest requests broad permissions, including storage, phone state, location, camera, overlays, usage statistics and package installation. Some requested permissions are privileged or restricted; declaration does not prove they will be granted or used maliciously. No runtime behavior or modern Android compatibility was tested. Use a dedicated test device for eventual dynamic analysis rather than treating this static pass as permission to install on a primary phone.

## Source retrieval limitations

See source-attempts.json. The original HTTP URL returned HTML, not a verified APK. HTTPS failed certificate validation with a hostname mismatch; TLS checking was not disabled. The dl.anywaretek.com wildcard Wayback CDX query returned an empty result. APKPure .net returned HTTP 403, while .com returned HTML. A subsequent content-inspection attempt was blocked by the tool safety check and was not bypassed. No independently retrieved second APK or trusted publisher signer baseline was obtained.

## Next evidence needed

A historical Google Play-installed copy or vendor-distributed original with credible provenance would let us compare signer certificates. A different mirror returning identical bytes corroborates distribution but does not independently prove publisher identity. Older releases may help compare signer continuity and firmware-update logic. Related PhoneBook-sagevt code is not a replacement Anyware companion APK and has not been validated for these PhonePads.
