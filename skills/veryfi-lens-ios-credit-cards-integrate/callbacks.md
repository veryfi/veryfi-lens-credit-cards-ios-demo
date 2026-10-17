# Callbacks

Implement `VeryfiLensDelegate`: `veryfiLensClose`, `veryfiLensError`, `veryfiLensSuccess`, `veryfiLensUpdate`. Optional `onHelpButtonClicked` only if Help presents a host screen.

`veryfiLensSuccess` for `.inApp` merges the on-device card dictionary into `data`. Keys from the UI card JSON:

| Key | Meaning |
|-----|---------|
| `credit_card_number` | PAN string |
| `credit_card_holder_name` | Name |
| `credit_card_date` | Date string. A second date is appended with `/` when present |
| `credit_card_type` | Brand string |
| `credit_card_cvv` | CVC |
| `credit_card_id` | Local UUID |
| `credit_card_is_invalid` | String from `(!isValidLuhn).description`, not a Bool |

`status` on that package success is `"done"`. If no card dictionary was stored, the in-app path reports failure message `No data was extracted using VeryfiInApp engine`.

`data` may also gain `detected_fields`, `img_paths`, or `local_package_path` from the success merge. Field detection is not part of this recipe.

The package is cleared after the in-app success. Copy the card fields inside `veryfiLensSuccess` before relying on image paths. Local auto-delete still defaults to true.

These keys differ from the headless credit-card dictionary (`card_number`, `card_name`, `card_dates`, `card_cvc`).

`veryfiLensClose` is the camera session ending. Card success and close are both delivered. The exact ordering beyond “success is emitted, then the camera close runs” is **Unconfirmed** for the UI coordinator versus the barcode and OCR path, which calls success and then schedules `closeLens()` on the main queue.

`veryfiLensError` covers configure and credential failure (`msg` `invalid_credentials`).

`inAppAPIProcessing` also posts document-ready with `document_type` `credit_card`, engine `mobile`, and no PAN fields. The PAN stays in the delegate payload. Credit card data is never sent to the backend.

Do not log `credit_card_number` or `credit_card_cvv` in analytics by default.
