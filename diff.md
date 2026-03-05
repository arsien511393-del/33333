# PDF Signer & Procurement Pending Processing

This document describes how the **PDF signer** works within **procurement pending** processing. It covers the flow from opening the signer to applying a signature and moving the item to the next step.

---

## Overview

- **Procurement pending** represents a procurement item at a specific **step** (e.g. BAC, Regional Director). Each pending record is tied to a **project**, a **step**, and an **assigned user**.
- The **PDF signer** lets an authorized user place an e-signature image on the latest PDF attachment for that step, then **automatically completes the current step and moves the item to the next step** with default assignment.

No shell/install or server setup is covered here; only application flow and behavior.

---

## How Users Reach the PDF Signer

1. **My Tasks** (`procurementpending/my-tasks`) — List of pending items assigned to the current user.
2. **View** a pending item, or open **Update Status** (`procurementpending/update-status-form/{id}`).
3. On the Update Status page, use the **Sign Document** button to open the **PDF Signer** (`procurementpending/sign-pdf-form/{id}`).

If the user has only **sign-pdf** (e.g. Regional Director), they can still open the signer via direct link or from view; they do not need **update-status-form**.

---

## PDF Signer Page Flow

### 1. Opening the form (`signPdfForm`)

- Loads the **pending** record and the **latest PDF** from the current step’s attachments (from `project_step_status`).
- Uses the **assigned user’s** e-signature image (`user_tbl.e_sign`). If the assigned user has no e-sign, the user is redirected back with a message to set e-sign in Account.
- Default **“Text to check in PDF”** is the assigned user’s full name. This text is used to find **matches** in the PDF (where to place the signature).
- **Matches** are computed from the PDF (bbox/layout or fallback). They are shown as a list; the user can check/uncheck which occurrences get a signature.

### 2. Live search (“Text to check in PDF”)

- The user can change the text and trigger a live lookup (`sign-pdf-matches/{id}`) to refresh the list of matches.
- Matches can have **exact position** (from pdftotext bbox) or **approximate position** (signature placed at a default spot on the page).

### 3. Submitting the signature (`signPdf` POST)

- **Permissions:** User must have `procurementpending/sign-pdf` (or equivalent as per RBAC).
- **Inputs:** `search_text`, `selected_occurrences[]` (which match indices to use).
- **Validation:**
  - If the user entered **search text** and that text is **not found** in the PDF, the system **does not** create a signed file and **does not** move the step. User is redirected with a message.
  - If no search text or text is found, signing continues.

### 4. What happens on successful sign

1. **Signature placement**
   - Positions are computed from bbox (or default positions on the first page).
   - The assigned user’s e-sign image is drawn on the PDF at those positions (all pages that have at least one position).
   - A new file is saved as `signed_XX_OriginalName.pdf` in the attachments upload folder (`uploads/attachments` by default).

2. **Attachments**
   - The new signed PDF path is **appended** to the **current** step’s `project_step_status.attachments` (latest status for this project/step).

3. **Current step completion**
   - The **current** `project_step_status` is updated to `status = 'Completed'`.

4. **Next step**
   - The next step is determined by `procurement_steps.step_id` (next higher `step_id`).
   - If there is a next step:
     - **Next step status:** Existing latest `project_step_status` for that step is updated, or a new one is created. It is set to `In Progress`, with default assignee from `procurement_steps.user_id` (or BAC Sec for step 2 if not set). The signed PDF is added to the next step’s attachments.
     - **Pending record** is updated: `step_id`, `status_id`, `assigned_desig_id`, `assigned_user`, `is_current = 1`, and `remarks` (e.g. “Auto-moved to next step after PDF signing.”).
     - A **movement** record is created in `project_step_movements` (from previous assignee to next assignee, `Completed` → `In Progress`).
   - If there is **no** next step:
     - The pending record is marked as completed/closed: `is_current = 0`, `closed_at` set.

5. **Redirect**
   - User is redirected to **My Tasks** with a success message (e.g. “PDF signed successfully. Signed file has been added to attachments and the item has been moved to the next step.”).

---

## Permissions (summary)

- **Open PDF Signer (form):** `procurementpending/sign-pdf-form` or `procurementpending/update-status-form` or `procurementpending/sign-pdf`
- **Preview PDF / matches / position / match image:** Same as above (or `procurementpending/sign-pdf-preview` for preview only).
- **Submit signature:** `procurementpending/sign-pdf` (or `update-status-form` as per your RBAC).

See **PERMISSIONS-GUIDE.md** for full URL-to-permission mapping and role examples.

---

## Key Behaviors (English summary)

| Situation | Result |
|-----------|--------|
| Assigned user has no e-sign | Redirect to view with message; must set e-sign in Account. |
| No PDF in latest attachments | Redirect to view with “No PDF found in latest attachments.” |
| User enters “Text to check in PDF” and it is **not** found in the PDF | No signed file created; step not moved; user notified. |
| User submits with valid (or empty) search text and text is found (or not required) | Signed PDF created and attached; current step set to Completed; pending moved to next step (or closed if no next step). |
| After successful sign | Redirect to **My Tasks** (not Update Status), because assignee may have changed. |

---

## Routes (reference)

| Method | Route | Purpose |
|--------|--------|--------|
| GET | `procurementpending/sign-pdf-form/{id}` | Open PDF signer page |
| GET | `procurementpending/sign-pdf-preview/{id}` | Serve PDF for inline preview |
| GET | `procurementpending/sign-pdf-matches/{id}` | AJAX: get matches for “Text to check in PDF” |
| GET | `procurementpending/sign-pdf-preview-position/{id}` | Preview PDF with signature box(es) |
| GET | `procurementpending/sign-pdf-match-image/{id}` | PNG of first page with signature rectangle for one match |
| POST | `procurementpending/sign-pdf/{id}` | Submit signature and apply to PDF |

All of the above are handled by `ProcurementPdfSignController`. Procurement pending list, view, and Update Status are in `ProcurementPendingController`.

---

## Processing summary (procurement pending)

1. **Pending** = one item (project + step + assignee) in the workflow.
2. **Update Status** = change step status, add remarks, attach files, optionally complete and move to next step manually.
3. **Sign PDF** = place e-signature on latest PDF, then **automatically** complete current step and move pending to next step (or close if last step).
4. Signed file is stored as `signed_XX_OriginalName.pdf` and appended to both current and next step attachments when moving.
5. Movement and assignee changes are recorded in `project_step_movements` and in the pending record’s `assigned_user` / `assigned_desig_id`.

This document focuses only on the application flow for **processing procurement pending** with the PDF signer, in English, without shell or environment setup.
