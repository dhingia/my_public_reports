# Project 1 — Technical & Product Analysis Report

**Project name (verified in source):** Smart School — School Management System (CodeCanyon item ID 19426018, version 7.2.0)
**Vendor:** QDOCS (`smart-school.in`, `support@qdocs.net`)
**Distribution:** Single-tenant PHP source code distributed via Envato CodeCanyon
**License type:** GPL (per `readme.txt`)
**Package analysed:** `codecanyon-19426018-smart-school-school-management-system` containing `smart_school_src/` (the installable v7.2.0 source) and `smart_school_update_7.1.0_to_7.2.0/` (delta update package)
**Analysis scope:** Source code + schema only. No live demo was available. No code was executed.

> This report is the result of static source-code analysis. Wherever a finding is supported by code, the file path and relevant symbol are cited.

---

## 1. Product Overview

### Purpose
Smart School is a complete school-administration ERP aimed at K-12 schools. It covers students, staff, fees, exams, attendance, transport, hostel, library, front-office, payroll, accounting, communication, alumni, online admissions and online examinations, plus a public-facing CMS.

### Target users
Six primary role families are seeded in `controllers/install/database.sql` (`INSERT INTO roles`):

| id | Role | Type |
|---|---|---|
| 1 | Admin | System |
| 2 | Teacher | System |
| 3 | Accountant | System |
| 4 | Librarian | System |
| 6 | Receptionist | System |
| 7 | Super Admin | System + super-admin flag |

Plus two non-staff user types in `users` table: Student, Parent, and Guest (used for online admission preview).

### Business model
- **CodeCanyon one-time purchase** with Envato **purchase-code** verification.
- **Add-on modules** sold separately on CodeCanyon and installed via an in-app ZIP uploader that extracts to `./addons/` and executes SQL via `multi_query()`. 11 add-ons are advertised in `addons` seed (CBSE exam, 2FA, Multi-Branch, Behaviour Records, Online Course, Zoom/GMeet live class, QR Attendance, Quick Fees, Thermal Print, WhatsApp Messaging).
- Mobile apps (Flutter) are referenced via `mobile_api_url`, `app_primary_color_code`, etc. in `sch_settings` — i.e. the PHP backend is the API for vendor-supplied mobile apps.
- A **SaaS layer is stubbed but disabled**. `config/saas-config.php` has `$config['saas_enabled'] = false;` and `libraries/SaasValidation.php` returns `TRUE` for every quota/validation call. `libraries/ResourceQuota.php` is an empty class.

### Technology stack
| Layer | Technology |
|---|---|
| Language | PHP (compatible with both PHP 7.x and 8.x) |
| Framework | CodeIgniter 3 (`system/` directory, `MY_Controller` pattern) |
| Database | MySQL via `mysqli` driver (utf8 / utf8_general_ci collation) |
| Frontend | Server-rendered PHP views + jQuery + AdminLTE-style theme; Bootstrap, FullCalendar, DataTables, Dropify, Select2, SweetAlert |
| Mobile API | REST-ish JSON endpoints (`controllers/App.php`, `controllers/Biometric.php`) |
| Push notifications | Firebase Cloud Messaging v1 (OAuth + service-account JSON) |
| PDF | mPDF |
| Mail | PHPMailer, AWS SES |
| Payments | 28 gateway integrations (`controllers/onlineadmission/` and `controllers/gateway_ins/`) |
| SMS | 11 providers — Clickatell, Twilio, MSG91, SMS Country, TextLocal, Bulk SMS, MobiReach, Nexmo, Africa's Talking, SMS EG, Custom |

### Main capabilities (high level)
Student lifecycle (admission → enrollment → attendance → fees → exams → TC), staff lifecycle (joining → attendance → leave → payroll → exit), fee collection with discounts/fines/instalments, examination & marksheet generation, transfer certificate generation, ID cards, library circulation, hostel allocation, transport routing with pickup points, inventory, front-office (enquiries/visitors/dispatch/complaints), alumni, internal chat, public website with CMS, biometric & QR attendance, online admission with payment, online examination, notification centre (SMS/Email/Push/WhatsApp), multi-language (78), multi-theme (6 admin themes), backup/restore, audit log.

---

## 2. Complete Feature Inventory

Feature modules are derived from the controller tree (`application/controllers/admin/*`, `application/controllers/user/*`) and from the seed in `sidebar_menus`. Permissions are gate-kept by `Rbac::hasPrivilege($category, $permission)` where `$permission ∈ {can_view, can_add, can_edit, can_delete}`.

### 2.1 Student Information
- Controllers: `controllers/admin/Student.php`, `controllers/admin/Onlineadmission.php`, `controllers/admin/Onlinestudent.php`, `controllers/admin/Alumni.php`, `controllers/admin/Stdtransfer.php`, `controllers/admin/Disable_reason.php`
- Tables: `students`, `student_session`, `student_doc`, `student_timeline`, `alumni_students`, `alumni_events`, `disable_reason`, `online_admissions`, `online_admission_payment`
- Features: admission (admin + online paid form), bulk import via CSV, multi-class enrollment, document upload, student timeline, transfer certificate, ID card, admission number auto-generation, alumni management.

### 2.2 Fees Collection
- Controllers: `controllers/admin/Feemaster.php`, `controllers/admin/Feegroup.php`, `controllers/admin/Feetype.php`, `controllers/admin/Feediscount.php`, `controllers/admin/Feesforward.php`, `controllers/admin/Feereminder.php`, `controllers/admin/Offlinepayment.php`, `controllers/Studentfee.php`, `controllers/admin/Studentfee.php`
- Tables: `feemasters`, `fee_groups`, `feetype`, `fee_groups_feetype`, `fee_session_groups`, `student_fees`, `student_fees_master`, `student_fees_deposite`, `student_fees_discounts`, `student_applied_discounts`, `student_fees_processing`, `student_transport_fees`, `transport_feemaster`, `fee_receipt_no`, `fees_reminder`, `fees_discounts`, `offline_fees_payments`, `cumulative_fine`
- Features: fee type / fee group / fee master definition, group–type–session assignment with fine type (`none|percentage|fixed|per_day`) and `due_date`, fine per day, discounts, back-date fee collection, partial payment, fee carry-forward, offline bank payment reconciliation, balance-fee report, fee receipt PDF.
- **Critical implementation detail**: `student_fees_deposite.amount_detail` is stored as serialized JSON inside a TEXT column. Each deposit is a JSON list keyed by incrementing counter with `amount`, `amount_discount`, `amount_fine`, `payment_mode`, `date`, `description`, `received_by`. This is essentially a sub-table embedded in a TEXT field.

### 2.3 Attendance
- Controllers: `controllers/admin/Stuattendence.php`, `controllers/admin/Staffattendance.php`, `controllers/admin/Subjectattendence.php`, `controllers/admin/Biometric.php`, `controllers/Attendencereports.php`
- Tables: `student_attendences`, `staff_attendance`, `student_subject_attendances`, `attendence_type`, `staff_attendance_type`, `class_section_times`
- Features: class+section daily attendance, subject-wise attendance, biometric push-API, half-day/late/holiday types, low-attendance limit alert, cron auto-marks-absent at scheduled time.

### 2.4 Examinations
- Controllers: `controllers/admin/Exam.php`, `controllers/admin/Examgroup.php`, `controllers/admin/Examschedule.php`, `controllers/admin/Examresult.php`, `controllers/admin/Onlineexam.php`, `controllers/admin/Admitcard.php`, `controllers/admin/Marksheet.php`, `controllers/admin/Mark.php`
- Tables: `exams`, `exam_groups`, `exam_schedules`, `exam_group_class_batch_exams`, `exam_group_class_batch_exam_students`, `exam_group_class_batch_exam_subjects`, `exam_group_exam_connections`, `exam_group_exam_results`, `exam_group_students`, `mark_divisions`, `grades`, `onlineexam`, `onlineexam_attempts`, `onlineexam_questions`, `onlineexam_students`, `onlineexam_student_results`, `questions`, `template_admitcards`, `template_marksheets`
- Features: hierarchical "exam group → exam → schedule → marks → result"; 5 grading systems; mark divisions; online exam with attempts, duration, negative marking, random questions, passing percentage, auto-publish, descriptive answers with word limit; question bank; admit cards and marksheet templates.

### 2.5 Academics (subjects, timetable, syllabus, homework, lesson plan)
- Tables: `subjects`, `subject_groups`, `subject_group_class_sections`, `subject_group_subjects`, `subject_syllabus`, `subject_timetable`, `homework`, `homework_evaluation`, `submit_assignment`, `daily_assignment`, `lesson`, `topic`, `lesson_plan_forum`
- Features: subject grouping, class teacher subject mapping, timetable generation, syllabus tracking (lesson → topic → status), homework with marks/evaluation/documents.

### 2.6 Human Resources / Payroll
- Tables: `staff`, `staff_designation`, `department`, `staff_roles`, `staff_attendance`, `staff_attendance_type`, `staff_payroll`, `staff_payslip`, `payslip_allowance`, `staff_leave_request`, `staff_leave_details`, `leave_types`, `holiday_type`, `staff_id_card`, `staff_timeline`, `staff_rating`
- Features: staff CRUD with extensive profile; payslip with allowance/deduction/leave deduction/tax/net_salary; payslip status (`generated`, `paid`, `unpaid`, `not_generate`); leave request/approval workflow; staff ID card generation; staff rating.

### 2.7 Library
- Tables: `books`, `book_issues`, `libarary_members` (sic — typo in schema)
- Features: book catalog with rack/shelf, member issuance, due-date tracking, fine calculation.

### 2.8 Hostel & Transport
- Tables: `hostel`, `hostel_rooms`, `room_types`, `vehicles`, `vehicle_routes`, `transport_route`, `transport_feemaster`, `pickup_point`, `route_pickup_point`, `student_transport_fees`
- Features: hostel & room-type allocation; vehicle registry; route + pickup-point mapping with lat/long; transport fee master.

### 2.9 Inventory
- Tables: `item`, `item_store`, `item_category`, `item_stock`, `item_supplier`, `item_issue`, `dispatch_receive`
- Features: store/items/categories/suppliers/stock, item issue to staff/student, dispatch & receive.

### 2.10 Front Office
- Tables: `enquiry`, `enquiry_type`, `follow_up`, `general_calls`, `complaint`, `complaint_type`, `visitors_book`, `visitors_purpose`, `dispatch_receive`, `reference`, `source`
- Features: admission enquiries + follow-ups, visitor book, general call log, complaint register, postal dispatch/receive, reference & source master.

### 2.11 Communication
- Libraries: `libraries/Mailer.php`, `libraries/Mailgateway.php`, `libraries/Mailsmsconf.php` (78 KB), `libraries/Smsgateway.php` (138 KB), `libraries/Pushnotification.php`
- Tables: `notification_setting`, `send_notification`, `read_notification`, `notification_roles`, `chat_connections`, `chat_messages`, `chat_users`, `messages`, `email_config`, `email_template`, `sms_config`, `sms_template`
- Features: per-event template configuration (`config/mailsms.php` lists 24 event types), per-channel toggle (SMS / Email / Push / WhatsApp), per-recipient-class toggle, bulk email/SMS broadcast, internal chat.

### 2.12 Public Website / Front CMS
- Controllers: `controllers/Welcome.php` (1715 lines), `controllers/admin/Frontcms.php`
- Tables: `front_cms_pages`, `front_cms_page_contents`, `front_cms_menus`, `front_cms_menu_items`, `front_cms_programs`, `front_cms_program_photos`, `front_cms_settings`, `front_cms_media_gallery`, `contents`, `content_for`, `content_types`
- Features: CMS with pages, menus, programs/news, media gallery, theme picker, Google Analytics, social links, contact-us form, complaint form, online admission form, public exam-result lookup.

### 2.13 Reports
- Controllers: `controllers/admin/Report.php`, `controllers/admin/Attendencereports.php`, `controllers/admin/Balancefees.php`, `controllers/admin/Financereports.php`, `controllers/admin/Transaction.php`
- Features: fee collection by class/date/month, balance fees, attendance reports, income/expense reports with 11 date-range presets, transaction log, user-log, audit trail.

### 2.14 System / Settings
- Controllers: `controllers/admin/Schsettings.php`, `controllers/admin/Language.php`, `controllers/admin/Currency.php`, `controllers/admin/Customfield.php`, `controllers/admin/Systemfield.php`, `controllers/admin/Sidemenu.php`, `controllers/admin/Roles.php`, `controllers/admin/Adminuser.php`, `controllers/admin/Module.php`, `controllers/admin/Addons.php`, `controllers/admin/Audit.php`
- Tables: `sch_settings` (~150 columns including 60+ field-visibility booleans), `sessions`, `classes`, `sections`, `class_sections`, `school_houses`, `categories`, `languages`, `currencies`, `custom_fields`, `sidebar_menus`, `sidebar_sub_menus`, `roles`, `roles_permissions`, `staff_roles`, `permission_group`, `permission_category`, `permission_student`, `addons`, `addon_versions`, `logs`, `userlog`, `print_headerfooter`, `filetypes`
- Features: school settings with ~150 boolean field-visibility toggles, multi-session management, custom-field builder (`libraries/Form_builder.php`), sidebar-menu builder, role and permission matrix editor, addon manager, audit log viewer, language phrase editor.

### 2.15 Authentication & Account
- Controllers: `controllers/Site.php` (admin login, userlogin, forgotpassword, resetpassword, logout), `controllers/admin/Admin.php::changepass()`, `controllers/user/User.php`
- Features: separate admin/staff vs student/parent login pages; optional captcha; optional Google Authenticator 2FA (addon); login-token rotation on each user login (`users.login_token = bin2hex(random_bytes(32))`); forgot-password via encrypted verification code email; CSRF token in forms.

### 2.16 Backup & Maintenance
- Controllers: `controllers/admin/Admin.php::backup()`, `controllers/Cron.php::autobackup()`, `hooks/maintenance_hook.php`
- Features: manual and scheduled DB-dump backups to `./backup/database_backup/`; maintenance mode gated by IP allow-list; student panel "feature lock" with grace period.

### 2.17 Online Admission & Payments
- 28 gateway controllers in `controllers/onlineadmission/` (Stripe, Razorpay, PayU, PayPal, CCAvenue, Instamojo, SSLCommerz, Paystack, Flutterwave, Mollie, Midtrans, Paytm, Toyyibpay, Skrill, Twocheckout, Payhere, Payfast, Cashfree, Billplz, DPO Pay, Jazzcash, MomoPay, Onepay, Kowri, IPayAfrica, Pesapal, Walkingm, Checkout)
- 6 "instant payment notification" (gateway_ins) controllers for INS callbacks: Ihela, Payfast, Payhere, Skrill, Toyyibpay, Twocheckout
- Generic Webhook for Instamojo in `controllers/Webhooks.php::insta_webhook()` (validates MAC via HMAC-SHA1)
- Stripe flow uses Payment Intents API

### 2.18 Advertised addons (from `addons` seed; all `current_stage=0` = "not yet purchased")
| # | Addon | Slug | Purpose |
|---|---|---|---|
| 1 | Smart School CBSE Examination | sscbse | CBSE marksheets |
| 2 | Two Factor Authentication | sstfa | Google Authenticator |
| 3 | Multi Branch | ssmb | Multi-tenant |
| 4 | Behaviour Records | ssbr | Incident scoring |
| 5 | Online Course | ssoclc | Paid video/document courses |
| 6 | Zoom Live Class | sszlc | Zoom integration |
| 7 | GMeet Live Class | ssglc | Google Meet integration |
| 8 | QR Code Attendance | ssqra | Self-scan attendance |
| 9 | Quick Fees Create | ssqfc | One-click fee assignment |
| 10 | Thermal Print | sstpa | POS-style receipt printer |
| 11 | WhatsApp Messaging | sswm | WhatsApp Business API |

---

## 3. Business Workflows (end-to-end)

### 3.1 Online Admission with Payment
1. Prospective student visits `welcome/admission`.
2. Form is rendered using `sch_settings` field-visibility flags. CAPTCHA is optional.
3. On submit, a record is created in `online_admissions` with status "pending" and a `reference_no`.
4. If `sch_settings.online_admission_payment` is on and `online_admission_amount > 0`, applicant is redirected to the active gateway controller (e.g. `onlineadmission/Stripe::index()`).
5. Gateway collects payment; on success the gateway controller calls `onlinestudent_model->paymentSuccess($gateway_response)` which writes `online_admission_payment`, updates `online_admissions.status`, and inserts a row into `gateway_ins`/`gateway_ins_response` for audit.
6. `Mailsmsconf::mailsms('online_admission_fees_submission', $sender_details)` dispatches configured channels.
7. Admin opens the application and either admits (converts to real student in `students`) or rejects.

### 3.2 Fee Collection (in-person)
1. Accountant opens `studentfee/getcollectfee/{student_session_id}`.
2. System loads `student_fees_master` for the current session and lists all `fee_groups_feetype` rows for the student's class.
3. Accountant enters amount, payment mode, date, optional discount, optional fine.
4. `Studentfee::addstudentfee()` writes a row in `student_fees` and appends a JSON object to `student_fees_deposite.amount_detail`.
5. `fee_receipt_no` is incremented atomically.
6. `Mailsmsconf::mailsms('fee_submission', …)` fires.
7. Receipt PDF is generated using `views/studentfee/printFeesByName.php` + `backend/fee_receipt_pdf_style.css` + `print_headerfooter` template.

### 3.3 Daily Attendance (faculty → cron)
1. Class teacher navigates `admin/stuattendence` and marks attendance type per student.
2. `Stuattendence_model->add()` bulk-inserts rows in `student_attendences`.
3. A row in `class_section_times` defines the deadline time for that class-section.
4. Cron `Cron::student_attendance($secret_key)` runs periodically; if any class_section has not submitted by `class_section_times.time`, the system auto-inserts `attendence_type_id=4` (Absent) for every enrolled student.
5. `Mailsmsconf::sendAbsentAttendance(...)` is fired (if enabled).

### 3.4 Examination → Marksheet
1. Admin creates an `exam_group` with `exam_type`.
2. Admin creates `exams` within the group and an `exam_group_class_batch_exams` row.
3. Teachers enter marks via `admin/mark`.
4. `examresult_model` aggregates and computes grades via `grades` table.
5. `Marksheet_model` + `template_marksheets` + `print_headerfooter` render the printable marksheet PDF.
6. Optionally `mailsms('email_pdf_exam_marksheet', …)` emails a tokenised PDF download link.

### 3.5 Addon Installation (in-app)
1. Super Admin uploads a ZIP via `admin/addons` form.
2. `Addons::index()` moves the ZIP to `./addons/{name}.zip` and extracts to `./addons/temp/{name}`.
3. `Addons::check_valid_zip()` requires either `installer/` or `updater/` sub-folder.
4. On success, ZIP is re-extracted to `./addons/{name}`.
5. `addon_info.json` is read via `read_json()`.
6. `addons_model::addAddonSetupDetails()` updates `addons` and inserts rows in `addon_versions`.
7. The installer's SQL file is executed via `execute_sql()` which uses **mysqli `multi_query()`** — note that this is a code-execution surface.
8. Addon controllers are auto-loaded via `MY_Controller::__construct()` (which calls `spl_autoload_register('my_autoloader')` from `config/routes.php` for any class beginning with `MY_Addon_`).

### 3.6 Notification dispatch (mailsmsconf)
1. Anywhere in code: `$this->mailsmsconf->mailsms($event_key, $sender_details)`.
2. `mailsmsconf` looks up `notification_setting` where `type = $event_key`.
3. Per-channel (`is_mail`, `is_sms`, `is_whatsapp`, `is_notification`) it builds a message from the `template` field with variable substitution.
4. Recipients are determined by `is_student_recipient` / `is_guardian_recipient` / `is_staff_recipient` plus the `$sender_details` payload.
5. SMS goes through `Smsgateway::sendSMS()` which dispatches to whichever `sms_config.type` row has `is_active='yes'`.
6. Email goes through `Mailer` (PHPMailer) using `email_config` (SMTP, SES).
7. Push goes through `Pushnotification::send()` → Firebase FCM v1 with OAuth service-account token.
8. Read receipts for in-app notifications are stored in `read_notification`.

### 3.7 Forgot Password (staff)
1. Staff enters email on `site/forgotpassword`.
2. `Site::forgotpassword()` looks up `staff` by email, generates `verification_code = enc_lib->encrypt(uniqid(mt_rand()))`, persists it.
3. `mailsms('forgot_password', …)` sends a reset link of form `site/resetpassword/{role}/{verification_code}`.
4. On click, `Site::resetpassword($role, $verification_code)` validates, presents new-password form.
5. New password is hashed via `enc_lib->passHashEnc($password)` → `password_hash($password, PASSWORD_DEFAULT)`.

---

## 4. UI / UX Analysis

### Navigation
- Admin panel uses AdminLTE-style sidebar (`views/layout/sidebar.php`) where menus are **dynamic from DB** (`sidebar_menus`, `sidebar_sub_menus` with `access_permissions`, `permission_group_id`, `addon_permission`, `activate_controller`, `activate_methods`, `lang_key`, `icon`).
- A top search form in the sidebar lets admin search students by name.
- A top-bar renders theme settings from JS (`themeSettings.theme_color`, `theme_background`, `theme_navigation`).
- User panel (`views/layout/student/`) has a separate, simplified nav.
- Front-end site uses `views/front/` plus themes from `backend/themes/front-main.css`.

### Dashboards
- `views/admin/dashboard.php` renders: unread notifications, fee collection (current month + month-by-month chart), expenses, total students/teachers, role counts, MySQL version & `sql_mode` warning (specifically flags `ONLY_FULL_GROUP_BY` as incompatible), and SaaS storage-quota alerts (rendered but always empty because `SaasValidation::applicationQuotas()` returns TRUE).

### Forms
- All forms use CodeIgniter's `form_validation->set_rules(...)` with extensive callbacks.
- File uploads enforce MIME/extension/size via `config/app-config.php`.
- "Custom field" rendering is handled by `libraries/Form_builder.php` (40 KB).
- Date pickers, select2, dropify, sweet-alert, fullcalendar are bundled in `backend/`.

### Tables & Filters
- Server-side processing via `libraries/Datatables.php` (24 KB).
- Common date-range filter presets in `config/payroll.php → $config['search_type']` (11 presets).
- AJAX pagination via `libraries/Ajax_pagination.php`.

### Detail pages
- Student detail: tabs for profile, fees, attendance, exams, documents, timeline, transport, hostel.
- Staff detail: timeline.
- Fee detail: paid/dues/discounts/fines.

### Responsive design
- Bootstrap grid; admin theme has light/dark mode, RTL toggle, sidebar collapse toggle, container-fluid toggle, shadow toggle — all configurable in `sch_settings` and editable live from the top bar.
- 6 preset themes: `bold_blue`, `darkgray`, `default`, `material_pink`, `shadow_white`, `yellow`.

### Useful interaction patterns
- Encrypted share-links for parent PDF download using `enc_lib->encrypt()` on the tuple.
- "Lock panel" — admin can force-redirect student users to the fees page only.
- Login-token rotation: each successful student/parent login sets `users.login_token = bin2hex(random_bytes(32))` so other sessions on the same account can be invalidated.

---

## 5. Technical Architecture

### Layered MVC
- **System core**: CodeIgniter 3.
- **Application core**: `application/core/MY_Controller.php` defines `MY_Controller`, `Admin_Controller`, `Student_Controller`, `Studentgateway_Controller`, `Public_Controller`. `MY_Model` adds `log()` for audit trail and a teacher-class-section scope filter.
- **Controllers**: 35 root + 110 admin + 27 user + 28 onlineadmission + 6 gateway_ins = ~206 controller files.
- **Models**: 155 models, all extending `MY_Model`.
- **Libraries**: ~60 libraries, including very large ones — `Smsgateway.php` (138 KB, 3145 lines), `Customlib.php` (90 KB, 2493 lines), `Mailsmsconf.php` (79 KB), `Form_builder.php` (40 KB), `Datatables.php` (24 KB), `Mailgateway.php` (25 KB).
- **Helpers**: `addon_helper.php`, `csv_helper.php`, `custom_helper.php`, `customfield_helper.php`, `json_output_helper.php`, `lang_helper.php`, `menu_helper.php`, `mime_helper.php`.

### Routing (`config/routes.php`)
- `default_controller = welcome/index`
- Front-CMS URL rewriting: `page/{slug} → welcome/page`, `read/{slug} → welcome/read`, `online_admission → welcome/admission`, `examresult → welcome/examresult`, `cbseexam → welcome/cbseexam`, `annual_calendar → welcome/annual_calendar`.
- Token-based PDF download routes: `download-receipt/{token}`, `download-marksheet/{token}`, `download-exam-marksheet/{token}`.
- Per-role reset-password routes.
- Cron: `cron/{key} → cron/index/{key}`.
- An `spl_autoload_register` in `routes.php` lazily loads any class starting with `MY_Addon_` from `application/core/` — this is the addon controller autoloader.

### Services & integrations
- **Firebase Cloud Messaging** (`libraries/Pushnotification.php`): uses Google Auth `ServiceAccountCredentials` to mint an OAuth bearer token from `application/third_party/firebase_notification_key.json` and POSTs to FCM v1.
- **AWS S3** (`libraries/Aws3.php`, `libraries/Aws_mail.php`) — alternative storage & mail via AWS SDK.
- **Zoom** API (`controllers/App::zoom()`) — uses a **hard-coded bearer token** `sY6xc8tAS7Wj8-MXyXxheg` in source. **Credential-leak finding**.
- **Omnipay** (`application/third_party/omnipay/vendor/autoload.php`) for payment abstractions.
- **PHPMailer** in `application/libraries/PHPMailer/`.
- **Google Drive** integration (`google_drive_setting` table).

### Background jobs (`controllers/Cron.php`)
All cron entry points require the `cron_secret_key` stored in `sch_settings.cron_secret_key` to be passed in the URL. Cron routes:
- `cron/{key}` → `index()` runs **autobackup + feereminder + eventreminder + schedulesmsemails** sequentially.
- `cron/student_attendance/{key}` → marks absent for un-submitted class-sections.
- `Cron::autobackup()` writes a `.sql` dump to `./backup/database_backup/`.

### Events & dispatch table
Events are not OOP events but a flat lookup table `notification_setting(type, is_mail, is_sms, is_whatsapp, is_notification, template, …)`. Event keys are hard-coded in `config/mailsms.php → $config['mailsms']` (24 events).

### State / transactions
- CodeIgniter transactions are used in models for multi-step writes.
- `MY_Model::log()` writes audit entries into `logs` after every insert/update/delete on most models.

### Mobile API
- `controllers/App.php::index()` returns the configured `mobile_api_url`, app colours, language list.
- Mobile app authentication is via `users_authentication` table: `users_id, token, staff_id, expired_at`.

---

## 6. Database & Domain Model

### General characteristics
- 196 tables created in `controllers/install/database.sql` (9,064 lines).
- Engine: InnoDB. Charset: `utf8mb3` / `utf8mb3_general_ci` (**not** full `utf8mb4` — emoji may be lost).
- All tables have `created_at` and `updated_at` timestamps.
- Lifecycle fields: `is_active` (varchar sometimes, int sometimes — inconsistent), `verification_code`, `disable_at`, `is_alumni`, `is_leave`.
- Tenant ownership: none. All data belongs to a single school.

### Core entities & relationships
```mermaid
erDiagram
    sessions ||--o{ student_session : "enrolls"
    students ||--o{ student_session : "per-year"
    classes ||--o{ class_sections : "has"
    sections ||--o{ class_sections : "in"
    class_sections ||--o{ student_session : "places"
    student_session ||--o{ student_fees_master : "billed"
    fee_groups_feetype ||--o{ student_fees_master : "assigns"
    student_fees_master ||--o{ student_fees_deposite : "pays"
    student_session ||--o{ student_attendences : "daily"
    staff ||--o{ staff_roles : "assigned"
    roles ||--o{ staff_roles : "via"
    roles ||--o{ roles_permissions : "grants"
    permission_category ||--o{ roles_permissions : "on"
    students }o--|| users : "login"
    staff }o--|| users : "login (some)"
    exam_groups ||--o{ exams : "contains"
    exam_group_class_batch_exams ||--o{ exam_group_exam_results : "yields"
    student_session ||--o{ exam_group_exam_results : "scores"
    homework }o--|| class_sections : "for"
    homework ||--o{ homework_evaluation : "evaluated"
    onlineexam ||--o{ onlineexam_attempts : "attempts"
    addon_versions }o--|| addons : "tracks"
```

### Key tables (detailed)

#### `users` (student & parent login)
Columns: `id`, `user_id`, `username` (varchar 50), **`password` varchar(50)** (PLAIN-TEXT — see §7), `childs` (text, JSON list of child IDs for parents), `role`, `lang_id`, `currency_id`, `verification_code`, `login_token` (varchar 255, rotated), `is_active`, `created_at`, `updated_at`.

#### `staff` (~50 columns)
Personal, contact, employment (joining/leaving, EPF, basic_salary, payscale, contract_type, shift, location), bank, social, documents, `password` (varchar 250, stores bcrypt hash), `verification_code`, `disable_at`, `is_active`, `created_at`, `updated_at`. Linked to `users` via `user_id`.

#### `students` (~50 columns)
Personal/guardian/contact/demographic data: `admission_no`, `roll_no`, `firstname/middlename/lastname`, `dob`, `gender`, `category_id`, `school_house_id`, `hostel_room_id`, `adhar_no`, `samagra_id`, `bank_account_no`, `ifsc_code`, `guardian_*`, `father_*`, `mother_*`, `rte`, `previous_school`, `height`, `weight`, `image`, `parent_id`, `is_active`.

#### `student_session` (per-year enrollment)
`session_id, student_id, class_id, section_id, hostel_room_id, vehroute_id, route_pickup_point_id, transport_fees, fees_discount, is_leave, is_active, is_alumni, default_login`. Most important table for scoping any per-year query.

#### `fee_groups_feetype` (the fee assignment)
`fee_session_group_id`, `fee_groups_id`, `feetype_id`, `session_id`, `amount`, `fine_type` (`none|percentage|fixed|per_day`), `due_date`, `fine_percentage`, `fine_amount`, `fine_per_day`, `is_active`. Drives fine calculation.

#### `student_fees_deposite` (the receipts)
`student_fees_master_id`, `fee_groups_feetype_id`, `student_transport_fee_id`, `amount_detail` (TEXT — serialized JSON of payment history), `is_active`. Each payment appends a JSON object.

#### `sch_settings` (the global config table)
~150 columns: school identity, currency, date/time format, language, RTL, theme, attendance type & low-attendance limit, online-admission, admission/staff-id auto-numbering, 60+ field-visibility booleans, app colours & API URL, cron secret, maintenance mode, feature lock + grace period, WhatsApp range/hours per panel, `superadmin_restriction`, `saas_key`.

#### `notification_setting` (the comms matrix)
One row per event key (24 events). Columns: `type`, `is_mail`, `is_sms`, `is_whatsapp`, `is_notification`, display-flags, `subject`, `template_id`, `whatsapp_template_id`, `template` (longtext with `{{var}}` placeholders), `variables` (CSV list).

#### `addons` / `addon_versions`
- `addons`: 21 columns of metadata (`product_id`, `name`, `config_name`, `short_name`, `directory`, `current_version`, `current_stage` (0=bought, 1=folder-available, 2=installed), `installation_by`, `last_update`, `product_order`).
- `addon_versions`: ordered version list per addon for incremental updater logic.

---

## 7. Authentication & Authorization

### User types
1. **Staff (admin-side login)** — record in `staff` joined 1:1 to `users` via `user_id`. Login handled in `Site::login()` → `Staff_model::checkLogin()`. Password hashed with `password_hash(PASSWORD_DEFAULT)` and verified with `password_verify()` via `Enc_lib::passHashDyc()`.
2. **Student** — record in `students`, joined to `users` where `users.role='student'` and `users.user_id = students.id`.
3. **Parent** — record in `users` where `role='parent'`, joined to `students` via `students.parent_id = users.id`. The `users.childs` field stores a JSON list of student IDs.
4. **Guest** — `role == 'guest'` recognised in `Auth::is_logged_in_user()` (used for online-admission preview).

### Roles (`roles`)
Six system roles seeded. `is_system = 1` prevents deletion; `is_superadmin = 1` only on role id 7. Custom roles can be created via `admin/roles`.

### Permissions
- **Staff** (`permission_group` → `permission_category` → `roles_permissions`): each `(role_id, perm_cat_id)` row has 4 booleans: `can_view`, `can_add`, `can_edit`, `can_delete`. `Rbac::hasPrivilege($cat, $perm)` checks these. **Super Admin bypasses all checks**.
- **Student/Parent** (`permission_student`): toggles per group_id for student & parent.
- Module-level gating uses `permission_group.is_active` and `Module_lib::hasActive($short_code)`.

### Tenant isolation
**None in this build.** The `Multi Branch` addon (id 3 in `addons` seed) is advertised as the way to add multi-school isolation, but is not installed. Without it, all data belongs to one school.

### Record-level access
- `MY_Model::applyTeacherClassSectionFilter()` restricts teacher queries to their assigned class-sections when `sch_settings.class_teacher == 'yes'` and the staff is a Teacher role (role id 2).
- `sch_settings.superadmin_restriction` controls whether Super Admin rows are visible to non-super-admin staff.

### Security mechanisms
- CSRF token rendered in every form via `customlib::getCSRF()`.
- Captcha library per-page configurable.
- Optional Google Authenticator 2FA (addon).
- Login-token rotation on every student/parent login.
- `Auth::is_logged_in()` also re-checks `staff.is_active == 1` on every request.

### **CRITICAL SECURITY FINDING — plain-text student/parent passwords**
`models/User_model.php` lines 93 and 111:
```php
if ($query->num_rows() == 1 && $result[0]->password == $data['password']) { return $result; }
```
And `User_model::checkLoginParent()` line 137:
```php
$this->db->where('password', ($data['password']));
```
The `users` table schema confirms `password varchar(50)` — far too short for any modern hash. The column type and the equality comparison both indicate that **student and parent passwords are stored in plain text**. The `User_model::checkOldPass()` (line 221) and `::saveNewPass()` (line 245) confirm the same plain-text pattern. **Most severe security issue in the codebase.**

### Hardcoded credentials & crypto keys
- `Enc_lib` AES-256-CBC uses **hardcoded** `$pub_key = 'ss@pubkey'` and `$pvt_key = 'ss@pvtkey'` — anyone with the source can decrypt any `enc_lib->encrypt()` payload (verification codes, share-content tokens, fee-receipt tokens, marksheet tokens).
- `controllers/App::zoom()` uses hardcoded `Authorization: Bearer sY6xc8tAS7Wj8-MXyXxheg` Zoom API key.

---

## 8. Integrations & Automation

### External services (verified in source)

| Category | Providers |
|---|---|
| SMS | Clickatell, Twilio, MSG91, SMS Country, TextLocal, Bulk SMS, MobiReach, Nexmo, Africa's Talking, SMS EG, Custom |
| Email | SMTP (PHPMailer), AWS SES, Mailgun-style via Omnipay |
| Push | Firebase Cloud Messaging v1 (OAuth service-account) |
| Payments | 28 gateways in `controllers/onlineadmission/` + `controllers/gateway_ins/` INS receivers |
| Live class | Zoom (`App::zoom()`), Google Meet (addon-gated) |
| Biometric | Device push API — JSON `{serial_number, user_id, t}` validated against `sch_settings.biometric_device` CSV |
| Cloud storage | AWS S3, Google Drive |
| Map | `pickup_point.latitude/longitude` columns exist |
| Analytics | `front_cms_settings.google_analytics` rendered on public site |

### Webhooks
- `Webhooks::insta_webhook()` — Instamojo payment status receiver. Verifies MAC via `hash_hmac("sha1", implode("|", $data), $salt)`. Inserts a row in `gateway_ins_response`.
- The `gateway_ins` controllers are server-to-server callback receivers for gateway-side notifications.

### Automation (cron)
- `Cron::index($key)` — runs four jobs in sequence: `autobackup`, `feereminder`, `eventreminder`, `schedulesmsemails`.
- `Cron::student_attendance($key)` — auto-marks absent.

### Retry & error handling
- SMS gateway calls **swallow exceptions**: `catch (Exception $e) { return true; }` in `Smsgateway::sendSMS()` for Clickatell and `if ($response->IsError) { return true; }` for Twilio — both return `true` even on failure. **SMS delivery failures are not surfaced**.
- Email uses PHPMailer exceptions and proper SMTP error codes.
- Addon installer uses `multi_query()` which can leave the DB in an inconsistent state on partial failure; `execute_sql()` returns a status array but the caller only logs to `addons/error_log.txt` and does not roll back.

---

## 9. AI Capabilities

**There are no AI capabilities implemented in this project.**

Verification:
- `grep -rli "openai|chatgpt|gpt-|anthropic|claude|gemini|ai_assist|ai_generate"` over `controllers/`, `libraries/`, `models/`, `helpers/` returned **zero results**.
- The only `ai_` matches in the codebase were inside language translation strings for unrelated words containing the substring "ai".
- The `questions` table supports `question_type ∈ {singlechoice, multichoice, true_false, descriptive}` but there is **no question-generation, automated marking of descriptive answers, or AI-powered recommendation** anywhere.
- Online exam descriptive answers are scored by `answer_word_count` only. No semantic scoring.

This is a fully **rule-based** system.

---

## 10. SaaS Architecture & Monetization

### Current state
- `config/saas-config.php`: `$config['saas_enabled'] = false;`
- `libraries/SaasValidation.php` — every method returns `TRUE` unconditionally.
- `libraries/ResourceQuota.php` — empty class.
- The dashboard view iterates over `$storageused` (an empty array in this build) to render quota alerts — the UI is there, but no data backs it.

### Licensing / monetization actually implemented
1. **Envato purchase-code verification** — `Admin_Controller::check_license()` reads `config/license.php → $config['SSLK']` and validates it against an AES-derived string of the install's `base_url()` via `Aes::validchk('encrypt', base_url())`. If the regex matches and the substring matches, the license is valid; otherwise the license string is wiped from `license.php`.
2. **Add-on purchases** — separate CodeCanyon product IDs (`addons.product_id`). Verified by `Auth::addon_update_check()` which posts to the vendor's update-check API.
3. **Mobile app** — bundled separately, uses `mobile_api_url` from `sch_settings`.

### Multi-tenancy
- **Not implemented** in this build. The Multi-Branch addon (id 3 in `addons` seed) is advertised as the mechanism, but is not installed.
- The `branch_cookie` deleted in `Auth::logout()` and `sch_settings.base_url` / `folder_path` columns hint at multi-branch infrastructure being prepared but not active.
- The `saas_key` field on `sch_settings` is present but unused in any verified code path.

### Plans, entitlements, credits, billing
**None present.** No `plans`, `subscriptions`, `credits`, `invoices` (for the SaaS itself) tables exist.

---

## 11. Strengths, Weaknesses & Risks

### Strengths
1. **Comprehensive feature set in one codebase** — most K-12 operational needs covered out of the box.
2. **Highly configurable** — `sch_settings` has 60+ field-visibility toggles, plus per-event channel toggles in `notification_setting`.
3. **RBAC + module system** is consistent — every admin controller method calls `Rbac::hasPrivilege()` and `module_lib->hasActive()` at the top.
4. **Audit log** — `MY_Model::log()` writes to `logs` on every CUD operation.
5. **Mobile API ready** — token-based auth in `users_authentication`, FCM push configured.
6. **Multi-language (78)**.
7. **Backup automation** — built-in DB-dump cron with secret-key gating.
8. **24-channel-event comms matrix** is a clean abstraction.
9. **Tokenised PDF links** — encrypted, shareable receipt/marksheet download URLs.
10. **Theme system** — 6 themes + live toggle.
11. **Encrypted share-content links** for distributing uploaded material to parents.

### Weaknesses
1. **CRITICAL: plain-text student/parent passwords** (`users.password varchar(50)`, compared with `==` in `User_model::checkLogin`).
2. **Hardcoded AES keys** in `Enc_lib`.
3. **No multi-tenancy** — single-DB, single-school. The `saas_enabled` flag is a stub.
4. **No input parameterised queries in places** — `Module_model::getPermissionByModulename()` does string concatenation.
5. **SMS error handling swallows failures**.
6. **utf8mb3 charset** — limits emoji/astral-plane characters.
7. **Inconsistent schema typing** — `is_active` is `varchar(255)` in many tables, `int` in others.
8. **~196 tables, many with TEXT-stored JSON** — not queryable, hard to migrate.
9. **Single-DB payment settings** — only one gateway can be `is_active='yes'` at a time.
10. **Old framework** — CodeIgniter 3 (EOL October 2025). No upgrade path to CI4.
11. **Hardcoded Zoom bearer token** in `controllers/App::zoom()`.
12. **No proper queue** — cron is a single HTTP hit running 4 jobs sequentially.
13. **`ONLY_FULL_GROUP_BY` incompatibility** detected at runtime but not refactored.
14. **Heavy fat-controller pattern** — `Welcome.php` 1715 lines, `Site.php` 1003, `Customlib.php` 2493, `Smsgateway.php` 3145, `Mailsmsconf.php` 78KB.
15. **Addon installer executes arbitrary SQL via `multi_query()`** — security risk.

### Security issues (summary)
| Severity | Finding | Evidence |
|---|---|---|
| Critical | Plain-text student/parent passwords | `models/User_model.php` lines 93, 111, 137, 224; `users.password varchar(50)` in schema |
| High | Hardcoded AES keys | `libraries/Enc_lib.php` (`ss@pubkey` / `ss@pvtkey`) |
| High | Hardcoded Zoom API bearer token | `controllers/App::zoom()` |
| High | Addon installer executes raw multi_query SQL | `helpers/addon_helper.php::execute_sql()` |
| Medium | SQL string concatenation in `Module_model::getPermissionByModulename()` | `models/Module_model.php` line 80 |
| Medium | SMS failures silently swallowed | `libraries/Smsgateway.php` lines 62, 80 |
| Medium | AES-derived license check is reversible | `core/MY_Controller.php::check_license()` |
| Medium | `users_authentication` table stores tokens without hashing | `users_authentication` schema |
| Low | utf8mb3 collation | throughout `database.sql` |
| Low | Inconsistent `is_active` typing | throughout `database.sql` |

### Scalability concerns
- The `student_fees_deposite.amount_detail` JSON-in-TEXT pattern works for a 1k-student school but degrades at 50k+ students.
- No read replicas, no caching layer (`cache_on = false`).
- `logs` table grows unbounded (no archival).
- No queue for email/SMS — bulk sends block the request thread.

### Technical debt
- "Multi-class `_create()`" methods left as stubs (`Student::____________________________________create()`).
- `libraries/Version_control.php` documents deprecated PHP-version checks but is still loaded.
- The whole `application/controllers/install/` directory should be deleted after install but is shipped.
- Mixed coding style.
- Heavy duplication: every SMS provider branch in `Smsgateway::sendSMS()` is copy-pasted 9 times.

---

## 12. Reusable Product Concepts (for an Education-Management SaaS)

### 12.1 Per-event × per-channel × per-recipient notification matrix
**Why it matters:** Schools need fine-grained control over what gets sent where.
**How Smart School does it:** One `notification_setting` row per event key with booleans for each (channel × recipient-class) combination, plus a `{{var}}`-templated message and a `variables` CSV.
**Implementation consideration:** In a modern SaaS, model this as `NotificationTemplate(event, channel, recipient_role, locale, body, dlt_template_id, whatsapp_template_id, active)`. Add per-user opt-out flags. Use a job queue per channel.

### 12.2 Per-year enrollment ("student_session") as the unit of scope
**Why it matters:** Almost every report is scoped to an academic year. Promoting students to the next class should not lose history.
**How Smart School does it:** `student_session` is a per-year record linking `student → session → class → section → hostel → transport route → pickup point`. All queries filter by `session_id = current_session`.
**Implementation consideration:** Use a composite `(student_id, academic_year_id)` foreign key on every operational table. Provide a "promote students to next session" admin action.

### 12.3 Granular per-field visibility on forms
**Why it matters:** Different schools collect different fields.
**How Smart School does it:** `sch_settings` has ~60 boolean columns that gate form fields. Brittle (schema change per field) but conceptually right.
**Implementation consideration:** Replace with a `FieldDefinition(entity, key, type, label, required, visible, default)` table driven by a JSON-Schema or Form-Builder library.

### 12.4 Tokenised shareable document links
**Why it matters:** Schools email parents PDF receipts/marksheets; they don't want to attach large files but also don't want a public URL.
**How Smart School does it:** `enc_lib->encrypt(implode(",", [...]))` becomes the URL token.
**Implementation consideration:** Use **signed JWTs** with a short expiry instead of reversible AES. Bind the token to a specific document + tenant + expiry + intent. Log every download.

### 12.5 Hierarchical exam group → exam → schedule → marks → result
**Why it matters:** Real schools have "Term 1 → Unit Test 1 → Math → 80 marks → result".
**How Smart School does it:** Five-level hierarchy `exam_groups → exams → exam_group_class_batch_exams → exam_group_class_batch_exam_subjects → exam_group_exam_results`.
**Implementation consideration:** In a modern schema, use `Assessment → AssessmentComponent → AssessmentComponentSubject → StudentScore`. Add a `grading_scheme` reference table for the 5 grading types.

### 12.6 Auto-numbering with prefix + digit length
**Why it matters:** Admission numbers follow school-specific formats.
**How Smart School does it:** `sch_settings.adm_prefix`, `adm_no_digit`, `adm_start_from`, `adm_auto_insert` flag.
**Implementation consideration:** Generalise into a `Sequence(tenant_id, entity, prefix, suffix, digits, next_value, padding_char, reset_per_year)` table. Use SQL `SELECT … FOR UPDATE` to allocate atomically.

### 12.7 Module / addon entitlements gate sidebar + controllers
**Why it matters:** SaaS monetisation often needs per-plan feature gating.
**How Smart School does it:** `permission_group.short_code` is a module key; `Module_lib::hasActive()` checks it.
**Implementation consideration:** Use a `FeatureFlag(plan_id, feature_key, enabled)` table backed by a feature-service with caching. Inject a `feature(flag)` helper in templates and a `FeatureMiddleware` in controllers.

### 12.8 Per-class-section schedule deadline for attendance
**Why it matters:** If a teacher forgets to mark attendance by 10 AM, the school needs the system to auto-mark absent.
**How Smart School does it:** `class_section_times.time` is the deadline; cron `Cron::student_attendance()` queries and bulk-inserts absent records.
**Implementation consideration:** Schedule as a delayed job per class-section, posted when the teacher opens the attendance form. Eliminates the global cron.

### 12.9 Audit log with role-context, IP, platform, agent
**Why it matters:** Schools need compliance (FERPA / GDPR).
**How Smart School does it:** `MY_Model::log()` writes to `logs(message, record_id, user_id, action, ip_address, platform, agent, time)`.
**Implementation consideration:** Use structured event logging (OpenTelemetry logs → Loki / Elasticsearch) with searchable JSON.

### 12.10 Encrypted, time-windowed share-content links
**Why it matters:** Teachers distribute study material; links must be revocable.
**How Smart School does it:** `share_contents` + `share_content_for` + `share_upload_contents` tables, with `Site::download_content($share_id, $content_id)`.
**Implementation consideration:** Same as 12.4 — use signed URLs.

---

## 13. Evidence & Confidence

### Confidence summary
| Category | Verified in source | Documented but not verified | Verified in live demo | Advertised only | Not verified |
|---|---|---|---|---|---|
| Feature inventory | ✅ | n/a | ❌ | n/a | n/a |
| Business workflows | ✅ | n/a | ❌ | n/a | n/a |
| UI/UX | ✅ | n/a | ❌ | n/a | Live interaction |
| Architecture | ✅ | n/a | ❌ | n/a | Runtime behaviour |
| DB schema | ✅ | n/a | ❌ | n/a | Index performance |
| AuthN/AuthZ | ✅ | n/a | ❌ | n/a | n/a |
| Integrations | ✅ | n/a | ❌ | n/a | Live gateway calls |
| AI | ✅ absence | n/a | ❌ | n/a | n/a |
| SaaS | ✅ stubbed | n/a | ❌ | n/a | n/a |
| Addon descriptions | n/a | ✅ | ❌ | ✅ | Functional correctness of addons |

### Source paths for important claims
| Claim | Evidence |
|---|---|
| 196 tables | `controllers/install/database.sql` |
| 35 root + 110 admin controllers | `application/controllers/` directory listing |
| 155 models | `application/models/` directory listing |
| Plain-text student/parent passwords | `application/models/User_model.php` lines 93, 111, 137, 224 |
| Staff passwords hashed (bcrypt) | `application/libraries/Enc_lib.php::passHashEnc()` + `models/Staff_model.php::checkLogin()` |
| Hardcoded AES keys | `application/libraries/Enc_lib.php` lines 3-4 |
| Hardcoded Zoom token | `application/controllers/App.php::zoom()` |
| SaaS disabled | `application/config/saas-config.php` + `application/libraries/SaasValidation.php` |
| License = Envato purchase code | `application/core/MY_Controller.php::check_license()` |
| 28 payment gateways | `application/controllers/onlineadmission/` directory listing |
| 11 SMS providers | `application/libraries/Smsgateway.php::sendSMS()` switch chain |
| 78 languages | `application/language/` directory listing |
| Cron jobs | `application/controllers/Cron.php` |
| FCM push | `application/libraries/Pushnotification.php` |
| Audit log | `application/core/MY_Model.php::log()` |
| Addon installer runs `multi_query()` | `application/helpers/addon_helper.php::execute_sql()` |
| No AI code | grep returned 0 hits in code |

### Limitations of this analysis
1. **No live demo was available** — package is source-only.
2. **No documentation was bundled** — `readme.txt` says "documentation: Quick Installation Guide & User's Manual" but the `documentation/` folder was not present.
3. **`smart_school_update_7.1.0_to_7.2.0/`** (delta package) was not deeply analysed.
4. **Addons are advertised but not installed** — `addons.current_stage = 0` for all rows.
5. **No automated test suite** is present.
6. **Vendor mobile apps** (Flutter) were not in the package.

---

## 14. Final Assessment

Smart School is a **mature, feature-dense, single-tenant K-12 ERP** built on CodeIgniter 3. Its competitive advantage is breadth: most K-12 operational needs are covered out of the box, configurable without code, and translatable into 78 languages.

The **most valuable capabilities** for a modern education-management SaaS to learn from are:

1. The **per-event × per-channel × per-recipient notification matrix** (`notification_setting`).
2. The **`student_session` per-year enrollment** model.
3. The **hierarchical exam-group → exam → result** model.
4. The **field-visibility toggles** + custom-field builder pattern.
5. The **tokenised shareable PDF links**.

The **weaknesses** are equally clear:

1. **Plain-text passwords for students/parents** is an inexcusable critical vulnerability.
2. **Hardcoded AES keys and Zoom tokens** make encryption and integrations security-theatre.
3. **The "SaaS" layer is a stub**.
4. **CodeIgniter 3 is end-of-life**.
5. **TEXT-stored JSON** for fee deposit history will not scale.

**Unresolved questions:**
- Does the vendor's hosted/mobile-app distribution hash passwords differently?
- Are the 11 advertised addons independently licensed and security-audited?
- Is there a migration path to utf8mb4 and CI4?
- Does the vendor's update server receive any telemetry beyond purchase code?

**Overall analysis confidence:** **High** for static code analysis; **Moderate** for behavioural claims; **Low** for the addon ecosystem. The findings about plain-text passwords, hardcoded keys, and stubbed SaaS are **certain**.

---

*End of Project 1 report.*
