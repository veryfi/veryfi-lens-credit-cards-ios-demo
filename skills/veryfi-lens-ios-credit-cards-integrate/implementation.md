# Implementation

Client contract: `configure` → present or upload → `VeryfiLensDelegate`.

Public settings type: `VeryfiLensSettings()`. Entry: `VeryfiLens.shared()` after `VeryfiLens`.

## Credentials

```swift
let credentials = VeryfiLensCredentials(
    clientId: "<clientId>",
    username: "<username>",
    apiKey: "<apiKey>",
    url: "https://api.veryfi.com" // or the URL Veryfi gave the customer
)
```

Store secrets in the host’s secure config, not in git. All four fields must be non-empty. Empty fields produce `veryfiLensError` with `msg` `invalid_credentials` and `data.empty_fields`.

Set `delegate` before or immediately after `configure` so that error is not dropped.


## Configure

```swift
let settings = VeryfiLensSettings()
// apply the recipe in settings.md

VeryfiLens.shared().delegate = self

VeryfiLens.shared().configure(with: credentials, settings: settings) { success in
    DispatchQueue.main.async {
        guard success else { return }
        // present or upload
    }
}
```

`completion(true)` means validation produced a usable `UserInfo` (or cached previous access). Remote `lens_settings` merge into settings unless `ignoreRemoteSettings` is true. This recipe sets that flag true.

Validation also returns account entitlements. Local fraud detection and local field detection are extra paid features gated by that backend response. `fraudDetectionIsOn` and `documentFieldsToDetect` only request entitled capabilities.

If validation fails: `completion(false)`, `lensConfigurationFailedNotification`, and typically a no-access error path. Reading `utils` before configure notifies `VeryfiLens is not configured`.

Reconfigure by calling `configure` again. Live tweaks: `VeryfiLens.shared().utils.update(settings:)` after a successful configure. That does not re-run full validation.

`appVersion` is an optional host version string for analytics. `isReactNative` is true only in React Native hosts. `customHeaders` adds HTTP headers on Lens API requests.


## Present — UIKit

Call only after `configure` returns true.

| Method | Role |
|--------|------|
| `showCamera(in:)` | Full Lens camera. Primary path for this recipe |
| `showGallery(in:)` | Photo library import |
| `showDocumentBrowser(in:)` | Standalone Files / iCloud Drive. No camera source flags required |
| `lensVC()` | Embeddable camera `UIViewController` |
| `showCrop(in:image:completion:)` | Manual crop of an existing image |
| `closeCamera(in:)` | Dismiss the camera flow |

```swift
VeryfiLens.shared().delegate = self
VeryfiLens.shared().configure(with: credentials, settings: settings) { success in
    DispatchQueue.main.async {
        guard success else { return }
        VeryfiLens.shared().showCamera(in: self)
    }
}
```

SwiftUI camera cover is the other path. One presentation only.

`deviceBlockListIsOn` may close the flow with `msg` `device_blocked`.

## Files, gallery, and browser entry points

The document browser can be opened from several independent places. `browseIsOn` only adds Browse under Import in the camera More menu. It is not a global Files permission.

| Entry | Settings | This recipe |
|-------|----------|-------------|
| Camera gallery control | `galleryIsOn` | Off. `galleryIsOn = false`. Do not call `showGallery` unless support agrees. Whether this flavor detects a card in a gallery photo is **Unconfirmed**. |
| Gallery control as a Photos-versus-Files menu | `galleryIsOn` and `galleryButtonWithFilesIsOn` | Off. Leave `galleryButtonWithFilesIsOn` at its default false unless the camera gallery control should open a Photos-versus-Files menu. |
| More menu → Import → Browse | `moreMenuIsOn` and `browseIsOn` | Not set. Default `browseIsOn` is true, and `moreMenuIsOn` is false, so the Browse row is hidden. |
| Camera top-right Files button | `showBrowserButton` | Not set. Default false. |
| Standalone Files, no camera | `showDocumentBrowser(in:)` or SwiftUI `.documentBrowser` | The API exists on the UI SDK. This recipe does not use Files. Whether a credit card can be read from a picked file is **Unconfirmed**. Do not send card images to a cloud extractor. |

SDK defaults before this recipe: `galleryIsOn` true, `browseIsOn` true, `moreMenuIsOn` true, `showBrowserButton` false, `galleryButtonWithFilesIsOn` false.

Use one presentation at a time. Do not call `showCamera` and a SwiftUI Lens cover together.

`autoCropGalleryIsOn` and `autoCropBrowserIsOn` are best effort for paper photos and scans. They also require `autoDocDetectionAndCropIsOn`. Internal digital-copy checks can preserve screenshots and digital copies. An original-file PDF is not page-cropped. Do not promise that every import is cropped.


## Present — SwiftUI

Configure and set the delegate the same way. Prefer the modifiers. Do not also call `showCamera` for the same presentation.

```swift
.veryfiLensFullScreenCover(isPresented: $showLens, flow: .camera)
.veryfiLensSheet(isPresented: $showGallery, flow: .gallery)
.veryfiLensSheet(isPresented: $showFiles, flow: .documentBrowser)
```

Item-based: `@State var activeFlow: VeryfiLensFlow?` with `.veryfiLensFullScreenCover(item:)`.

`VeryfiLensFlowView`, `VeryfiLens.shared().cameraView()`, and `VeryfiLensCameraView` ship with the full UI SDK surface compiled into this module. Session end posts `Notification.Name.veryfiLensDidClose`. Built-in modifiers dismiss on that notification. Keep a `VeryfiLensDelegate`. SwiftUI does not replace callbacks.


## UI no-UI helpers are not Headless

| Path | Use on this SDK |
|------|-----------------|
| `uploadImages(paths:)` | Host already has image or PDF paths. No Lens UI. Share Extensions and other short-lived processes. This path always uses cloud extraction. |
| `headlessUpload(images:)` / `headlessUpload(url:)` | In-memory images (first 4) or one URL. No Lens UI. Document type is `receipt`. Extraction follows `dataExtractionEngine`. |
| `VeryfiLensHeadless` | A different product. The host owns the camera. Do not import it for this flavor. |

Do not use `uploadImages` or `headlessUpload` to capture a card number. `uploadImages` always uses cloud extraction. Card data must stay on device and is never sent to the backend. The supported path is the Lens camera with `.inApp`. Whether those helpers run the credit-card detector is **Unconfirmed**.

`headlessUpload` on `VeryfiLens` is not `VeryfiLensHeadless`. A host-owned camera uses the headless credit-card binary and `VeryfiLensCreditCardsSettings`.

### `uploadImages(paths:)`

Call after `configure` returns true. The same `VeryfiLensDelegate` reports close, progress, success, and errors. `nil` reads `Documents/image.jpg`. Images are capped by `packageMaxScans` (clamped to 1...10). The document type is the first `documentTypes` entry, or `receipt` when that list is empty. This path always uses cloud extraction.

In a Share Extension, set `shareExtensionModeIsOn = true` before configure. Lens then skips thumbnails and posts the package zip directly. Leave the flag false in the main app so `uploadImages` uploads the package to storage and then processes it.

Copy a shared attachment into the extension container when the original URL is not readable, and pass that local path. Load credentials from the app group. The extension does not have the host app's in-memory session.

A bad file is reported on `veryfiLensError` and skipped. `msg` can be `invalid_file`, `empty_file`, `invalid_pdf_file`, `image_too_small`, `empty_image`, `invalid_image`, or `blurred_image`. A screenshot rejection stops the call.

Do not call `headlessUpload` from a Share Extension. Use `uploadImages(paths:)`.

Custom UI with detection and crop, when the host is not using the Headless product, uses `VeryfiLens.shared().utils` after configure. Do not construct `VeryfiLensUtils()`.


## Public functions

`VeryfiLens.shared()` is the entry point. Set `delegate` before `configure`. Read `credentials`, `settings`, and `utils` from that instance.

| `VeryfiLens` | Role |
|--------------|------|
| `configure(with:settings:completion:)` | Validate credentials and apply settings. Present or upload only after `true`. |
| `showCamera(in:)` / `showGallery(in:)` / `showDocumentBrowser(in:)` | Lens UI. One presentation at a time. |
| `showCrop(in:image:completion:)` | Manual crop of an image the host already has. |
| `lensVC()` | Embeddable camera controller. |
| `closeCamera(in:)` | Dismiss the camera flow. |
| `uploadImages(paths:)` | No UI. Image and PDF paths. Share Extension when `shareExtensionModeIsOn` is true. |
| `headlessUpload(images:)` / `headlessUpload(url:)` | No UI. In-memory images or one URL. Type `receipt`. This is not `VeryfiLensHeadless`. |
| `allowReuploadingFiles(withIdentifiers:)` | Clears `.always` duplicate hashes from a success `file_hashes`. |
| `shareOCRFeedback()` | Shares the last **code**-scan logs. It is not a barcode result API. |

`utils` is available after a successful configure:

| `VeryfiLens.shared().utils` | Role |
|------------------------------|------|
| `update(settings:)` | Replace settings and the uploader configuration. Does not re-validate credentials. |
| `updateCameraInfoView(_:)` | Replace the camera banner. Does not accept nil. |
| `packagesInQueue()` | Package ids still queued. |
| `retryAllPackages()` / `retryPackage(with:metadata:imageSourceTag:)` | Retry queued uploads. |
| `discardAllPackages()` / `discardPackage(with:)` | Upload queue only. Does not delete retained files after success. |
| `clearLensStorage()` | Queue plus package folders, zips, PDFs, TIFFs, thumbnails, and shared originals. |
| `getImages(from:)` / `getMetaData(from:)` / `getPdfURL(from:)` / `getZipPath(for:)` | Read artifacts while they still exist. |
| `base64(for:)` | Base64 string for `Data`. |
| `version` | SDK version string. |
| `isLensCameraCircleChromeStyleActive` | Circular camera chrome on iOS 26 when the SDK major version is 3 or newer. |
| `debugNativeModelsInventory()` / `resetDownloadedNativeModelsStateForDebugging()` | Demo diagnostics for the Remote Models flavor. |


## Help, custom screens, and camera banners

Help and `showInfoButton` share one camera control. Choose one tap behavior.

| Goal | Settings / API |
|------|----------------|
| Replay Lens's built-in guide | `showHelpButton = true`, `showDefaultScanningGuide = true` |
| Present a host view controller or sheet | `showHelpButton = true`, `showDefaultScanningGuide = false`, implement `onHelpButtonClicked(_:)` |
| Open a host UIKit view over the camera | `showHelpButton = false`, set `showInfoButton` to a `VeryfiInfoView` |
| Show an immediate top-of-camera banner | Set `cameraInfoView`. Independent of the shared Help/Info control |

Do not set both `showHelpButton` and `showInfoButton` expecting two buttons. Help wins.

`showInfoButton` is not purely cosmetic in the current runtime: a non-nil value makes the camera Files browser PDF-only and selects specialized retry cleanup and navigation. Use the delegate callback for a general-purpose custom help screen unless that specialized flow is intended.

`onHelpButtonClicked` runs only when `showHelpButton` is true and `showDefaultScanningGuide` is false. If the default guide is true, Lens shows its own guide and does not call the optional method.

The callback receives Lens's camera view controller so the host can present from the visible controller. For a view owned inside the camera, subclass `VeryfiInfoView`, let Lens pin it to the camera edges, and call inherited `close()`.

`cameraInfoView` appears immediately at the safe-area top with 16-point side insets. Give the banner intrinsic height or internal top and bottom constraints. Replace it while the camera is open with `VeryfiLens.shared().utils.updateCameraInfoView(_:)`. The update fades the old banner. It does not accept nil.

`customLensStrings` is `[languageCode: [sourceKey: replacement]]`. Lens looks up `Locale.current.languageCode`. The host app must list those languages under Project or Target Info → Localizations, or `CFBundleLocalizations`. The source key is the English string currently shown, copied exactly, or a `_veryfi_…` key from the unique-keys section of [customizable strings](https://docs.veryfi.com/lens/mobile/settings/customizable_strings/). Unlisted keys keep Lens's own localization. Keep format placeholders such as `%d` and `%@`.

`alertBackgroundColor`, `alertBackgroundDarkColor`, `alertFontColor`, `alertFontDarkColor`, `alertButtonHeight`, `alertButtonCornerRadius`, `alertContentMargins`, and `alertContentSpacing` style Lens's custom camera alert (blocked device, denied camera, denied Photos, configuration that failed validation). They do not restyle the system alerts used for submit, blur, glare, and no-document prompts. `alertSheetStyleIsOn` chooses an action sheet versus a regular system alert for those prompts. `swapAlertActions` reorders buttons on a regular alert. On iOS 26, system alerts stay regular alerts.

### Undetected-document and blur submit policy

`allowSubmitUndetectedDocsIsOn` is a legacy umbrella:

- Undetected true, blurred false or true: users can continue from both no-document and blur warnings.
- Undetected false, blurred true: a missing rectangle is blocked, and a detected blurry document can continue.
- Undetected false, blurred false: both warnings require a retake.

The blur warning shows “I'm happy with it” when either flag is true. The no-document warning only uses `allowSubmitUndetectedDocsIsOn`.


## Local resources and who deletes them

Local resources live in the app Documents directory. They are not the Photos or iCloud backup copies.

| Kind | Typical callback keys | On-disk examples |
|------|----------------------|------------------|
| Package images | `img_original_path`, `img_paths`, `img_original_paths` | `Documents/veryfi_pckupl/…/imageN.jpg` |
| Retained zip | `data.local_package_path` when auto-delete is off | `Documents/veryfi_zips/…zip` |
| Thumbnails | `img_thumbnail_path` / `img_thumbnail_paths` | `Documents/thumbnails/…` |
| Stitched / imported PDF | `img_stitched_pdf_path` | `Documents/veryfi_pdfs/…` |
| Check TIFFs | `img_check_front_tiff_path`, `img_check_back_tiff_path` | `Documents/veryfi_check_tiffs/…` |
| Shared session copies | same original-path keys when auto-delete is on | `Documents/image.jpg`, `original.pdf` |

`getImages(from:)`, `getPdfURL(from:)`, and `getZipPath(for:)` work only while those files still exist.

`autoDeleteLocalResourcesAfterProcessing` defaults to true. Lens then deletes that package’s local resources after processing. Copy what you need inside `veryfiLensUpdate` or `veryfiLensSuccess`. A single sequential document can keep the default if the host copies immediately.

Set it false when the host displays files after the callback, or when several packages are in flight. The host then owns deletion. `clearLensStorage()` wipes Lens local files and the upload queue. `discardAllPackages()` and `discardPackage(with:)` clear the upload queue only.

There is no Lens API that deletes one callback path. Use `FileManager` after the host has copied the file.

Check TIFF keys appear only when `retrieveCheckTiffIsOn` is on, which requires Remote Models. Do not read those keys on a Checks-specific binary.


## Decoupled capture

Do not switch this live read to `.none` or `.cloudAPI`. `.inApp` is the on-device engine for this document type. `.none` skips extraction. Cloud extraction is the wrong place for a payload that must stay on device when the type is `credit_card`. For `code` and `barcode`, the camera success path is the local result, not a Process a Document body.

`uploadWithoutProcessingIsOn` applies only with `.none`. Leave it unset on this recipe.


`inAppAPIProcessing` also calls the document-ready path with `document_type` `credit_card`, engine `mobile`, and no PAN fields. OCR or barcode bodies are added only for `code` and `barcode`. `credit_card` is not in the card set that forces the partner endpoint through the business-card, insurance, passport, and driver-license flag. The PAN stays in the delegate payload.

`cropLayoutWithAutoDetection` is the hybrid frame plus green box. Do not also require `cropLayoutIsOn`. The saved crop is still the frame’s rectangular bounds.

`documentTypes == ["credit_card"]` with the More menu on shows only “Close camera after submit” when `afterScanCloseCameraIsOn` is true. This recipe keeps the More menu off.


## Notifications

- `lensIsConfiguredNotification` — configure succeeded
- `lensConfigurationFailedNotification` — validation failed
- `veryfiLensDidClose` — SwiftUI session dismiss
