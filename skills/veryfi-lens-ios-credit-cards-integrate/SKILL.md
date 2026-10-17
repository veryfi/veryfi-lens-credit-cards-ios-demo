---
name: veryfi-lens-ios-credit-cards-integrate
description: >-
  Integrates the Veryfi Lens iOS Credit Cards UI SDK. Covers pod VeryfiLens-CreditCards, on-device .inApp extraction, VeryfiLensSettings, the Lens camera, and credit-card success keys. Use when scanning credit cards with the Lens camera UI on iOS. Card data stays on device and is never sent to the backend. No dedicated SPM repo is published.
---

# Veryfi Lens iOS Credit Cards

Walk a host app from install to a working first scan. Change the fewest flags. Do not invent SDK APIs.

**Public settings type:** `VeryfiLensSettings()`. Never recommend `VeryfiLensReceiptsSettings`, `VeryfiLensChecksSettings`, or other Demo `VeryfiLens*Settings` subclasses.

**No SDK schema changes.** The recipe is property assignments only.

## Quick flow

Use this when the host already asked for this flavor and did not leave a choice open.

Assumptions:

- Lens camera UI, UIKit or SwiftUI
- Document type `credit_card`
- On-device `.inApp` extraction
- Gallery off, backups off, stitch off


1. Install the artifact and plist keys in [install.md](install.md).
2. Apply the recipe in [settings.md](settings.md).
3. Run the conflict table in [troubleshooting.md](troubleshooting.md#conflicts).
4. `configure`, then present, as in [implementation.md](implementation.md).
5. Implement [callbacks.md](callbacks.md).
6. Track [checklist.md](checklist.md).

State those assumptions in the sample. Do not ask extra questions on this path.

## Guided flow

Ask only what is still missing:

| Topic | Options |
|-------|---------|
| UI | UIKit or SwiftUI |
| UI | UIKit or SwiftUI Lens camera. A host-owned `AVCaptureSession` is the headless credit-card SDK |
| Logging | Confirm the host will not write PAN or CVC to analytics |
| Privacy | Photos backup, iCloud backup, location, keep local files after process |

If the host needs a document type this binary does not ship, stop. Say which types this flavor supports. Do not retarget `documentTypes` onto models this XCFramework does not contain. Any-Documents templates, including nutrition facts, US passport, driver license, and prescription labels, are a different solution. Contact support for a template that is not already confirmed. Credit-card capture stays on the credit-card flavor and on device. Bank-check TIFF needs Remote Models.

## Workflow

```
Lens integrate:
- [ ] Install artifact and plist
- [ ] Recipe on VeryfiLensSettings
- [ ] Conflict check
- [ ] configure and delegate
- [ ] One presentation, or uploadImages when the host already has files
- [ ] close / update / success / error
```

Read one level deep, and only inside this folder:

- [install.md](install.md) — artifact, SPM or CocoaPods, plist, app size
- [implementation.md](implementation.md) — credentials, configure, present, Files and gallery, uploadImages, utils
- [settings.md](settings.md) — recipe and the public settings catalog
- [callbacks.md](callbacks.md) — delegate payloads
- [troubleshooting.md](troubleshooting.md) — symptoms and conflicts
- [clarifications.md](clarifications.md) — Unconfirmed items
- [checklist.md](checklist.md) — done when

## Warnings

- Leave `backupDocsToGallery` and `backupDocsToiCloud` false unless the host wants every scan copied into Photos or iCloud. Gallery backup still defaults to true. Hide the More menu, or the iCloud row, so a user cannot turn the copies back on.
- Any-Documents is marketed as a broad solution: `any_document` plus `anyDocumentTemplate`. The template list is not closed. Nutrition facts, US passport, driver license, and prescription labels are shortcut flavors of Any-Documents, not separate products. Contact support for another template, including one that is not supported yet. Do not set `anyDocumentType` (deprecated local regex OCR). This binary is not that solution.
- Credit card data is local only and is never sent to the backend. `.inApp` (`"mobile"`) is only for `credit_card`, `code`, and `barcode`.
- Local fraud detection (`fraudDetectionIsOn`) and local field detection (`documentFieldsToDetect`) are paid features gated by the validated account and backend entitlement. Settings cannot unlock them. Field detection also needs the Field Detection or Remote Models flavor.
- Bank-check TIFF (`retrieveCheckTiffIsOn`) requires Remote Models. It is not in the Checks-specific flavor yet, because the TIFF model is large.
- When size matters, recommend Remote Models. Measure the final app with the SDK linked and with it removed. Do not quote the XCFramework file size. [SDK binary sizes](https://docs.veryfi.com/lens/mobile/settings/sdk_binary_sizes/).
- `uploadImages(paths:)` and `headlessUpload` on `VeryfiLens` are UI SDK helpers. They are not the Headless product (`VeryfiLensHeadless`).
- Public UI settings are `VeryfiLensSettings()`. Do not use Demo subclasses in client code.
- Headless products use `VeryfiLensCreditCardsSettings` or `VeryfiLensReceiptSettings` (singular), `VeryfiLensHeadlessCredentials`, and `VeryfiLensHeadless.shared()`.
- Leave `dictateIsOn` false. It is obsolete and not partner available. The SDK default is still true.
- Leave `ignoreRemoteSettings` true unless the host wants dashboard `lens_settings` to override the recipe.
- Leave `glareDetectionIsOn` false unless the host explicitly wants the current glare gate. It is a suboptimal algorithmic detector.


## Suggestions

- Copy card fields inside `veryfiLensSuccess` before relying on image paths.
- Add branding after the first successful local read.
- If the host owns the camera and passes sample buffers, use the headless credit-card SDK and `VeryfiLensCreditCardsSettings`. Do not point `VeryfiLens.shared().showCamera` at that product.
- If size matters, say so. Remote Models is the size recommendation for flavors that embed heavy models. Measure the final app with the SDK linked and without it. Do not quote the XCFramework file size. Whether a Remote Models build replaces this credit-card UI binary is **Unconfirmed** here. Ask Veryfi before substituting artifacts.


## Do not

- Do not set `.cloudAPI` or `.none` for a card number.
- Do not install `veryfi-lens-spm`.
- Do not use Demo subclass `VeryfiLensCreditCardSolutionSettings`.
- Do not treat `uploadImages` or `headlessUpload` as the card scan.
- Do not leave `backupDocsToGallery` true if gallery is later turned on.


## Output

When generating host code, produce:

1. Info.plist keys actually needed
2. `configure` plus a retained delegate
3. One presentation path
4. The recipe’s `VeryfiLensSettings` mutations, with a one-line comment on each non-obvious flag
5. A short conflict note if any residual risk remains
