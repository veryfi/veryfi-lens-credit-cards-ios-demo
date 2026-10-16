# Install

Lens camera UI for credit cards. This is not the headless credit-card SDK. Headless uses `VeryfiLensHeadless`, `VeryfiLensHeadlessCreditCards`, and `VeryfiLensCreditCardsSettings`. Client settings type here: `VeryfiLensSettings()`. Do not use `VeryfiLensCreditCardSolutionSettings`.

Card data stays on device. `.inApp` (`"mobile"`) is required. `veryfiLensSuccess` `data` is the on-device extraction. Credit card data is never sent to the backend.

## Artifact and import

| | |
|--|--|
| Pod | `VeryfiLens-CreditCards` |
| CocoaPods repo | `veryfi-lens-cocoapods-credit-cards` |
| Dedicated SPM | **No** |
| XCFramework file name | `VeryfiLens.xcframework` |
| Swift import | `VeryfiLens` |

```swift
import VeryfiLens
```



## Swift Package Manager

No dedicated SPM repo is published for this flavor. `supportsSPM` does not include it. Do not add `https://repo.veryfi.com/shared/lens/veryfi-lens-spm.git`. That URL is the Full UI binary.

Install with CocoaPods or by embedding the XCFramework Veryfi supplied for this flavor.


## CocoaPods

Public docs call CocoaPods deprecated and prefer SPM where a package exists. CocoaPods remains the pipeline path for this flavor.

```ruby
source 'https://repo.veryfi.com/shared/lens/veryfi-lens-podspec.git'
source 'https://github.com/CocoaPods/Specs.git'

target 'YourApp' do
  use_frameworks!
  pod 'VeryfiLens-CreditCards'
end
```

Public pages pin a version with `pod 'VeryfiLens-CreditCards', 'x.x.x.x', :linkage => :dynamic`. The current version number is **Unconfirmed** here. Use the version Veryfi gave the customer.


## Manual XCFramework

Embed `VeryfiLens.xcframework`. The pod name selects the flavor. Embedding a Full `VeryfiLens.xcframework` does not turn it into this product.

## Permissions

| Key | When |
|-----|------|
| `NSCameraUsageDescription` | Camera. This recipe is camera-only |
| `NSPhotoLibraryUsageDescription` | Not required while gallery stays off |
| `NSPhotoLibraryAddUsageDescription` | Not required. Backup stays off |
| `NSLocationWhenInUseUsageDescription` | Not required. Location stays off |

Do not add Contacts or Calendar usage strings unless product confirms those features ship in this flavor. The legacy framework README lists them. That list is **Unconfirmed** and likely stale.

iOS 13+ for UIKit Lens. SwiftUI helpers are documented as iOS 15+.

## App size

When installed size matters, recommend the **Remote Models** flavor. A specific flavor embeds its models in the binary. Remote Models keeps that binary smaller and downloads most of the heavy model resources after install, only for the features the host turned on.

Bank-check TIFF follows that split. `retrieveCheckTiffIsOn` is not supported on the Checks-specific flavor yet, because the TIFF model is large. Remote Models downloads it after install.

Measure the final build product, once with this Veryfi Lens SDK linked and once without it. The distributed XCFramework is a fat artifact that includes every architecture and platform slice. That file size is not the size added to the installed app. Store delivery can shrink the download further.

Flavor tables and the artifact-versus-app explanation: [SDK binary sizes](https://docs.veryfi.com/lens/mobile/settings/sdk_binary_sizes/).


## Public installation page

Public installation URL for this UI flavor was not fetched. **Unconfirmed** whether a docs page exists. The pod name is `VeryfiLens-CreditCards`. Do not add `veryfi-lens-spm`.

## Optional local evidence

These paths are evidence from the Veryfi Lens iOS source tree. They are not part of this skill pack. Ignore them when this folder is the only checkout. Nothing in the integration depends on opening them.

- `fastlane/swift/FastlaneSwiftRunner/SPM.swift — repo names and XCFramework file names`
- `fastlane/swift/FastlaneSwiftRunner/Pipeline.swift — supportsSPM and supportsCocoaPods`
- `fastlane/swift/FastlaneSwiftRunner/Cocoapods.swift — pod repos and XCFramework file names`
- `fastlane/swift/FastlaneSwiftRunner/Builder.swift — FrameworkType raw values (pod names)`
- `Veryfi-Lens/Public/VeryfiLens.swift — UI entry point compiled into this module`
- `Veryfi-Lens/Public/Model/VeryfiLensSettings.swift — public UI settings`
- `VeryfiLensEngine/Shared/Models/CreditCard.swift — UI json keys`
- `VeryfiLensEngine/Shared/Upload/PackageUploader/PackageUploader.swift — inAppAPIProcessing`
- `fastlane/swift/FastlaneSwiftRunner/Pipeline.swift — credit cards omitted from supportsSPM`
