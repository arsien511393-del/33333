## PDF Signer – Summary of Additions

- **Overview**
  - Added a full RFQ/PDF signer flow that uses the **latest attachment** of a `ProcurementPending` item and automatically moves the item to the next step after successful signing.
  - Signatures are taken from the **user’s uploaded e-signature image** (`e_sign` on `user_tbl`) and applied directly on the PDF.

- **New / Updated Backend Logic**
  - **Controller**: `ProcurementPdfSignController`
    - `signPdfForm($rec_id)`: loads latest PDF attachment, resolves assigned user’s `e_sign`, pre-computes matches for the default search text (usually signer’s full name), and opens the signer page (`pages.procurementpending.sign-pdf`).
    - `signPdfPreview($rec_id)`: streams the latest PDF attachment for inline preview.
    - `signPdfMatches($rec_id)`: AJAX endpoint that returns matches for a search phrase in the PDF (used to populate “List of names” without reloading the page).
    - `signPdfPreviewPosition($rec_id)`: generates a **PDF preview** with rectangles showing where the signature will be placed.
    - `signPdfMatchImage($rec_id)`: generates a **PNG image** of the page with a rectangle on the target match (used for per‑match preview).
    - `signPdf(Request $request, $rec_id)`: applies the assigned user’s e-signature onto the PDF (one or more positions), saves a new `signed_XX_originalname.pdf`, attaches it to the current step status, and **auto-moves the pending item to the next step** (including creating/updating `ProjectStepStatus` and `ProjectStepMovements`).
    - Internal helpers:
      - Resolve and normalize attachment paths (`resolveAttachmentPath`, `getLatestPdfAttachment`, `getLatestPdf`).
      - Find Poppler executables (`getPdftotextPath`, `getPdftoppmPath`) and run them safely on Windows via `proc_open`.
      - Extract text/word bounding boxes from PDF (`extractPdfWordsBbox`, `parseBboxHtml`, `extractPdfLines`, `extractPdfTextWithPhp`, `computePdfMatches`, `findOccurrencesInPdfLayout`) to detect where the signer’s name appears.
      - Compute final signature rectangles on the PDF for FPDI (`getSignaturePositions`), including **fallback default positions** when exact coordinates are not available.
  - **Dependencies / libraries used**
    - `setasign\Fpdi\Fpdi` – overlay signature image onto existing PDF pages.
    - `Smalot\PdfParser\Parser` – PHP‑level PDF text extraction fallback.
    - Poppler tools: `pdftotext` and `pdftoppm` (via `proc_open`) – for text and image rendering.
    - `Intervention\Image` – draw preview rectangles on PNG images from `pdftoppm`.

- **New Routes**
  - In `routes/web.php` under `procurementpending`:
    - `GET procurementpending/sign-pdf-form/{rec_id}` → `ProcurementPdfSignController@signPdfForm` (`procurementpending.sign-pdf-form`).
    - `GET procurementpending/sign-pdf-matches/{rec_id}` → `ProcurementPdfSignController@signPdfMatches` (`procurementpending.sign-pdf-matches`).
    - `GET procurementpending/sign-pdf-preview/{rec_id}` → `ProcurementPdfSignController@signPdfPreview` (`procurementpending.sign-pdf-preview`).
    - `GET procurementpending/sign-pdf-preview-position/{rec_id}` → `ProcurementPdfSignController@signPdfPreviewPosition` (`procurementpending.sign-pdf-preview-position`).
    - `GET procurementpending/sign-pdf-match-image/{rec_id}` → `ProcurementPdfSignController@signPdfMatchImage` (`procurementpending.sign-pdf-match-image`).
    - `POST procurementpending/sign-pdf/{rec_id}` → `ProcurementPdfSignController@signPdf` (`procurementpending.sign-pdf`).

- **New / Updated Database & Model**
  - **Migration** `2026_01_30_100000_add_e_sign_to_user_tbl.php`
    - Adds `e_sign` (`string`, 255, nullable) to `user_tbl` after `user_photo`.
  - **Migration** `2026_02_26_000001_add_sign_pdf_permissions.php`
    - Inserts permissions for:
      - `procurementpending/sign-pdf-form`
      - `procurementpending/sign-pdf-preview`
      - `procurementpending/sign-pdf`
    - Grants them to `role_id` 1–5 by default (if not already existing).
  - **Model** `UserTbl`
    - `e_sign` added to `$fillable` and to various field lists (list, view, account edit/view, etc.), so the e-signature path is persisted and available to the signer.

- **New / Updated UI**
  - **Sign PDF page** – `resources/views/pages/procurementpending/sign-pdf.blade.php`
    - Left panel: PDF preview (`iframe` using `procurementpending.download-attachment` with `inline=1`).
    - Right panel:
      - **“List of names”**: shows matches for the current search text with checkboxes per match and badges indicating **exact** vs **approximate** positions.
      - Form to **Sign PDF**:
        - Field: **“Text to check in PDF (optional)”** (pre-filled with signer’s name by default).
        - Submits to `procurementpending.sign-pdf` with `search_text` and selected occurrences.
      - Behavior when no matches:
        - Still allows signing, but the system re-checks and **will not create a signed file or move the step** if the text is not found.
  - **Account E‑Sign page** – `resources/views/pages/account/esign.blade.php`
    - New standalone page for users to **upload their e-signature image** (JPG/PNG/GIF/JPEG, max 5MB).
    - Uses the existing Dropzone‑based uploader: `fileuploader/upload/e_sign`.
    - Shows current e-signature preview if already set.
  - **Account routes** in `routes/web.php`
    - `GET account/esign` → `AccountController@esign` (`account.esign`).
    - `POST account/esign` → `AccountController@esignStore` (`account.esign.store`).

- **Behavior / Flow Notes**
  - Only users with the proper permissions (via RBAC) can:
    - Open the signer form / matches (`procurementpending/sign-pdf-form`, `procurementpending/sign-pdf-matches`).
    - Preview the document (`procurementpending/sign-pdf-preview`).
    - Actually sign the PDF (`procurementpending/sign-pdf`).
  - The signer always uses the **latest PDF attachment** from the current step’s status; signed outputs are stored as:
    - `uploads/attachments/signed_XX_originalname.pdf` (auto‑incrementing `XX` per original document).
  - After a successful sign:
    - The new signed PDF is appended to the current `ProjectStepStatus.attachments`.
    - The system automatically sets the current step to **Completed**, creates/updates the **next step status** with the default assignee, updates the `ProcurementPending` record, logs a movement entry, and redirects the user back to **My Tasks** with a success message.

