## Pagkakaiba: `Procurement System` (development) vs `backup04032026` (live)

**High-level overview**  
- **`Procurement System/`** = mas bago, may mga bagong feature at enhancements.  
- **`backup04032026/`** = kasalukuyang live/backup version.  
- Ang lahat ng nasa `changes/` ay alinman sa:
  - Bagong files na **wala** sa `backup04032026`, o
  - Lumang files pero **may binago** kumpara sa `backup04032026`.

**Reference format na gagamitin sa baba**  
- Gagamit tayo ng short notation na ganito:  
  - `@web.php (92-93, backup)` → tumutukoy sa file na `routes/web.php`, lines 92–93 sa **backup** version.  
  - `@web.php (86-92, dev)` → tumutukoy sa `routes/web.php`, lines 86–92 sa **development/Procurement System** version.  
- Halimbawa:  
  - `@web.php (92-93, backup)` – route na:  
    - `Route::post('procurementpending/approve-rfq/{rec_id}', 'ProcurementPendingController@approveRfq')->name('procurementpending.approve-rfq');`  
    - Note: pareho ito sa dev at backup (walang change).

---

## 1. Routes at endpoints (`routes/web.php`)

Sa `Procurement System/routes/web.php` **may mga dagdag at binagong routes** na wala o iba sa `backup04032026/routes/web.php`.

- **Bagong routes para e-sign (PDF signing workflow)**  
  - `procurementpending/sign-pdf-form/{rec_id}` → `ProcurementPdfSignController@signPdfForm`  
  - `procurementpending/sign-pdf-matches/{rec_id}` → `ProcurementPdfSignController@signPdfMatches`  
  - `procurementpending/sign-pdf-preview/{rec_id}` → `ProcurementPdfSignController@signPdfPreview`  
  - `procurementpending/sign-pdf-preview-position/{rec_id}` → `ProcurementPdfSignController@signPdfPreviewPosition`  
  - `procurementpending/sign-pdf-match-image/{rec_id}` → `ProcurementPdfSignController@signPdfMatchImage`  
  - `procurementpending/sign-pdf/{rec_id}` (POST) → `ProcurementPdfSignController@signPdf`  
  → **Wala ang mga routes na ito sa `backup04032026/routes/web.php`.**

- **Bagong routes para Award Signatory module**  
  - `awardsignatory` → list/index  
  - `awardsignatory/index/{filter?}/{filtervalue?}`  
  - `awardsignatory/view/{rec_id}`  
  - `awardsignatory/add` (GET + POST)  
  - `awardsignatory/edit/{rec_id}`  
  - `awardsignatory/delete/{rec_id}`  
  → **Buong `awardsignatory` route group ay wala sa `backup04032026`.**

- **Bagong Excel export endpoints**  
  - `app/export-full-excel` → `AnnualProcurementPlanController@exportFullToExcel`  
  - `ppmp/export-full-excel` → `PPMPController@exportFullToExcel`  
  → Sa live, meron lang basic `export-excel` per record, wala yung **full export** versions.

- **Mas strict na RBAC sa downloads at file upload**  
  - Sa dev: tinanggal ang `->withoutMiddleware(['rbac'])` sa:
    - `app/download-attachment/{app_req_id}`  
    - `ppmp/download-attachment/{ppmp_req_id}/{file_index?}`  
    - `fileuploader/upload/{fieldname}`  
    - `fileuploader/s3upload/{fieldname}`  
    - `fileuploader/remove_temp_file`  
  → Ibig sabihin, sa dev version kailangan nang dumaan sa RBAC middleware ang mga actions na ito; sa live, mas “bukas” pa sila.

- **Mas mahigpit na deletion routes**  
  - Tinanggal ang ilang `GET` delete routes sa dev:
    - `procurementpending/delete/{rec_id}`  
    - `usertbl/delete/{rec_id}`  
    - `app/delete/{app_id}` (GET at DELETE variants)  
    - `ppmp/delete/{ppmp_id}` (GET at DELETE variants)  
  → Sa live, available pa ang mga delete endpoints na ito.

- **E-sign routes**  
  - Sa dev, `account/esign` at `account/esign` (POST) **naka-attach na sa RBAC middleware** (tinanggal ang `withoutMiddleware(['rbac'])`).  

- **Bagong API search endpoints**  
  - `api/app/search` → JSON search ng APP records (title + budget, may pagination flag).  
  - `api/users/search` → JSON search ng `UserTbl` (full_name).  
  → Wala ang dalawang API routes na ito sa live.

---

## 2. Menu at navigation (`app/Helpers/Menu.php`)

- **Bagong sidebar submenu item**  
  - Sa dev may bagong entry:  
    - Path: `awardsignatory`  
    - Label: `"Award"`  
    - Icon: material icon `description`  
  → Ibig sabihin, sa dev UI meron nang **"Award"** section sa menu na naka-link sa bagong Award Signatory module; wala ito sa live.

- **Bagong helper method para PPMP label**  
  - `Menu::ppmpLabel($ppmp_ref)` – ginagamit para kunin ang PPMP title batay sa `ppmp_ref` (for read-only display, hal. sa APP edit).  
  → Sa live version, wala pa itong helper; direct ref string lang ang madalas ginagamit.

---

## 3. Controllers at models (bagong features / behavior)

**Bagong controllers (wala sa live):**

- `app/Http/Controllers/AwardSignatoryController.php`  
  - CRUD at view para sa Award Signatory records.  
  - Naka-tie sa bagong `awardsignatory` routes at views.

- `app/Http/Controllers/ProcurementPdfSignController.php`  
  - Responsable sa e-signing ng RFQ/award PDFs (preview, matches, signing workflow).  
  - Ginagamit ng mga `procurementpending/sign-pdf-*` routes.

**Controllers na nagbago ang logic (parehong file pero iba ang laman):**

- `AnnualProcurementPlanController`  
  - Dagdag: full Excel export (`exportFullToExcel`), mas advanced attachments/download handling, at archive-related logic base sa migrations.  

- `PPMPController`  
  - Dagdag: full Excel export, archived/deleted handling, approve/decline remarks at iba pang enhancements na naka-tie sa bagong columns.  

- `ProcurementPendingController`  
  - Mas maraming status-update logic, tracking, at integration sa PDF signing flow.  

- `HomeController`, `PermissionsController`, `ProcurementCategoriesController`, `ProcurementProjectsController`, `Rbac` middleware, `ProcurementPendingStatusUpdateRequest`, at iba pa  
  - May adjustments para sa bagong permissions, division/archiving logic, at mas mahigpit na access control.

**Bagong model (wala sa live):**

- `app/Models/AwardSignatory.php`  
  - Eloquent model para sa bagong Award Signatory table.  
  - Konektado sa bagong migrations at controllers.

**Models na binago:**

- `Permissions.php`, `UserTbl.php`  
  - Dinagdagan ng fields/relations para:
    - E-sign settings (`e_sign` field at related logic)  
    - Division / role at bagong permissions set  
    - Archiving flags at audit-related attributes.

---

## 4. Database layer (migrations, seeders, SQL dump)

**Bagong migrations (dev only, wala sa live):**  
Lahat ng sumusunod ay nasa `Procurement System/database/migrations` pero wala sa `backup04032026`:

- Bagong tables at structure:
  - `2024_12_20_000003_create_project_type_table.php`  
  - `2026_01_12_093255_create_security_audit_logs_table.php`  
  - `2026_01_15_000000_create_failed_login_attempts_table.php`  
  - `2026_01_20_000010_create_rfq_signatory_table.php`  
  - `2026_01_20_000011_create_ppmp_signatory_table.php`  
  - `2026_01_20_000012_create_app_signatory_table.php`  
  - `2026_02_04_000001_create_award_signatory_table.php`  

- Structural changes / extra columns:
  - `2026_01_30_000001_signatory_use_user_id.php`  
  - `2026_01_30_000002_signatory_add_user_id_columns_if_missing.php`  
  - `2026_01_30_100000_add_e_sign_to_user_tbl.php`  
  - `2026_01_30_120000_add_rfq_award_document_path_to_procurement_steps.php`  
  - `2026_01_30_130000_replace_rfq_award_with_step_document_in_procurement_steps.php`  
  - `2026_01_31_000001_create_division_tbl.php`  
  - `2026_01_31_000002_add_division_id_to_user_tbl.php`  
  - `2026_01_31_000003_add_division_permissions.php`  
  - `2026_01_31_000004_add_division_ref_to_division_tbl.php`  
  - `2026_01_31_000005_add_is_archived_to_user_tbl.php`  
  - `2026_01_31_000006_add_approve_remarks_to_ppmp.php`  
  - `2026_02_01_000001_add_is_archived_to_ppmp_and_app.php`  
  - `2026_02_02_000001_add_decline_remarks_to_ppmp.php`  
  - `2026_02_03_000001_add_is_archived_to_procurement_projects.php`  
  - `2026_02_04_000002_add_award_signatory_permissions.php`  
  - `2026_02_26_000001_add_sign_pdf_permissions.php`  

→ Sa madaling sabi: **dev version** may suportang:
- Divisions at division permissions  
- Archiving flags (user, PPMP, APP, procurement projects)  
- Remarks para approve/decline  
- E-sign columns at RFQ award document path  
- Security audit logs at failed login attempts  
- Award signatory at iba pang signatory tables  

**Iba pang DB assets:**

- `Procurement System/database/region3_epms.sql`  
  - SQL dump na wala sa live folder; malamang dev snapshot/export ng database schema/data.

**Seeders:**

- Bagong seeders na wala sa live:
  - `BacCommitteeUsersSeeder.php`  
  - `PpmpPendingSeeder.php`  
  - `PpmpSampleSeeder.php`  
- `DatabaseSeeder.php`  
  - Magkaiba ang laman sa dev vs live (mas maraming initial data at permissions setup sa dev).

---

## 5. Frontend assets at public files

- **CSS & JS**  
  - `public/css/custom-style.css` – iba ang laman (additional styling para bagong pages/modules).  
  - `public/css/page-css.css` – updated per page styles.  
  - `public/js/page-scripts.js` – extra JS logic para tracking, search, at PDF sign workflows.  

- **HTACCESS**  
  - Root `.htaccess` at `public/.htaccess` parehong may differences → malamang performance/caching/security tweaks sa dev version.  
  - `public/.htaccess-copy` – extra copy lang sa dev.

- **Uploads / attachments / files**  
  - `Procurement System/public/uploads/attachments`:
    - Maraming **sample RFQ, Resolution to Award, at signed PDF/docs** na wala sa live (test/sample data).  
  - `backup04032026/public/uploads/attachments`:
    - May ilang RFQ files na nasa live lang, wala sa dev copy.  
  - `Procurement System/public/uploads/files`:
    - Maraming `.png` at ilang `.pdf` demo assets para UI/previews.  
  - `backup04032026/public/uploads/files`:
    - May ilang Excel at image na nasa live lang.  

→ Sa kabuuan, **magkaiba ang laman ng uploads** dahil magkaibang test data / live data ang naka-save.

---

## 6. Config / dependencies

- `composer.json` at `composer.lock` **magkaiba**  
  - Sa dev: may dagdag/updated packages (hal. auditing, PDF/Excel, e-signature related, at iba pang helpers).  
  - Sa live: mas luma o mas kaunti ang dependencies.  

- `bootstrap/cache`  
  - Sa live lang may `packages.php` at `services.php` (compiled cache ng Laravel).  
  - Sa dev side kadalasan hindi ito committed o nag-iiba habang nagde-develop.

---

## 7. Environment at git metadata

- `.env`  
  - Parehong meron pero **magkaiba ang laman** (iba ang DB/app keys/host).  
  - **Hindi dapat i-copy direkta** mula dev papuntang live.

- `.git` folder  
  - Nasa `Procurement System/` lang (source repo metadata).  
  - Wala nito sa `backup04032026/` (deploy copy lang siya, hindi buong git repo).

---

## 8. Summary (para sa deployment decisions)

- **Bagong features sa dev na wala pa sa live:**
  - Award Signatory module (routes, controller, model, views, migrations, permissions).  
  - PDF e-sign workflow para procurement pending (lahat ng `sign-pdf-*` routes at controller).  
  - Full Excel exports para APP at PPMP.  
  - Mas malalim na permissions, divisions, archiving, audit logs, at e-sign configuration.  
  - Bagong API endpoints para app search at user search (AJAX/Select2-style).  

- **Security / access changes:**
  - Pinahigpit ang RBAC sa downloads at file uploads (tinanggal `withoutMiddleware(['rbac'])`).  
  - Tinanggal o binago ang ilang delete routes para maging mas safe ang operations.  

- **Database changes:**  
  - Maraming bagong tables at columns (division, audit logs, signatories, archived flags, remarks, e-sign, RFQ award path, atbp).  

Kung gusto mo, pwede tayong gumawa ng mas detalyadong section per file (hal. `AnnualProcurementPlanController.php`, `PPMPController.php`, at iba pa) na naka-bullet lang din, base sa kung aling mga bahagi ang pinaka-importante sa deployment mo.

