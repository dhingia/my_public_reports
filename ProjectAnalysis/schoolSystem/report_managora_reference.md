# Reference Project Analysis for Managora — Smart School + Addon Bundle Pack

**Reference project:** Smart School v7.2.0 (CodeCanyon item 19426018) + Smart School Addon Modules Bundle Pack (CodeCanyon item 50933268)
**Vendor:** QDOCS (`smart-school.in`)
**Stack:** CodeIgniter 3 (PHP 7/8) + MySQL + jQuery/AdminLTE + 11 addon modules
**Analysis method:** Static source-code review only. No execution, no live demo, no dependency installation, no DB access.
**Combined scale:** 273 MySQL tables, ~286 controllers, ~205 models, ~858 language files (78 languages × 11 addons).

> This document studies the reference project to build a knowledge base for Managora. It does **not** assume anything about Managora's current implementation. Where Managora is mentioned in recommendations, the wording is **"could benefit"** or **"could adapt"** — never "Managora has/lacks X."

---

## 1. Executive Summary

The reference project is a **mature, feature-dense, single-tenant K-12 ERP** with an 11-addon ecosystem that broadens it into a near-comprehensive K-12 + LMS + multi-branch + 2FA + WhatsApp-capable platform. Its breadth is genuinely impressive — most operational needs of an Indian K-12 school, a coaching institute, or a small training centre are covered out of the box, configurable without code, and translatable into 78 languages.

The **most valuable transferable concepts** for an AI-first multi-tenant education SaaS like Managora are:

1. The **per-event × per-channel × per-recipient notification matrix** — the cleanest abstraction in the codebase.
2. The **`student_session` per-year enrollment** scoping pattern — the right way to model academic-year data.
3. The **hierarchical exam → marks → result → rank** model with multiple grading schemes and co-scholastic observations.
4. The **forward-compatible dispatcher + provider plugin** pattern — core pre-calls addon-provided gateway classes.
5. The **module-toggle + sub-feature permission matrix** as DB-seeded RBAC.
6. The **tokenised shareable PDF links** for parent distribution.
7. The **CBSE observation framework** (`parameters → subparameters → terms → per-student scores`) — generalises to any rubric-based assessment.
8. The **LMS-style course structure** with paid checkout + AWS S3 media + idempotent payment processing.
9. The **OAuth2 with refresh tokens** pattern for per-teacher third-party service connections (Zoom, Google).
10. The **TOTP 2FA** implementation (RFC 6238, `random_bytes`, FK CASCADE).

The **concepts to avoid** are equally clear:

1. **Plain-text student/parent passwords** (`users.password varchar(50)`, `==` comparison) — the single most severe issue.
2. **Multi-database multi-tenancy with plain-text DB credentials** in a directory table.
3. **`multi_query()` SQL installer** with no rollback and no sandbox.
4. **Hardcoded AES keys** for "encrypted" tokens.
5. **SMS/WhatsApp gateway failures silently swallowed** (`return true` in catch).
6. **Leaky addon boundary** where core code directly loads addon classes without a `hasActive` guard.
7. **CodeIgniter 3 EOL** (October 2025) with no upgrade path.
8. **No AI capabilities anywhere** — completely rule-based.

For Managora specifically, the highest-leverage opportunities are: (a) build the notification matrix from day one as a first-class abstraction with a per-event, per-channel, per-recipient-role, per-locale matrix; (b) make multi-tenancy a single-DB `tenant_id` column with row-level security rather than multi-database; (c) make AI a first-class citizen — the reference project has zero AI; (d) build a proper plugin/extension system with signed-JWT licensing rather than reversible AES; (e) abstract payment gateways, SMS, WhatsApp, Email, Push, and AI providers behind a unified `ChannelRegistry` so adding a new provider is a one-file change.

---

## 2. Product Overview

**Purpose:** A complete school-administration ERP for K-12, extended via 11 addons into CBSE exams, MOOC-style online courses, behaviour records, multi-branch federation, QR attendance, Zoom/GMeet live classes, TOTP 2FA, thermal printing, quick-fees installment plans, and WhatsApp messaging.

**Target users:** 6 staff roles (Admin, Teacher, Accountant, Librarian, Receptionist, Super Admin) + Student + Parent + Guest (for online-admission preview). Addons extend these roles' workflows rather than introducing new user types.

**Business model:** One-time CodeCanyon purchase per addon; bundle pack discount; URL-bound AES licensing. No subscription, no metering, no per-seat pricing. The SaaS layer in core is **stubbed and disabled** (`saas_enabled=false`, `SaasValidation` returns `TRUE` everywhere).

**Stack:**
- CodeIgniter 3, PHP 7/8, MySQL (utf8mb3 / utf8mb3_general_ci — not utf8mb4)
- jQuery + AdminLTE, Bootstrap, FullCalendar, DataTables, Dropify, Select2, SweetAlert
- mPDF for PDF generation; PHPMailer + AWS SES for email; Firebase FCM v1 for push
- 28 payment-gateway integrations; 11 SMS providers; 2 WhatsApp providers (Meta Cloud + Twilio)
- AWS S3 for media storage (Online Course addon); Google Drive integration
- 6 admin themes with live dark/light/sidebar-collapse toggles; 78 languages

**Distribution:** Core PHP source + 11 addon ZIPs that overlay the core's filesystem and execute SQL via `mysqli::multi_query()`. Addon code is loaded via `spl_autoload_register('my_autoloader')` for any class starting with `MY_Addon_`.

---

## 3. Complete Feature and Module Inventory

> Selective — focusing on capabilities relevant to Managora's domain. Each capability is annotated with **Verified in source** (V) / **Documented by vendor** (D) / **Not verified** (NV).

### 3.1 Core ERP modules (Project 1)
| Module | Key capabilities | Evidence |
|---|---|---|
| Student Information | Admission (admin + online paid form), bulk CSV import, multi-class enrollment, document upload, student timeline, transfer certificate, ID card, alumni, disable/reactivate | V — `controllers/admin/Student.php`, `controllers/admin/Onlineadmission.php` |
| Fees Collection | Fee type/group/master definition, group–type–session assignment with `fine_type ∈ {none, percentage, fixed, per_day}`, `due_date`, fine per day, discounts, back-date fee collection (configurable), partial payment, fee carry-forward between sessions, offline bank payment reconciliation, balance-fee report, fee receipt PDF, automatic receipt numbering | V — `controllers/admin/Feemaster.php`, `controllers/Studentfee.php`, tables `fee_groups_feetype`, `student_fees_deposite` |
| Attendance | Class+section daily, subject-wise, biometric push-API, half-day/late/holiday types, low-attendance limit alert, cron auto-marks-absent | V — `controllers/admin/Stuattendence.php`, `controllers/Biometric.php`, `controllers/Cron.php::student_attendance()` |
| Examinations | Hierarchical `exam_group → exam → schedule → marks → result → rank`; 5 grading systems; mark divisions; online exam with attempts/duration/negative marking/random questions/passing percentage/auto-publish/descriptive word limit; question bank; admit cards + marksheet templates | V — `controllers/admin/Examgroup.php`, `controllers/admin/Onlineexam.php`, tables `exam_group_*`, `onlineexam*`, `questions` |
| Academics | Subject grouping, class teacher subject mapping, timetable, syllabus tracking (lesson → topic → status), homework with marks/evaluation/documents, lesson-plan forum | V — `controllers/admin/Subjectgroup.php`, `controllers/admin/Syllabus.php`, `controllers/Homework.php` |
| HR / Payroll | Staff CRUD with extensive profile, payslip with allowance/deduction/leave deduction/tax/net_salary, payslip status (`generated`/`paid`/`unpaid`/`not_generate`), leave request/approval, staff timeline, staff ID card, staff rating | V — `controllers/admin/Staff.php`, `controllers/admin/Payroll.php`, tables `staff`, `staff_payslip` |
| Library | Book catalog with rack/shelf, member issuance, due-date tracking, fine calculation | V — `controllers/admin/Book.php`, tables `books`, `book_issues` |
| Hostel & Transport | Hostel & room-type allocation; vehicle registry; route + pickup-point mapping with lat/long; transport fee master | V — `controllers/admin/Hostel.php`, `controllers/admin/Route.php`, tables `hostel*`, `transport_*` |
| Inventory | Store/items/categories/suppliers/stock, item issue, dispatch & receive | V — `controllers/admin/Item.php`, tables `item*` |
| Front Office | Admission enquiries + follow-ups, visitor book, general call log, complaint register, postal dispatch/receive, reference & source master | V — `controllers/admin/Enquiry.php`, tables `enquiry`, `visitors_book`, `complaint` |
| Communication | 24 event types × 4 channels (SMS/Email/Push/WhatsApp) × 3 recipient classes (student/guardian/staff); bulk broadcast; internal chat | V — `libraries/Mailsmsconf.php`, `config/mailsms.php`, table `notification_setting` |
| Front CMS | Pages (manual/homepage/list-view), menus, programs/news with photos, media gallery, theme picker, Google Analytics, social links, contact-us form, complaint form, online admission form, public exam-result lookup, annual calendar | V — `controllers/Welcome.php` (1715 lines), `controllers/admin/Frontcms.php`, tables `front_cms_*` |
| Reports | Fee collection by class/date/month, balance fees, attendance, income/expense (11 date-range presets), transaction log, user-log, audit trail | V — `controllers/admin/Report.php`, `config/payroll.php → $config['search_type']` |
| System Settings | School settings with ~150 boolean field-visibility toggles; multi-session; class/section/house/category master; custom-field builder; sidebar-menu builder; role and permission matrix editor; addon manager; audit log viewer; language phrase editor; file-type manager | V — `controllers/admin/Schsettings.php`, `controllers/admin/Customfield.php`, table `sch_settings` (~150 columns) |
| Authentication | Separate admin/staff vs student/parent login pages; optional captcha; optional Google Authenticator 2FA (addon); login-token rotation; forgot-password via encrypted verification code email; CSRF | V — `controllers/Site.php`, `libraries/Auth.php` |
| Backup & Maintenance | Manual and scheduled DB-dump backups to `./backup/database_backup/`; maintenance mode gated by IP allow-list; student panel "feature lock" with grace period | V — `controllers/admin/Admin.php::backup()`, `controllers/Cron.php::autobackup()`, `hooks/maintenance_hook.php` |

### 3.2 Addon modules (Project 2 — 11 addons)
| # | Addon | Capabilities | Evidence |
|---|---|---|---|
| 1 | CBSE Examination (v4.0) | Term-based exam workflow, written + practical assessment types, co-scholastic observations, grade ranges, rank generation (4 strategies), marksheet + admit card templates | V — 28 tables, 14 controllers in `cbseexam/` |
| 2 | Online Course (v5.0) | MOOC LMS: course → section → lesson/quiz/exam/assignment → progress → certificate; AWS S3 media; cart-based checkout; 27 payment gateways; guest buyers | V — 29 tables, 12 admin controllers + 27 gateway controllers |
| 3 | Behaviour Records (v4.0) | Incident catalog with positive/negative points; assignment + comments; 7 report types (student/class/house/incident wise) | V — 4 tables, 6 controllers |
| 4 | Multi Branch (v4.0) | Multi-database federation: master install holds plain-text DB credentials for N branch installs; cross-branch reports | V — 1 table `multi_branch`, 2 controllers |
| 5 | QR Code Attendance (v3.0) | Self-scan via browser camera or scanner-gun; reuses existing ID-card QR codes | V — 1 table `QR_code_settings`, 3 controllers |
| 6 | Zoom Live Class (v8.0) | OAuth2 Zoom integration; per-teacher or per-school account; meeting creation, join tracking | V — 5 tables, library `Zoom_api.php` (Guzzle) |
| 7 | GMeet Live Class (v7.0) | Google Calendar API + Hangouts Meet conference data; OAuth2 offline refresh (tokens in session) | V — 5 tables, library `Google.php` |
| 8 | Two Factor Authentication (v4.0) | TOTP via Google Authenticator (RFC 6238); per-user 16-char base32 secret; QR-code setup; admin global toggle | V — 2 tables, library `Google_authenticator.php` |
| 9 | Thermal Print (v2.0) | POS-style 58/80mm fee receipts | V — 1 table `thermal_print_settings`, 1 controller |
| 10 | Quick Fees Create (v2.0) | One-click per-student fee master with installment plan builder (down-payment + N installments + fine config) | V — 0 new tables, 1 controller |
| 11 | WhatsApp Messaging (v1.0) | Meta Cloud API + Twilio WhatsApp; integrates with `notification_setting.is_whatsapp` channel | V — 1 table `whatsapp_config`, library `Whatsappgateway.php` |

### 3.3 Important settings / configurations
- **`sch_settings` table** (~150 columns): school identity, currency, date/time format, language, RTL, theme, attendance type & low-attendance limit, online-admission config, admission/staff-id auto-numbering (prefix + digit length + start-from), 60+ field-visibility booleans for student/staff forms, app colours & API URL, cron secret, maintenance mode, feature lock + grace period, WhatsApp range/hours per panel, `superadmin_restriction`, `saas_key`.
- **`notification_setting` table**: per-event × per-channel × per-recipient-class toggles + `{{var}}`-templated body + `variables` CSV list + DLT/WhatsApp template IDs.
- **`payment_settings` table**: one active gateway at a time; `gateway_mode` (Testing/Live); processing-fee config (`charge_type`, `charge_value`).

### 3.4 Administrative / operational capabilities
- **Addon manager** (`admin/addons`): ZIP upload, extract, `addon_info.json` parse, SQL execution via `multi_query()`.
- **Sidebar menu builder** (`admin/sidemenu`): drag-and-drop admin sidebar configuration.
- **Role & permission matrix editor** (`admin/roles`): create custom roles, grant `can_view/add/edit/delete` per permission category.
- **Audit log viewer** (`admin/audit`): search `logs` table.
- **Backup manager** (`admin/admin/backup`): manual DB-dump download.
- **Custom-field builder** (`admin/customfield`): add custom fields to student/staff/etc. forms.
- **Language phrase editor** (`admin/language`): edit translations per language.

### 3.5 Reporting and analytics
- Fee collection by class/date/month, balance fees, attendance, income/expense, transaction log, user-log, audit trail.
- 11 date-range presets: today/this_week/last_week/this_month/last_month/last_3_month/last_6_month/last_12_month/this_year/last_year/period.
- Multi Branch addon's `Branch::overview()` is a cross-branch dashboard with 12 KPIs per branch.
- CBSE addon has per-exam grade-distribution chart (`Chart.php`).
- Online Course addon has course sales + progress reports.
- **No business-intelligence layer** — all reports are direct SQL queries rendered as DataTables.

---

## 4. Business Workflows

> Selective — documenting the workflows most relevant to Managora's domain. Each workflow is end-to-end with records, status transitions, validations, notifications, integrations, and failure scenarios.

### 4.1 Online Admission with Payment (high-value workflow)
**Actors:** Prospective student (public visitor), Admin, Payment gateway.
**Records touched:** `online_admissions`, `online_admission_payment`, `gateway_ins`, `gateway_ins_response`, `users`, `students`.

**Steps:**
1. Visitor visits `welcome/admission`. Form rendered using `sch_settings` field-visibility flags (so each school hides/shows fields without code). CAPTCHA optional.
2. On submit (`Welcome::submitadmission()`), record created in `online_admissions` with status `"pending"` and a `reference_no`. Validations: `class_id`, `firstname`, `dob`, `section_id` required; email format if `student_email` field is enabled; unique email check via `onlinestudent_model->check_student_email_exists`.
3. If `sch_settings.online_admission_payment=1` and `online_admission_amount > 0`, redirect to the active gateway controller (e.g. `onlineadmission/Stripe::index()`).
4. Stripe flow: `create_customer()` → `create_payment_intent()` (Payment Intents API) → `insert_payment()` posts the gateway response back.
5. On success, `onlinestudent_model->paymentSuccess($gateway_response)` writes `online_admission_payment`, updates `online_admissions.status`, inserts audit rows in `gateway_ins` + `gateway_ins_response`.
6. `Mailsmsconf::mailsms('online_admission_fees_submission', $sender_details)` fires — dispatches SMS/Email/Push/WhatsApp per the `notification_setting` matrix for this event.
7. Admin reviews the application in `admin/onlineadmission/review/{reference_no}` — admit (converts to a real `students` row + `users` row with default-login credentials) or reject.

**Status transitions on `online_admissions.status`:** `pending → paid → admitted` OR `pending → paid → rejected` OR `pending → cancelled` (timeout).

**Notifications fired:** `online_admission_form_submission`, `online_admission_fees_submission`, `online_admission_fees_processing` (if gateway is async).

**Integrations:** Active payment gateway (one of 28), SMS gateway, Email (PHPMailer), Push (FCM), WhatsApp (if addon installed).

**Failure scenarios:**
- Gateway timeout: `gateway_ins_response` records the timeout; admin can manually mark paid after bank confirmation.
- Duplicate callback: idempotency check via `online_admission_payment.transaction_id` uniqueness.
- Form validation error: re-render with errors; CAPTCHA re-generated.

### 4.2 Fee Collection (in-person at counter)
**Actors:** Accountant, Student/Parent, optionally Receipt Printer.
**Records touched:** `student_fees_master`, `fee_groups_feetype`, `student_fees`, `student_fees_deposite`, `fee_receipt_no`, `student_applied_discounts`.

**Steps:**
1. Accountant opens `studentfee/getcollectfee/{student_session_id}`.
2. System loads `student_fees_master` for the current session and lists all `fee_groups_feetype` rows assigned to the student's class.
3. Accountant enters amount, payment mode (`cash`/`cheque`/`online`), date, optional discount (must already be `student_applied_discounts`-assigned), optional fine.
4. `Studentfee::addstudentfee()` validates: amount > 0; date not in future unless `sch_settings.collect_back_date_fees=1`; discount ≤ amount.
5. Writes a row in `student_fees` and **appends a JSON object to `student_fees_deposite.amount_detail`** (TEXT column) keyed by an incrementing counter: `{"1":{"amount":1000,"discount":0,"fine":0,"payment_mode":"cash","date":"2025-09-01","description":"…","received_by":1}}`.
6. `fee_receipt_no` is incremented atomically (separate table to avoid contention).
7. `Mailsmsconf::mailsms('fee_submission', …)` fires.
8. Receipt PDF generated using `views/studentfee/printFeesByName.php` + `backend/fee_receipt_pdf_style.css` + `print_headerfooter` template. If Thermal Print addon installed and `thermal_print_settings.is_print=1`, a small-format 58/80mm receipt is rendered instead.

**Status transitions on `fee_groups_feetype`:** `is_active=yes/no`. On `student_fees_deposite`: no status field — the JSON itself is the history.

**Validations:** Amount > 0; payment mode in enum; discount pre-assigned; fine computed from `fine_type` (`none|percentage|fixed|per_day`) × days late × `fine_per_day`.

**Notifications:** `fee_submission` event → SMS/Email/Push/WhatsApp.

**Failure scenarios:**
- Concurrent payment: two accountants paying the same student — `fee_receipt_no` allocation is atomic, but the JSON-append to `amount_detail` is **not transactional** — risk of overwriting.
- Bad JSON: if a previous write was corrupted, all subsequent reads fail.

### 4.3 Daily Attendance (faculty → cron auto-mark-absent)
**Actors:** Class Teacher, Cron, Students.
**Records touched:** `student_attendences`, `class_section_times`, `attendence_type`.

**Steps:**
1. Class teacher navigates `admin/stuattendence` and selects class + section + date.
2. For each enrolled student (via `student_session`), teacher marks a type from `attendence_type`: Present (1) / Late with excuse (2) / Late (3) / Absent (4) / Holiday (5) / Half Day (6) / Half Day Second Shift (8).
3. `Stuattendence_model->add()` bulk-inserts rows in `student_attendences`.
4. A row in `class_section_times` defines the deadline time for that class-section.
5. Cron `Cron::student_attendance($secret_key)` runs periodically (typically every 5–15 minutes); if any class-section has not submitted by `class_section_times.time`, the system auto-inserts `attendence_type_id=4` (Absent) for every enrolled student via `stuattendence_model->batch_insert($attendance_data)`.
6. `Mailsmsconf::sendAbsentAttendance(...)` fires (if enabled in `notification_setting` for `student_absent_attendence` event) — same for `student_present_attendence`.

**Validations:** Date not in future; class-section has enrolled students; teacher has RBAC `can_add` on `student_attendance` permission category.

**Failure scenarios:**
- Teacher submits after cron auto-marked absent: a duplicate row is inserted (no de-dup), causing inflated counts in reports.
- Cron secret key wrong: returns "Invalid Key or Direct access is not allowed" — silent failure.

### 4.4 CBSE Marksheet Generation (CBSE addon — high-complexity workflow)
**Actors:** Admin, Teacher, Exam Coordinator.
**Records touched:** 15+ CBSE tables.

**Steps:**
1. Admin creates `cbse_terms` (e.g. "Term 1", "Term 2").
2. Admin creates `cbse_exam_grades` (e.g. "A1", "A", "B") with `cbse_exam_grades_range` (min/max marks percentage per grade).
3. Admin creates `cbse_exam_assessments` (e.g. "Scholastic", "Co-Scholastic") and `cbse_exam_assessment_types` (e.g. "Written", "Practical", "Periodic Test").
4. Admin creates `cbse_exams` linked to a term + assessment + grade.
5. Admin creates `cbse_exam_timetable` rows per subject per exam: date, time_from, time_to, duration, room_no, `is_written`, `is_practical`, `written_maximum_marks`, `practical_maximum_mark`.
6. Admin maps class-sections via `cbse_exam_class_sections`; students auto-enrolled into `cbse_exam_students`.
7. Teacher enters marks via `admin/cbseexam/result` → writes `cbse_student_subject_marks` (one row per student per timetable per assessment_type) with `marks`, `is_absent`, `note`.
8. Admin clicks "Generate Rank" → `Exam::examrankgenerate()` computes `cbse_student_subject_result` (aggregated) and `cbse_student_exam_ranks` (rank within exam) using one of four strategies: `term_wise`, `all_term`, `multi_exam_without_term`, `exam_wise_rank`. Code is 1600+ lines in `Exam.php`.
9. For co-scholastic: teacher fills `cbse_observation_term_student_subparameter` against `cbse_observation_subparameter` rows under `cbse_observation_parameters`.
10. Admin picks a `cbse_template` (marksheet layout) + `cbse_template_class_sections` mapping, then `Result::printmarksheet()` generates the PDF via mPDF using `backend/cbse_pdf_style.css`.

**Status transitions on `cbse_exams.is_publish`:** `0 → 1` (visible to students).

**Validations:** Marks ≤ maximum_marks; absence flag; grade-range lookup.

**Notifications:** CBSE addon's `Cbse_mail_sms::mailsms('exam_result', $sender_details)` → Project 1's `Mailsmsconf::sendResult()`.

**Failure scenarios:**
- Partial marks entry: rank generation excludes students with missing marks (silent).
- Grade-range overlap: undefined behaviour — the first matching range wins.

### 4.5 TFA Login Flow (TFA addon)
**Actors:** Staff or Student/Parent user.
**Records touched:** `google_authenticator`, `user_google_authenticate_codes`, Project 1's `staff`/`users` tables.

**Steps:**
1. Admin enables 2FA globally via `admin/gauthenticate/admin` → sets `google_authenticator.use_authenticator=1`.
2. Each staff user visits `admin/gauthenticate/admin/setup` → `Admin::setup()` calls `google_authenticator->generateQR($staff['employee_id'])` which creates a 16-char base32 secret via `random_bytes()` and returns a QR URL (`otpauth://`).
3. User scans the QR with Google Authenticator / Authy / 1Password etc.
4. User enters the 6-digit code → `Admin::verify_setup()` calls `google_authenticator->verifyQR($secret, $code)`; on success, the secret is stored in `user_google_authenticate_codes` linked to `staff_id`.
5. Next login: Project 1's `Site::login()` checks `module_lib->hasModule('google_authenticator') && hasActive()` and redirects to `gauthenticate/login`.
6. User enters username + password (first factor). `Gauthenticate::login()` verifies via Project 1's `Staff_model::checkLogin()` (bcrypt) or `User_model::checkLogin()` (**plain-text comparison** — see §10).
7. `Gauthenticate::login()` prompts for 6-digit TOTP code → `google_authenticator->verifyQR($gauth->secret_code, $code)`.
8. On success, Project 1's `admin` or `student` session is set; user redirected to dashboard.

**Failure scenarios:**
- Lost device with secret: admin can delete the user's `user_google_authenticate_codes` row via `Admin::delete_setup()` — but this requires admin to already be 2FA-authenticated, creating a chicken-and-egg if Super Admin loses their device (no documented recovery flow).

### 4.6 Multi-Branch Switch (Multi Branch addon — flawed workflow)
**Actors:** Super Admin.
**Records touched:** `multi_branch`, Project 1's `application/config/database.php` (manual edit).

**Steps:**
1. Super Admin installs Project 1 + all 11 addons on master DB (`school_master`).
2. Super Admin registers the bundle Envato purchase code → `license.php` populated with 11 encrypted `addon_prod`/`addon_ver` pairs bound to `master.example.com`.
3. Super Admin installs Project 1 separately on `school_branch2` DB (different DB credentials).
4. Super Admin registers Project 1's purchase code on `branch2.example.com` — **separate license needed because base_url differs**.
5. Super Admin registers the bundle purchase code on `branch2.example.com` — **but this fails**, because the bundle's `addon_ver` entries are bound to `master.example.com`. The branch install needs its own bundle purchase.
6. On the master, Super Admin opens `admin/multibranch/branch` → enters branch2 DB credentials (`hostname`, `database_name`, `username`, `password`) → `Branch::add()` calls `multibranch_model->verify_branch($insert_Arr)` which opens a test mysqli connection. If successful, the row is inserted into `multi_branch` with `is_verified=1`. **Credentials stored in plain text.**
7. Super Admin must **manually edit master's `application/config/database.php`** to add a `$db['branch_1']` group with branch2 credentials.
8. Super Admin clicks "Switch Branch" → `Branch::switch()` sets `branch_cookie=branch_1` (2-year expiry) and updates admin session's `db_array.db_group`.
9. From master, every model query routes through branch2's DB group.

**Failure scenarios:**
- Branch DB credentials change: master's `multi_branch` row is stale; switch fails.
- Branch's `addons` table shows `current_stage=0`: addon controllers work on master (license is per-URL, not per-DB) but the in-app addon manager UI is confused.
- Cross-branch reports (`admin/multibranch/finance`) iterate every branch sequentially — at 50+ branches this would time out.

### 4.7 Online Course Purchase (Online Course addon)
**Actors:** Public visitor or Student, Payment gateway, Course Creator.
**Records touched:** `online_courses`, `online_course_section`, `course_lesson_quiz_order`, `online_course_lesson`, `online_course_quiz`, `course_progress`, `student_quiz_status`, `online_course_payment`, `online_course_processing_payment`, `online_course_certificate_template`.

**Steps:**
1. Admin creates `course_category` and `online_course_tag`.
2. Admin creates `online_courses` with `free_course` flag, `price`, `discount`, `course_category_id`, `thumbnail`.
3. Admin adds sections (`online_course_section`) and reorders via `Course::ordersection()` + `Course::updatesectionorder()` which writes `course_lesson_quiz_order`.
4. Admin adds lessons (`online_course_lesson` with video/audio/document URLs — local or S3) or quizzes (`online_course_quiz` with `course_quiz_question`s) into sections.
5. Admin may add `online_course_exam`, `online_course_assignment`, `online_course_certificate_template`.
6. Course is published via `Course::publish_unpublish()`.
7. Public visitor browses `welcome/online_course` and adds paid courses to cart (`controllers/Cart.php` using CodeIgniter's `cart` library).
8. On checkout, user selects a gateway → redirected to `students/online_course/{ActiveGateway}.php::index()`. The active gateway is determined by Project 1's `paymentsetting_model->getActiveMethod()` — same gateway as online-admission fees.
9. Gateway processes payment; on success writes `online_course_payment` and `online_course_processing_payment` (idempotency — the processing-payment row prevents double-credit on duplicate callbacks).
10. Student gains course access; `course_progress` updated as they complete sections; `student_quiz_status` and `online_course_exam_attempts` track attempts; `online_course_assignment_submit` holds uploaded files for evaluation via `online_course_assignment_evaluation`.
11. On course completion, certificate generated from `online_course_certificate_template`.

**Failure scenarios:**
- Duplicate gateway callback: `online_course_processing_payment` row prevents double-credit (idempotency verified).
- S3 upload failure: `Aws3::uploadFile()` has no retry — upload is lost.

### 4.8 Behaviour Incident + Multi-Channel Notification (cross-cutting workflow)
**Actors:** Teacher, Student, Parent.
**Records touched:** `student_behaviour`, `student_incidents`, `student_incident_comments`, Project 1's `notification_setting`, `send_notification`, `read_notification`.

**Steps:**
1. Admin/teacher creates `student_behaviour` incidents with positive or negative point values via `admin/behaviour/incidents`.
2. Teacher assigns an incident to a student via `admin/behaviour/studentincidents` (class → section → student → incident → date → remark).
3. `Studentincidents::add()` writes `student_incidents` row.
4. The addon's db.sql seed added `notification_setting.type='behaviour_incident_assigned'` with template `A new {{incident_title}} behaviour incident with {{incident_point}} point is assigned on you. {{student_name}} {{class}} {{section}} {{admission_no}} {{mobileno}} {{email}} {{guardian_name}} {{guardian_phone}} {{guardian_email}}`.
5. Project 1's `Mailsmsconf::mailsms('behaviour_incident_assigned', $sender_details)` fires → SMS/Email/Push/WhatsApp channels dispatch per the matrix.
6. Parent opens `user/Studentincidentcomments` → views the incident + adds a comment.
7. Comment stored in `student_incident_comments` (gated by `behaviour_settings.comment_option` JSON list).

**Integration depth:** Project 2 #4 (Behaviour) + Project 1 dispatcher + Project 1 SMS/Email/Push gateways + Project 2 #11 (WhatsApp) all fire for one incident assignment.

### 4.9 Quick Fees Installment Creation (Quick Fees addon)
**Actors:** Accountant.
**Records touched:** Project 1's `fee_groups`, `feetype`, `fee_groups_feetype`, `fee_session_groups`, `student_fees_master`.

**Steps:**
1. Accountant opens `admin/customfeesmaster` → selects class → section → student.
2. Existing `student_fees_master` for that student is loaded via `Customfeesmaster::getStudentFeesMaster()`.
3. Accountant clicks "Quick Create" → enters `total_fees`, `down_payment`, `balance_fees`, `no_of_installment`, `amount` (per installment), `day` (interval), `fine_type` (`none|percentage|fixed|per_day`), `fine_type_value`.
4. `Customfeesmaster::createInstallment()` validates inputs (e.g. `down_payment ≤ total_fees`) and renders a preview of the installment schedule via `_prtialCreateInstallment.php` view partial.
5. On confirm, `Customfeesmaster::savecustomfeesmaster()` writes:
   - one `fee_groups` row named `{student_id}-{admission_no}-{firstname}`
   - N `feetype` rows ("Installment 1", "Installment 2", ...)
   - N `fee_groups_feetype` rows with amount, due_date (computed from start_date + day × index), fine config
   - one `fee_session_groups` row
   - N `student_fees_master` rows
6. All writes happen in a CodeIgniter transaction.

**Weakness:** Uses raw `$_POST` access (`$_POST['total_fees']`, `$_POST['down_payment']`, etc.) instead of CodeIgniter's `$this->input->post()` — bypasses xss_clean. **Verified in source** — `Customfeesmaster.php::createInstallment()`.

### 4.10 Addon Installation (cross-cutting, security-critical)
**Actors:** Super Admin.
**Records touched:** `addons`, `addon_versions`, `license.php` config file, `./addons/{name}/` directory, host's `application/`, `backend/`, `uploads/` directories.

**Steps:**
1. Super Admin buys the bundle (or individual addon) on CodeCanyon; receives an Envato purchase code.
2. In Project 1, admin opens `admin/addons` and uploads the addon ZIP.
3. `Addons::index()` moves the ZIP to `./addons/{name}.zip` and extracts to `./addons/temp/{name}`.
4. `Addons::check_valid_zip()` requires either `installer/` or `updater/` sub-folder.
5. On success, ZIP is re-extracted to `./addons/{name}`.
6. `addon_info.json` is read via `read_json()` (`helpers/addon_helper.php`).
7. `addons_model::addAddonSetupDetails()` updates the `addons` table row matching `product_id` (so `current_stage` transitions from 0 → 1 → 2).
8. Admin registers the addon via `admin/admin/updatePurchaseCode()` which posts the Envato purchase code + addon product_id to QDOCS's licensing server; the server returns an encrypted `addon_prod`/`addon_ver` pair bound to the install's `base_url()`, written to Project 1's `config/license.php`.
9. Admin runs the addon's `installer/install/db/db.sql` via the in-app "Install Database" button — `helpers/addon_helper.php::execute_sql()` uses `$CI->db->conn_id->multi_query($sql)` to execute all CREATE/INSERT statements at once.
10. Addon files under `installer/install/files/application/...`, `installer/install/files/backend/...`, `installer/install/files/uploads/...` are copied into Project 1's corresponding directories.
11. Addon controllers auto-loaded via `spl_autoload_register('my_autoloader')` in `config/routes.php` for any class starting with `MY_Addon_`.
12. New `sidebar_menus` and `sidebar_sub_menus` rows appear in admin sidebar — gated by `Rbac::hasPrivilege()` and `Module_lib::hasActive()`.
13. Every addon controller constructor calls `$this->auth->addonchk('{slug}', $return_url)` to verify the license at runtime.

**Failure scenarios:**
- Malformed SQL: `multi_query()` halts at the first error; partial schema changes are not rolled back.
- File-copy step is incomplete: addon classes missing → fatal errors when admin clicks the new sidebar item.
- License server unreachable: addon is installed but `addonchk()` fails at runtime → addon controllers redirect to setting page (which itself doesn't exist) → redirect loop.

---

## 5. Domain Model and Data Relationships

### 5.1 Main entities and tables (combined schema: 273 tables)

#### Core entities (Project 1, 196 tables)
| Entity | Table(s) | Purpose |
|---|---|---|
| School / Tenant | `sch_settings` (~150 cols), `sessions` | Single-school config + academic-year |
| Class structure | `classes`, `sections`, `class_sections`, `class_section_times`, `school_houses`, `categories` | Class + section + house + category master |
| Student | `students` (~50 cols), `student_session`, `student_doc`, `student_timeline`, `student_edit_fields`, `alumni_students` | Student profile + per-year enrollment + documents + timeline |
| Parent / Guardian | `users` (role='parent'), `students.parent_id` | Parent login + child linking |
| Staff | `staff` (~50 cols), `staff_designation`, `department`, `staff_roles`, `staff_timeline`, `staff_rating` | Staff profile + roles + timeline |
| Authentication | `users`, `users_authentication`, `api_session`, `google_authenticator` (addon) | Login + mobile API tokens |
| RBAC | `roles`, `roles_permissions`, `staff_roles`, `permission_group`, `permission_category`, `permission_student` | Roles + 4-action permission matrix |
| Sidebar | `sidebar_menus`, `sidebar_sub_menus` | Dynamic sidebar from DB |
| Fees | `fee_groups`, `feetype`, `fee_groups_feetype`, `fee_session_groups`, `feemasters`, `student_fees_master`, `student_fees`, `student_fees_deposite`, `fee_receipt_no`, `fees_reminder`, `fees_discounts`, `student_applied_discounts`, `student_fees_discounts`, `student_fees_processing`, `student_transport_fees`, `transport_feemaster`, `cumulative_fine`, `offline_fees_payments` | Fee structure + assignment + payment history (JSON-in-TEXT) + receipts |
| Attendance | `student_attendences`, `staff_attendance`, `student_subject_attendances`, `attendence_type`, `staff_attendance_type`, `student_attendence_schedules`, `staff_attendence_schedules` | Daily + subject-wise + biometric + QR |
| Exams | `exams`, `exam_groups`, `exam_schedules`, `exam_group_class_batch_exams`, `exam_group_class_batch_exam_students`, `exam_group_class_batch_exam_subjects`, `exam_group_exam_connections`, `exam_group_exam_results`, `exam_group_students`, `mark_divisions`, `grades`, `onlineexam`, `onlineexam_attempts`, `onlineexam_questions`, `onlineexam_students`, `onlineexam_student_results`, `questions`, `template_admitcards`, `template_marksheets` | Hierarchical exam + marks + result + rank + online exam |
| Academics | `subjects`, `subject_groups`, `subject_group_class_sections`, `subject_group_subjects`, `subject_syllabus`, `subject_timetable`, `homework`, `homework_evaluation`, `submit_assignment`, `daily_assignment`, `lesson`, `topic`, `lesson_plan_forum` | Subjects + syllabus + homework + lesson plan |
| HR | `staff_payroll`, `staff_payslip`, `payslip_allowance`, `staff_leave_request`, `staff_leave_details`, `leave_types`, `holiday_type`, `staff_id_card` | Payroll + leave |
| Library | `books`, `book_issues`, `libarary_members` (sic — typo) | Catalog + circulation |
| Hostel & Transport | `hostel`, `hostel_rooms`, `room_types`, `vehicles`, `vehicle_routes`, `transport_route`, `pickup_point`, `route_pickup_point` | Allocation + routing |
| Inventory | `item`, `item_store`, `item_category`, `item_stock`, `item_supplier`, `item_issue`, `dispatch_receive` | Stock + issue |
| Front Office | `enquiry`, `enquiry_type`, `follow_up`, `general_calls`, `complaint`, `complaint_type`, `visitors_book`, `visitors_purpose`, `dispatch_receive`, `reference`, `source` | Enquiries + visitors + complaints |
| Communication | `notification_setting`, `send_notification`, `read_notification`, `notification_roles`, `chat_connections`, `chat_messages`, `chat_users`, `messages`, `email_config`, `email_template`, `sms_config`, `sms_template` | Notification matrix + internal chat |
| Front CMS | `front_cms_pages`, `front_cms_page_contents`, `front_cms_menus`, `front_cms_menu_items`, `front_cms_programs`, `front_cms_program_photos`, `front_cms_settings`, `front_cms_media_gallery`, `contents`, `content_for`, `content_types` | Public website |
| Finance | `expenses`, `expense_head`, `income`, `income_head` | Income + expense |
| System | `addons`, `addon_versions`, `logs`, `userlog`, `print_headerfooter`, `filetypes`, `currencies`, `languages`, `custom_fields`, `custom_field_values` | Addon registry + audit + i18n + custom fields |

#### Addon entities (Project 2, 77 tables)
| Addon | Tables | Key entities |
|---|---|---|
| CBSE Exam (28) | `cbse_terms`, `cbse_exams`, `cbse_exam_assessments`, `cbse_exam_assessment_types`, `cbse_exam_timetable`, `cbse_exam_timetable_assessment_types`, `cbse_exam_class_sections`, `cbse_exam_students`, `cbse_student_subject_marks`, `cbse_student_subject_result`, `cbse_student_exam_ranks`, `cbse_exam_student_subject_rank`, `cbse_student_template_rank`, `cbse_template`, `cbse_template_class_sections`, `cbse_template_terms`, `cbse_template_term_exams`, `cbse_exam_grades`, `cbse_exam_grades_range`, `cbse_marksheet_type`, `cbse_observation_parameters`, `cbse_observation_subparameter`, `cbse_observation_class_section`, `cbse_observation_terms`, `cbse_observation_term_student_subparameter`, `cbse_exam_observations`, `cbse_category`, `cbse_template_admitcards` | Term + exam + assessment + observation + template + rank |
| Online Course (29) | `online_courses`, `online_course_section`, `course_lesson_quiz_order`, `online_course_lesson`, `online_course_lesson_attachment`, `online_course_quiz`, `course_quiz_question`, `course_quiz_answer`, `student_quiz_status`, `online_course_exam`, `online_course_exam_question`, `online_course_exam_attempts`, `online_course_exam_marks`, `online_course_exam_result`, `online_course_assignment`, `online_course_assignment_submit`, `online_course_assignment_evaluation`, `course_progress`, `course_rating`, `course_category`, `online_course_tag`, `online_course_payment`, `online_course_processing_payment`, `online_course_class_sections`, `online_course_certificate_template`, `online_course_start_date`, `online_course_settings`, `aws_s3_settings`, `guest` | Course + section + lesson/quiz/exam/assignment + progress + payment + certificate |
| Behaviour (4) | `student_behaviour`, `student_incidents`, `student_incident_comments`, `behaviour_settings` | Incident + point + comment |
| Multi Branch (1) | `multi_branch` | Branch DB credentials directory |
| Zoom (5) | `conferences`, `conference_staff`, `conferences_history`, `zoom_settings`, `conference_sections` | Meeting + history + multi-section/multi-teacher |
| GMeet (5) | `gmeet`, `gmeet_history`, `gmeet_sections`, `gmeet_settings`, `gmeet_staff` | Same shape as Zoom |
| QR Attendance (1) | `QR_code_settings` | Camera type + auto-attendance flag |
| TFA (2) | `google_authenticator`, `user_google_authenticate_codes` | Global toggle + per-user secret |
| Thermal Print (1) | `thermal_print_settings` | Toggle + school_name + address + footer |
| Quick Fees (0) | — | Reuses P1 fee tables |
| WhatsApp (1) | `whatsapp_config` | Same shape as P1 `sms_config` |

### 5.2 Relationships and ownership
```mermaid
graph TB
    subgraph "Core scoping"
        sch_settings[sch_settings<br/>single school config]
        sessions[sessions<br/>academic year]
        classes[classes]
        sections[sections]
        class_sections[class_sections]
        student_session[student_session<br/>PER-YEAR ENROLLMENT]
        students[students]
        staff[staff]
    end
    subgraph "Fees"
        fee_groups_feetype[fee_groups_feetype<br/>fine_type, due_date]
        student_fees_master[student_fees_master<br/>per-student per-session]
        student_fees_deposite[student_fees_deposite<br/>amount_detail TEXT-JSON]
        fee_receipt_no[fee_receipt_no<br/>atomic counter]
    end
    subgraph "Attendance"
        student_attendences[student_attendences]
        class_section_times[class_section_times<br/>deadline time]
    end
    subgraph "Exams"
        exam_groups[exam_groups]
        exams[exams]
        exam_group_exam_results[exam_group_exam_results]
    end
    subgraph "Communication"
        notification_setting[notification_setting<br/>per-event x channel matrix]
        send_notification[send_notification]
    end
    subgraph "RBAC"
        roles[roles]
        roles_permissions[roles_permissions<br/>can_view/add/edit/delete]
        permission_group[permission_group<br/>module toggle]
        permission_category[permission_category<br/>sub-feature]
    end

    sch_settings --> sessions
    sessions --> student_session
    classes --> class_sections
    sections --> class_sections
    class_sections --> student_session
    students --> student_session
    student_session --> student_fees_master
    fee_groups_feetype --> student_fees_master
    student_fees_master --> student_fees_deposite
    student_session --> student_attendences
    class_section_times --> student_attendences
    student_session --> exam_group_exam_results
    exam_groups --> exams
    exams --> exam_group_exam_results
    roles --> roles_permissions
    permission_category --> roles_permissions
    permission_group --> permission_category
    notification_setting -.->|event dispatch| send_notification
```

**Key relationships:**
- `student_session` is the **single most important table** — it links `student × session × class × section × hostel × transport route × pickup point`. Every per-year operational query filters by `session_id = current_session`. This is the right abstraction for academic-year scoping.
- `users.id ← students.parent_id` — parent links to multiple children via `users.childs` JSON list.
- `staff.user_id → users.id` — staff login goes through `users` but staff data lives in `staff`.
- `fee_groups_feetype → student_fees_master → student_fees_deposite` — three-level fee model where the deposite's `amount_detail` TEXT column holds a JSON history.
- `conferences.staff_id → staff.id` (FK CASCADE), `conferences_history.student_id → students.id` (FK CASCADE) — addon tables FK to core.
- `user_google_authenticate_codes.user_id → users.id` (FK CASCADE), `.staff_id → staff.id` (FK CASCADE).
- `cbse_template_class_sections.cls_section_id → class_sections.id` (FK).

### 5.3 Important fields and statuses
- `students.is_active` (`varchar(255)` — yes/no; inconsistent typing across schema).
- `staff.is_active` (`int` — 0/1; inconsistent with `students`).
- `staff.disable_at` (`date`) — when staff was disabled.
- `student_session.is_active`, `is_alumni`, `is_leave`, `default_login` — per-year lifecycle.
- `cbse_exams.is_publish` (`int(1)`) — controls student visibility.
- `conferences.status` (`int(1)`) — meeting status.
- `staff_payslip.status` (`varchar(100)`) — `generated` / `paid` / `unpaid` / `not_generate`.
- `enquiry.status` — `active` / `passive` / `dead` / `won` / `lost` — sales-funnel states.
- `fee_groups_feetype.fine_type` — `none` / `percentage` / `fixed` / `per_day`.
- `notification_setting.is_mail`, `is_sms`, `is_whatsapp`, `is_notification` — channel toggles.
- `users.login_token` (`varchar(255)`) — rotated on every login (`bin2hex(random_bytes(32))`).
- `users.password` (**`varchar(50)`** — plain text; **Critical finding**).
- `staff.password` (`varchar(250)` — bcrypt hash; correct).
- `sch_settings.cron_secret_key` — gates cron endpoints.
- `multi_branch.password` (plain text — **Critical finding**).

### 5.4 Constraints and business rules
- **Admission number auto-generation**: `sch_settings.adm_prefix` + `adm_no_digit` (length) + `adm_start_from` + `adm_auto_insert` flag — generates `ssadm19/20/000001`-style IDs.
- **Receipt numbering**: `fee_receipt_no` is a separate table with atomic UPDATE-then-SELECT to avoid contention.
- **Fee fine calculation**: `fine_type × fine_percentage OR fine_amount OR fine_per_day × days_late`.
- **Class teacher scope filter**: `MY_Model::applyTeacherClassSectionFilter()` restricts teacher queries to their assigned class-sections when `sch_settings.class_teacher == 'yes'` and the staff is a Teacher role (role id 2). This is **record-level access control**.
- **Super Admin bypass**: `Rbac::hasPrivilege()` returns true unconditionally for role 7 (`is_superadmin=1`).
- **Module gating**: `permission_group.is_active` + `Module_lib::hasActive($short_code)` controls module-level visibility in sidebar.
- **Addon licensing**: `Auth::addonchk('{slug}')` decrypts `addon_ver` entries with `addon_prod` keys (AES) and parses `"slug!!base_url()"` — both must match.

### 5.5 Transaction history
- `student_fees_deposite.amount_detail` is a TEXT column holding a JSON history of every payment: `{"1":{"amount":1000,"discount":0,"fine":0,"payment_mode":"cash","date":"2025-09-01","description":"…","received_by":1},"2":{...}}`. **Each payment appends a JSON object keyed by incrementing counter.** This is a sub-table embedded in a TEXT field.
- `staff_payslip` has separate `total_allowance`, `total_deduction`, `leave_deduction`, `tax`, `net_salary` columns — no separate payslip-line-item table.
- `student_timeline` and `staff_timeline` are dedicated tables for student/staff lifecycle events.
- `homework_evaluation` records per-student homework evaluation.
- `conferences_history.total_hit` counts join clicks per student/conference.

### 5.6 Audit records
- `logs` table: `message, record_id, user_id, action, ip_address, platform, agent, time` — written by `MY_Model::log()` after every CUD operation across most models (including addon models that extend `MY_Model`). Message format: `"New Record inserted On {table} id {id}"` / `"Record updated"` / `"Record deleted"`.
- `userlog` table: separate login history per user.
- `gateway_ins_response` table: raw posted data + response from gateway INS callbacks.
- `conferences_history`, `gmeet_history`: per-meeting join tracking.
- `cbse_observation_term_student_subparameter`: per-student per-term observation scores (audit trail of co-scholastic assessment).

### 5.7 Configuration models
- `sch_settings`: ~150 columns covering school identity, currency, date/time format, language, theme, attendance type, online-admission config, auto-numbering, 60+ field-visibility booleans, app colours, cron secret, maintenance mode, feature lock, WhatsApp range/hours.
- `notification_setting`: per-event × per-channel × per-recipient-class toggles + template + DLT template IDs.
- `payment_settings`: one active gateway; testing/live mode; processing-fee config.
- `email_config`: SMTP/SES/Mailgun; per-protocol fields.
- `sms_config`: 11 provider types; one active.
- `whatsapp_config`: Meta or Twilio; one active.
- `zoom_settings` / `gmeet_settings`: OAuth client ID + secret; per-teacher option flag.
- `thermal_print_settings`: toggle + header/footer text.
- `QR_code_settings`: camera type + auto-attendance flag.
- `behaviour_settings`: JSON list of allowed commenters.
- `print_headerfooter`: per-print-type header image + footer content.
- `custom_fields` + `custom_field_values`: admin-defined custom fields per entity.
- `filetypes`: allowed extensions + MIME types + sizes per file category.

---

## 6. UI/UX Patterns

### 6.1 Main navigation and module organization
- **AdminLTE-style sidebar** (`views/layout/sidebar.php`) where menus are **dynamic from DB** (`sidebar_menus`, `sidebar_sub_menus` with `access_permissions`, `permission_group_id`, `addon_permission`, `activate_controller`, `activate_methods`, `lang_key`, `icon`). Addons seed new rows; sidebar view auto-renders them gated by:
  1. `Rbac::hasPrivilege($cat, $perm)` (RBAC check)
  2. `Module_lib::hasModule($short_code) && hasActive($short_code)` (module-toggle check)
  3. `Auth::addonchk($addon_permission, false)` (addon license check)
- A top search form in the sidebar lets admin search students by name (POST → `admin/admin/search`).
- A top-bar renders theme settings from JS (`themeSettings.theme_color`, `theme_background`, `theme_navigation`) — applied client-side from a JSON blob.
- User panel (`views/layout/student/`) has a separate, simplified nav.
- Front-end site uses `views/front/` plus themes from `backend/themes/front-main.css`.

### 6.2 Dashboards
- `views/admin/dashboard.php`: unread notifications, fee collection (current month + month-by-month chart), expenses, total students/teachers, role counts, MySQL version & `sql_mode` warning (flags `ONLY_FULL_GROUP_BY` as incompatible), SaaS storage-quota alerts (rendered but always empty).
- `views/user/dashboard.php`: student/parent dashboard.
- Multi Branch's `Branch::overview()`: cross-branch dashboard with 12 KPIs per branch.
- CBSE addon's `Chart.php`: per-exam grade-distribution chart.
- Online Course addon's `Coursereport.php`: course sales + progress reports.

### 6.3 Forms and data-entry workflows
- All forms use CodeIgniter's `form_validation->set_rules(...)` with extensive callbacks (`callback_handle_upload`, `callback_validate_extension`, `callback_check_captcha`, `callback_validate_storage`).
- CSRF token in every form via `customlib::getCSRF()`.
- File uploads enforce MIME/extension/size via `config/app-config.php` (`$config['image_validate']`, `$config['csv_validate']`, `$config['file_validate']`).
- "Custom field" rendering handled by `libraries/Form_builder.php` (40 KB) and `helpers/customfield_helper.php`.
- Date pickers, select2, dropify, sweet-alert, fullcalendar bundled in `backend/`.

### 6.4 Search and filtering
- Server-side DataTables processing via `libraries/Datatables.php` (24 KB).
- Common date-range filter presets in `config/payroll.php → $config['search_type']` (11 presets: today/this_week/last_week/this_month/last_month/last_3_month/last_6_month/last_12_month/this_year/last_year/period).
- AJAX pagination via `libraries/Ajax_pagination.php`.
- Cross-field search: e.g. `admin/admin/search` accepts `search_text` and searches across student name, admission no, class.

### 6.5 Bulk operations
- Student CSV import (`Student::import()` with `handle_csv_upload()` validation).
- Student bulk delete (`Student::bulkdelete()`).
- Attendance bulk insert (`Stuattendence_model->batch_insert()`).
- Fee reminder bulk dispatch via cron.
- Email/SMS broadcast (`admin/mailsms` with multi-recipient selection).

### 6.6 Detail pages
- Student detail: tabs for profile, fees, attendance, exams, documents, timeline, transport, hostel — `views/admin/student/view.php`.
- Staff detail: timeline — `views/admin/staff/profile.php`.
- Fee detail: paid/dues/discounts/fines — `views/studentfee/view.php`.
- CBSE exam result detail: subject-wise marks, grades, rank, observations on printable marksheet template.
- Online Course preview: course outline (sections + lessons + quizzes) — `Course::coursepreview()`.

### 6.7 Reports
- Most reports are direct SQL queries rendered as DataTables; no business-intelligence layer.
- Date-range presets reuse the 11 presets from `config/payroll.php`.
- Some reports export to PDF via mPDF (`Studentfee::pdf()`, `Report::pdf_report()`).

### 6.8 Role-specific interfaces
- Admin: full sidebar (all modules visible per RBAC).
- Teacher: scoped sidebar + class-teacher filter applied to all student queries.
- Student/Parent: separate `views/user/` and `views/layout/student/` with simplified nav.
- Accountant: fees + finance modules only.
- Librarian: library + book issues only.
- Receptionist: front-office modules only.
- Super Admin: same as Admin but bypasses all RBAC checks.

### 6.9 Mobile and responsive behavior
- Bootstrap grid; admin theme has light/dark mode, RTL toggle, sidebar collapse toggle, container-fluid toggle, shadow toggle — all configurable in `sch_settings` and editable live from the top bar.
- 6 preset themes: `bold_blue`, `darkgray`, `default`, `material_pink`, `shadow_white`, `yellow`.
- QR Attendance scanner view uses `getUserMedia` for camera access — works on desktop and mobile.
- Thermal Print view is intentionally narrow (58/80mm), not responsive — meant for thermal printers.
- Mobile app (Flutter, separate from this bundle) uses `mobile_api_url` from `sch_settings`; token-based auth via `users_authentication` table.

### 6.10 Patterns transferable to a unified management workspace
1. **Dynamic sidebar from DB** — addons/extensions just seed menu rows; no view changes needed.
2. **Per-event × per-channel notification matrix UI** — admin sets toggles per (event × channel × recipient-class) on a single screen.
3. **Field-visibility toggles on forms** — 60+ booleans in `sch_settings` let each school customise student/staff forms without code.
4. **Theme live-toggle** — JS reads theme settings from a JSON blob and applies classes to `<body>`; no page reload.
5. **Tokenised shareable PDF links** — encrypted URLs for parent distribution without file attachment.
6. **Lock-panel pattern** — admin can force-redirect student users to the fees page only (`Student_Controller::__construct()` checks `check_lock_enabled()`).
7. **Login-token rotation** — each successful student/parent login sets `users.login_token = bin2hex(random_bytes(32))` to invalidate other sessions.

---

## 7. Technical Architecture

### 7.1 Frameworks and technologies
- CodeIgniter 3 (EOL October 2025) — no upgrade path to CI4 visible.
- PHP 7.x / 8.x compatible (`libraries/Version_control.php`).
- MySQL 5.7+ via `mysqli` driver; InnoDB; `utf8mb3` collation (not utf8mb4 — limits emoji).
- jQuery + AdminLTE + Bootstrap + FullCalendar + DataTables + Dropify + Select2 + SweetAlert + Toaster.
- mPDF for PDF generation.
- PHPMailer + AWS SES for email.
- Firebase FCM v1 for push (OAuth service-account JSON).
- AWS SDK (for S3, in Online Course addon).
- Guzzle HTTP client (for Zoom OAuth, in Zoom addon).
- Google API Client (for Google Calendar, in GMeet addon).
- Omnipay vendor bundle (for some payment abstractions).

### 7.2 Application structure
```
application/
├── config/           # 27 config files (routes, app-config, mailsms, saas-config, license, database, payroll, ...)
├── controllers/      # 35 root + 110 admin + 27 user + 28 onlineadmission + 6 gateway_ins + install/
├── core/             # MY_Controller (5 base classes), MY_Model
├── helpers/          # 8 helpers (addon, csv, custom, customfield, json_output, lang, menu, mime)
├── hooks/            # maintenance_hook
├── language/         # 78 languages × app_files/ + db_lang + form_validation_lang
├── libraries/        # ~60 libraries (Auth, Enc_lib, Rbac, Role, Module_lib, Customlib 90KB, Mailsmsconf 78KB, Smsgateway 138KB, Form_builder 40KB, Datatables 24KB, Mailgateway 25KB, Pushnotification, PHPMailer/, ...)
├── models/           # 155 models extending MY_Model
├── third_party/      # omnipay/, aws/, firebase_notification_key.json
├── views/            # 27 view directories
└── tmp/              # mPDF temp
backend/              # CSS, JS, themes (6), plugins, calender, captcha_images, custom, datepicker, dist, fullcalendar, images, import, installer_template, js, json-files, multiselect, pdf_style.css, fee_receipt_pdf_style.css, resume_pdf_style.css, plugins, report, rtl, sweet-alert, themes, toast-alert, usertemplate
backup/               # database_backup/, _tmp/, temp_uploaded/
system/               # CodeIgniter 3 framework
uploads/              # 39 upload directories (accountant_images, addon_images, admission_form, admit_card, ...)
temp/
addons/               # created at runtime by addon installer
```

### 7.3 Module extensibility
- **Addon ZIP-overlay contract**: each addon ships `addon_info.json` + `installer/install/{files,db}` + `installer/uninstall/{db.sql,unistall_directory.json}`. Files land in host's `application/`, `backend/`, `uploads/` directories; SQL runs via `multi_query()`.
- **`MY_Addon_*Controller` autoloader**: `config/routes.php` registers `spl_autoload_register('my_autoloader')` for any class starting with `MY_Addon_` from `application/core/`.
- **Forward-compatible dispatcher pattern**: Project 1's `Mailsmsconf::mailsms()` pre-calls `$this->CI->whatsappgateway->{method}()` (20 calls). The `whatsappgateway` class doesn't exist in P1 — it's provided by Project 2 #11. Calls are guarded by `if ($chk_mail_sms['whatsapp'])` so they only fire when admin has enabled the channel.
- **Module-toggle + sub-feature permission matrix as DB seeds**: addons seed `permission_group` (module-level toggle) + `permission_category` (sub-features) + `roles_permissions` (default grants). Core's `Module_lib::hasActive()` and `Rbac::hasPrivilege()` enforce these uniformly.
- **Notification-matrix extension**: addons seed new event types into existing `notification_setting` table. Dispatcher (`Mailsmsconf::mailsms($event_key, $sender_details)`) and per-channel gateways handle the rest.

### 7.4 Service and controller organization
- Fat-controller pattern: `Welcome.php` 1715 lines, `Site.php` 1003, `Customlib.php` 2493, `Smsgateway.php` 3145, `Mailsmsconf.php` 78KB, CBSE `Exam.php` 1600+ lines, CBSE `Result.php` 2400+ lines.
- Controllers extend one of 5 base classes: `MY_Controller`, `Admin_Controller`, `Student_Controller`, `Studentgateway_Controller`, `Public_Controller`.
- Addon controllers extend `MY_Addon_{Slug}Controller` which extends `Admin_Controller` (or `Student_Controller`).
- Models extend `MY_Model` which provides `log()` for audit and `applyTeacherClassSectionFilter()` for record-level access control.

### 7.5 Database access
- CodeIgniter's Query Builder (`$this->db->select()->from()->where()->get()`).
- Mostly parameterised queries via Query Builder, but **raw SQL string concatenation in a few places** — e.g. `Module_model::getPermissionByModulename()` line 80: `$sql = "select is_active from permission_group where short_code='" . $module_name . "'"`.
- Multi-DB support via `$db['group_name']` in `config/database.php`; loaded per-request via `$this->load->database($group, TRUE)`.
- Transactions via `$this->db->trans_start()` / `trans_complete()` in models.
- No read replicas, no caching layer (`cache_on = false`).

### 7.6 API architecture
- REST-ish JSON endpoints (`controllers/App.php::index()` returns mobile app config; `controllers/Biometric.php` accepts JSON `{serial_number, user_id, t}`).
- Mobile app authentication via `users_authentication` table: `users_id, token, staff_id, expired_at`.
- No formal API versioning, no OpenAPI spec.
- Some controllers use `json_output(200, array('status' => 200, 'message' => '...'))` helper for consistent JSON responses.
- 28 payment-gateway webhook/callback controllers (`controllers/onlineadmission/*`, `controllers/gateway_ins/*`, `controllers/Webhooks::insta_webhook`).

### 7.7 Background processing
- **Cron dispatcher** (`controllers/Cron.php`) — all entry points require `sch_settings.cron_secret_key` in the URL. Routes:
  - `cron/{key}` → `index()` runs `autobackup`, `feereminder`, `eventreminder`, `schedulesmsemails` sequentially.
  - `cron/student_attendance/{key}` → marks absent for un-submitted class-sections.
- **No queue** — cron is a single HTTP hit running 4 jobs sequentially; long-running reminders can time out.
- Addons do not ship their own cron jobs — they reuse the existing `schedulesmsemails` job.

### 7.8 Events and notifications
- "Events" are not OOP events — they're a flat lookup table `notification_setting(type, is_mail, is_sms, is_whatsapp, is_notification, template, ...)`. Event keys are hard-coded in `config/mailsms.php → $config['mailsms']` (24 events).
- Dispatch flow: `$this->mailsmsconf->mailsms($event_key, $sender_details)` → `mailsmsconf` looks up `notification_setting` → per-channel dispatch to `Smsgateway::sendSMS()`, `Mailgateway::sendLoginCredential()`, `Pushnotification::send()`, `Whatsappgateway::{eventMethod}()`.
- Read receipts for in-app notifications in `read_notification` table.

### 7.9 Authentication and authorization
- **Two parallel auth systems**:
  - **Staff (admin-side)**: `Site::login()` → `Staff_model::checkLogin()` → bcrypt via `Enc_lib::passHashDyc()`. Session key: `admin`.
  - **Student/Parent**: `Site::userlogin()` → `User_model::checkLogin()` → **plain-text comparison** (`$result[0]->password == $data['password']`). Session key: `student`.
- **RBAC**: `permission_group` (module-level) + `permission_category` (sub-feature with 4 booleans: `can_view/add/edit/delete`) + `roles_permissions` (per-role grants). `Rbac::hasPrivilege($cat, $perm)` checks. Super Admin (role 7) bypasses all.
- **Student/Parent module gating**: separate `permission_student` table with per-group student + parent toggles.
- **Record-level access**: `MY_Model::applyTeacherClassSectionFilter()` restricts teacher queries to their assigned class-sections when `class_teacher == 'yes'`.
- **CSRF**: token rendered in every form via `customlib::getCSRF()`.
- **Captcha**: per-page configurable via `libraries/Captchalib.php`.
- **2FA**: optional Google Authenticator (addon) — Project 1's `Site::login()` redirects to `gauthenticate/login` if module active.
- **Login-token rotation**: each student/parent login sets `users.login_token = bin2hex(random_bytes(32))`.
- **Addon licensing**: `Auth::addonchk('{slug}')` URL-bound AES check.

### 7.10 File management
- `uploads/` directory with 39 subdirectories (`accountant_images`, `addon_images`, `admission_form`, `admit_card`, ...).
- File paths stored as TEXT in DB tables (`students.image`, `staff.image`, `online_course_lesson_attachment`, etc.).
- `libraries/Media_storage.php` (11 KB) for file upload/delete abstractions.
- `helpers/mime_helper.php` for MIME-type lookup.
- `config/app-config.php` defines allowed MIME/extension/size per file category.
- AWS S3 option (Online Course addon) via `aws_s3_settings` table + `libraries/Aws3.php` (uses `Aws\S3S3Client`).
- Google Drive integration (`google_drive_setting` table).
- Image resize via `libraries/ImageResize.php`.
- QR code generation via `libraries/QR_Code.php`.
- PDF generation via `libraries/M_pdf.php` (wraps mPDF).

### 7.11 Logging and error handling
- `application/logs/` (CodeIgniter default).
- `addons/error_log.txt` — append-only log written by `addon_helper::logMessage()` for addon installer activity.
- `MY_Model::log()` writes audit entries to `logs` table after every CUD operation.
- Production errors routed via `routes['404_override'] = 'welcome/show_404'`.
- **SMS gateway calls swallow exceptions**: `catch (Exception $e) { return true; }` in `Smsgateway::sendSMS()` for Clickatell and `if ($response->IsError) { return true; }` for Twilio — both return `true` even on failure. **Failures not surfaced.**

### 7.12 External integrations
| Category | Providers | Library |
|---|---|---|
| SMS | Clickatell, Twilio, MSG91, SMS Country, TextLocal, Bulk SMS, MobiReach, Nexmo, Africa's Talking, SMS EG, Custom | `libraries/Smsgateway.php` (138 KB, 3145 lines) |
| Email | SMTP (PHPMailer), AWS SES, Mailgun-style via Omnipay | `libraries/Mailer.php`, `libraries/Mailgateway.php`, `libraries/Aws_mail.php` |
| Push | Firebase Cloud Messaging v1 (OAuth service-account) | `libraries/Pushnotification.php` |
| Payments | 28 gateways (Stripe, Razorpay, PayU, PayPal, CCAvenue, Instamojo, SSLCommerz, Paystack, Flutterwave, Mollie, Midtrans, Paytm, Toyyibpay, Skrill, Twocheckout, Payhere, Payfast, Cashfree, Billplz, DPO Pay, Jazzcash, MomoPay, Onepay, Kowri, IPayAfrica, Pesapal, Walkingm, Checkout) | `controllers/onlineadmission/*`, `controllers/gateway_ins/*` |
| Live class | Zoom (OAuth2 + Guzzle), Google Meet (Google Calendar API + Hangouts Meet) | Addon libraries `Zoom_api.php`, `Google.php` |
| Biometric | Device push API — JSON `{serial_number, user_id, t}` | `controllers/Biometric.php` |
| Cloud storage | AWS S3, Google Drive | `libraries/Aws3.php`, `google_drive_setting` table |
| WhatsApp | Meta Cloud API, Twilio WhatsApp | Addon library `Whatsappgateway.php` |
| Analytics | `front_cms_settings.google_analytics` | Rendered on public site |
| Map | `pickup_point.latitude/longitude` | Stored but not used in verified client-side map widget |

### 7.13 Architecture trade-offs
**Advantages:**
- Fat controllers = fast to develop, easy to grep for "what does X do".
- DB-driven sidebar + RBAC = addons can extend UI without view changes.
- Forward-compatible dispatcher = new channels/providers can be added as addon classes without core changes.
- `student_session` per-year scoping = clean academic-year model.

**Limitations:**
- CodeIgniter 3 is EOL (October 2025) — no security patches.
- Fat controllers become unmaintainable past 1000 lines.
- No service layer / dependency injection / typed DTOs.
- No queue — bulk operations block HTTP request.
- No formal API versioning or spec.
- utf8mb3 collation limits internationalisation.

---

## 8. SaaS and Commercial Model

### 8.1 Subscription plans
**None.** No `plans`, `subscriptions`, `credits` tables exist. The product is sold as one-time CodeCanyon purchases.

### 8.2 Feature entitlements
- **11 advertised addons** form a feature-entitlement catalogue. Each addon is a separately-purchasable feature that, once registered, unlocks its `permission_group` and sidebar entries.
- The bundle pack is a flat-rate all-11-addons purchase — no per-seat or per-student pricing.
- At the code level, `Auth::addonchk('{slug}')` gates every addon controller constructor. Without the addon, the controller redirects to the setting page (which itself doesn't exist) → likely redirect loop.

### 8.3 Usage quotas
- **Project 1's `SaasValidation` library is stubbed** (`saas_enabled=false`, every method returns `TRUE`).
- The dashboard UI iterates `$storageused` (always empty array in this build) to render quota alerts — UI is there, no data backs it.
- No metering of API calls, SMS sends, WhatsApp messages, AI credits (no AI exists).
- `Mailsms.php` calls `$this->saasvalidation->updateResouceQuota('storage', $upload_file_sizes)` in ~15 places, but in this build those are no-ops.

### 8.4 Multi-tenant and multi-branch support
- **No true multi-tenancy in core.** Single-DB, single-school.
- **Multi Branch addon (Project 2 #3)** is the closest — multi-database federation:
  - Each "branch" row in `multi_branch` stores `branch_name`, `branch_url`, `hostname`, `username`, `password`, `database_name`, `directory_path`, `is_verified`. **DB credentials stored in plain text.**
  - Master install's `Branch::switch()` sets a 2-year `branch_cookie` and loads the branch's DB group via `$this->load->database($branch_group, TRUE)`.
  - **Requires manual `database.php` edit** to add a `$db['branch_{id}']` group per branch — branches are not auto-provisioned.
  - Cross-branch reports (`admin/multibranch/finance`) iterate every branch sequentially.
- **Doesn't scale** past ~10 branches (no parallel query, no caching, manual config edit).

### 8.5 Add-on modules
- 11 addons in bundle, each with separate CodeCanyon product ID.
- Installed via in-app ZIP uploader → `multi_query()` SQL execution → file overlay.
- URL-bound AES licensing via `Auth::addonchk()`.

### 8.6 Billing and payment management
- **No SaaS billing** — no invoices for the SaaS itself.
- The `staff_payslip` and `student_fees` tables are for school HR/fees, not SaaS billing.
- No metering → no usage-based billing possible.

### 8.7 White-label functionality
- `sch_settings` supports `name`, `email`, `phone`, `address`, `admin_logo`, `admin_small_logo`, `admin_login_page_background`, `user_login_page_background`, `app_logo`, `app_primary_color_code`, `app_secondary_color_code`, `mobile_api_url`, `admin_mobile_api_url`, `theme` — i.e. each school can brand its install.
- However, the underlying source code is the same for all schools — no true white-label (no per-tenant code customisation).

### 8.8 Tenant provisioning
- **Manual.** Each new school install is a fresh Project 1 deployment (separate DB, separate `license.php`, separate purchase-code registration).
- Multi Branch addon's branch provisioning is also manual (requires `database.php` edit).
- No self-service tenant signup.

### 8.9 Account and organization management
- `sch_settings` row 1 is the single school's config.
- `sessions` table holds academic years.
- No `organizations` or `tenants` table.

### 8.10 Commercial and technical implications for Managora
- **The reference project's commercial model is fundamentally not SaaS** — it's a one-time-purchase self-hosted product with an addon ecosystem. Managora cannot directly adopt this model.
- The technical scaffolding for SaaS (quotas, plans, billing, multi-tenancy) is **stubbed or absent**. Managora would need to build these from scratch.
- The addon-licensing pattern (URL-bound AES) is reversible and not suitable for a true SaaS — signed JWT licenses would be better.

---

## 9. AI, Automation, and Integrations

### 9.1 AI integrations
**There are NO AI capabilities in either Project 1 or any of the 11 Project 2 addons.**

Verification: `grep -rli "openai|chatgpt|gpt-|anthropic|claude|gemini|ai_assist|ai_generate|llm"` over `controllers/`, `libraries/`, `models/`, `helpers/` in both projects returned **zero hits in code** (the only matches were inside language translation strings for unrelated words containing the substring "ai").

- Online Course quiz/exam questions are authored by teachers; no AI generation.
- CBSE Exam marksheet grading uses rule-based grade ranges (`cbse_exam_grades_range`), not AI.
- Behaviour Records incident assignment is manual; no AI-based incident detection.
- WhatsApp Messaging templates are pre-approved by Meta/Twilio; no AI generation.
- Online exam descriptive answers are scored by `answer_word_count` only. No semantic scoring.

The entire combined system is **rule-based**.

### 9.2 Chatbots and conversational interfaces
**None.** No chatbot, no conversational AI, no LLM integration anywhere.

The internal chat module (`chat_connections`, `chat_messages`, `chat_users`) is human-to-human only — student↔staff, staff↔staff. No bot presence.

### 9.3 Workflow automation
- **Cron jobs** (`Cron::index`): autobackup, feereminder, eventreminder, schedulesmsemails — all sequential, all gated by `cron_secret_key`.
- **Cron `student_attendance`**: auto-marks absent for un-submitted class-sections.
- **Fee reminders**: `feereminder_model->get(null, 1)` fetches reminders with `reminder_type` (`before` / `after`) and `day` count; cron sends SMS/Email/WhatsApp per the matrix.
- **Event reminders**: `calendar_model` + cron sends reminders for calendar events.
- **Scheduled SMS/Emails**: `schedulesmsemails` cron dispatches messages scheduled for the current time.
- **No visual workflow builder** — all automation is hard-coded in cron methods.

### 9.4 Triggers and scheduled actions
- **Per-event notification dispatch** is the only "trigger" system: an event key (e.g. `fee_submission`, `exam_result`, `behaviour_incident_assigned`) triggers `Mailsmsconf::mailsms()` which dispatches to enabled channels.
- **No event bus** — events are function calls, not pub/sub.
- **No webhook-out system** — the system receives webhooks (payment gateways) but doesn't emit them.

### 9.5 Webhooks
- **Inbound**:
  - `Webhooks::insta_webhook()` — Instamojo payment status receiver. Verifies MAC via `hash_hmac("sha1", implode("|", $data), $salt)`. Inserts a row in `gateway_ins_response`.
  - 6 `controllers/gateway_ins/` receivers for server-to-server payment callbacks (Ihela, Payfast, Payhere, Skrill, Toyyibpay, Twocheckout).
- **Outbound**: **none**. No Zoom webhook receiver, no Meta WhatsApp inbound-reply handler, no event-subscription system for external integrations.

### 9.6 Third-party APIs
- Zoom REST API v2 (OAuth2, Guzzle) — Zoom addon.
- Google Calendar API + Hangouts Meet (Google Client) — GMeet addon.
- AWS S3 (S3Client) — Online Course addon.
- Firebase FCM v1 (OAuth service-account) — Push notifications.
- Meta Cloud API + Twilio WhatsApp — WhatsApp addon.
- 11 SMS provider APIs — `Smsgateway.php`.
- AWS SES / Mailgun / SMTP — `Mailgateway.php`.
- 28 payment-gateway APIs — `controllers/onlineadmission/*`.

### 9.7 WhatsApp and communication automation
- WhatsApp addon integrates with `notification_setting.is_whatsapp` channel — same matrix as SMS/Email/Push.
- WhatsApp templates are pre-approved by Meta/Twilio; template IDs stored in `notification_setting.whatsapp_template_id`.
- 20 dispatch methods in `Whatsappgateway.php`: `sentRegisterWhatsapp`, `sentAddGroupFeeWhatsapp`, `sentAddFeeWhatsapp`, `sentFeeProcessingNotification`, `sentfeesreminderNotification`, `sendStudentLoginCredential`, `sendStaffLoginCredential`, `student_apply_leave`, `sentExamResultWhatsapp`, `sendPresentAttendancenotification`, `sentPresentStaffWhatsapp`, `sentAbsentStaffWhatsapp`, `sendAbsentAttendancenotification`, `sendstudentlhomework`, `sentOnlineexamStudentWhatsapp` (and 5 more).
- **No inbound WhatsApp reply handling** (no Meta webhook receiver) — communication is one-way.

### 9.8 Opportunities for intelligent agents (Managora inference)
**This is a recommendation, not an existing feature.**

The reference project's rule-based workflows are ripe for AI augmentation. Concrete opportunities Managora could explore:

1. **AI-powered question generation** from a textbook chapter or syllabus topic (the Online Course + CBSE exam modules already have the question-bank schema).
2. **AI-powered descriptive-answer scoring** (the `onlineexam.answer_word_count` field exists but no semantic scoring).
3. **AI-powered attendance anomaly detection** (the `student_attendences` table has the data; a model could flag at-risk students for low attendance).
4. **AI-powered fee-defaulter prediction** (the `student_fees_deposite` history has the data; a model could predict which students are likely to default).
5. **AI-powered parent communication** (auto-draft WhatsApp replies, summarise behaviour incidents for parents).
6. **AI-powered timetable generation** (the `timetable` table exists; a model could auto-generate optimal timetables given teacher availability + room constraints).
7. **AI-powered admission enquiry scoring** (the `enquiry` table has `status ∈ {active, passive, dead, won, lost}` — a model could score new enquiries by conversion likelihood).
8. **AI-powered homework evaluation assistant** (the `homework_evaluation` table has the data; a model could pre-fill evaluations for objective questions).
9. **AI-powered marksheet narrative generation** (the CBSE marksheet has structured marks; an LLM could generate a natural-language summary for parents).
10. **AI-powered chatbot for student/parent queries** ("What's my child's attendance this month?", "When is the next fee due?").

---

## 10. Security, Reliability, and Scalability

### 10.1 Authentication design
- **Two parallel auth systems with different password storage**:
  - Staff: bcrypt via `password_hash(PASSWORD_DEFAULT)` + `password_verify()`. **Correct.**
  - Student/Parent: **plain text** (`users.password varchar(50)`, `==` comparison in `User_model::checkLogin` lines 93, 111, 137). **Critical vulnerability.**
- Login-token rotation on every student/parent login (`users.login_token = bin2hex(random_bytes(32))`).
- CSRF token in every form.
- Optional captcha per page.
- Optional TOTP 2FA (addon).
- `Auth::is_logged_in()` re-checks `staff.is_active == 1` on every request.

### 10.2 Data isolation
- **No tenant isolation in core.** Single-DB, single-school.
- Multi Branch addon uses multi-database (one MySQL DB per branch) — credentials in plain text in `multi_branch` table. **No row-level isolation within a single DB.**
- Teacher record-level access: `MY_Model::applyTeacherClassSectionFilter()` restricts teacher queries to assigned class-sections.

### 10.3 Sensitive information handling
- **Plain-text storage of:**
  - Student/parent passwords (`users.password varchar(50)`).
  - Multi Branch DB credentials (`multi_branch.hostname/username/password/database_name`).
  - WhatsApp API credentials (`whatsapp_config.authkey` — Meta access token or Twilio auth token).
  - AWS S3 credentials (`aws_s3_settings.api_key`/`api_secret`).
  - Zoom OAuth client secret (`zoom_settings.zoom_api_secret`).
  - GMeet OAuth client secret (`gmeet_settings.api_secret`).
  - TOTP secrets (`user_google_authenticate_codes.secret_code`).
  - Payment-gateway API keys/secrets (`payment_settings.api_secret_key`, `salt`, `api_password`).
  - SMTP passwords (`email_config.smtp_password`).
  - SMS provider credentials (`sms_config.authkey`, `password`, `api_id`).
- **Encrypted (but with hardcoded keys):**
  - Verification codes, share-content tokens, fee-receipt tokens, marksheet tokens — all encrypted via `Enc_lib` AES-256-CBC with hardcoded keys `ss@pubkey` / `ss@pvtkey`. Anyone with source can decrypt.

### 10.4 Credential storage
- See §10.3 — multiple categories of credentials stored in plain text.
- `users_authentication.token` (mobile API tokens) stored without hashing.
- No secrets-manager integration (no AWS Secrets Manager, no HashiCorp Vault).

### 10.5 Database integrity
- InnoDB engine with FK constraints in most tables.
- `ON DELETE CASCADE` on addon tables referencing core (`conferences.staff_id → staff.id`, `user_google_authenticate_codes.user_id → users.id`, etc.).
- `utf8mb3` collation (not utf8mb4) — limits emoji and astral-plane characters.
- Inconsistent `is_active` typing (`varchar(255)` in some tables, `int` in others).
- **`student_fees_deposite.amount_detail`** is TEXT-stored JSON — not queryable, not transactional, prone to corruption on concurrent writes.
- **`ONLY_FULL_GROUP_BY` incompatibility** detected at runtime (`views/admin/dashboard.php` warns) but SQL not refactored — schools must disable it at MySQL server level.

### 10.6 Input validation
- CodeIgniter's `form_validation->set_rules()` with callbacks (`callback_handle_upload`, `callback_validate_extension`, `callback_check_captcha`, `callback_validate_storage`) — generally good.
- File uploads enforce MIME/extension/size via `config/app-config.php`.
- **`xss_clean` filter** applied on most form fields.
- **Weaknesses:**
  - `Module_model::getPermissionByModulename()` line 80: raw SQL string concatenation (`"... where short_code='" . $module_name . "'"`).
  - `Customfeesmaster::createInstallment()` uses raw `$_POST['total_fees']`, `$_POST['down_payment']`, etc. instead of `$this->input->post()` — bypasses xss_clean.
  - `addon_helper::execute_sql()` runs untrusted SQL via `multi_query()` — no sandboxing.

### 10.7 Error handling
- CodeIgniter's default error display (suppressed in production via `ENVIRONMENT == 'production'`).
- `application/logs/` for CodeIgniter logs.
- `addons/error_log.txt` for addon installer activity.
- `MY_Model::log()` writes audit entries to `logs` table.
- **SMS gateway calls swallow exceptions**: `catch (Exception $e) { return true; }` — failures not surfaced.
- **Addon installer `multi_query()` can leave DB inconsistent** on partial failure; `execute_sql()` returns status array but caller only logs to `addons/error_log.txt` and does not roll back.
- Zoom library wraps Guzzle requests in try/catch.
- GMeet library handles token expiry via `$client->isAccessTokenExpired()` and refreshes.

### 10.8 Audit logging
- `MY_Model::log()` writes to `logs(message, record_id, user_id, action, ip_address, platform, agent, time)` after every CUD operation across most models (including addon models that extend `MY_Model`).
- `userlog` table for login history.
- `gateway_ins_response` for raw gateway callbacks.
- `conferences_history` / `gmeet_history` for meeting join tracking.
- `logs` table grows unbounded — no archival.

### 10.9 Performance and scalability risks
- `student_fees_deposite.amount_detail` JSON-in-TEXT doesn't scale past a few thousand students per school (fee-collection report queries must deserialize JSON in PHP).
- Multi Branch cross-branch reports load each branch's DB in turn — no parallel query, no caching. Doesn't scale past ~10 branches.
- CBSE rank generation is O(students × subjects × exams) per template — for 1000 students × 10 subjects × 4 exams = 40,000 mark aggregations per click, all in PHP.
- `online_course_processing_payment` table has no TTL — grows unbounded.
- No read replicas, no caching layer (`cache_on = false`).
- No queue for email/SMS/WhatsApp — bulk sends block the request thread.
- Fat controllers (`Welcome.php` 1715 lines, `Smsgateway.php` 3145 lines) — high memory per request.

### 10.10 Backup and recovery mechanisms
- `Cron::autobackup()` writes a `.sql` dump to `./backup/database_backup/db_ver_{ver}_{ts}.sql` using CodeIgniter's `dbutil->backup()`.
- Manual backup via `admin/admin/backup` — downloads a `.sql` file.
- No automated restore UI — admin must manually import the SQL.
- No incremental backups, no off-site backup, no point-in-time recovery.
- The `backup/database_backup/` directory grows unbounded — no rotation.

### 10.11 Confirmed vulnerabilities vs potential risks vs unverified concerns
**Confirmed vulnerabilities (visible in source):**
- Plain-text student/parent passwords.
- Plain-text Multi Branch DB credentials.
- Plain-text API credentials (WhatsApp, AWS S3, Zoom, GMeet, payment gateways, SMTP, SMS).
- Hardcoded AES keys in `Enc_lib`.
- Hardcoded Zoom bearer token in `App::zoom()`.
- `multi_query()` installer with no sandbox.
- SQL string concatenation in `Module_model::getPermissionByModulename`.
- SMS failures silently swallowed.
- TOTP secrets stored plain-text.
- Leaky addon boundary (`Welcome::cbseexam()` loads CBSE classes without `hasActive` guard).

**Potential risks (likely but not verified end-to-end):**
- Concurrent fee payment overwriting `amount_detail` JSON (no transactional append).
- Multi Branch license check failing when switched to a branch (addon licensed per-URL, not per-DB).
- Addon installer partial failure leaving DB in inconsistent state.
- Zoom/GMeet tokens expiring mid-session (GMeet) or being revoked (no refresh job).

**Unverified concerns:**
- Vendor's licensing server behaviour (does it validate `base_url` against Envato, or trust the client?).
- Mobile app's API consumption patterns (Flutter source not in package).
- Whether the vendor's hosted distribution hashes passwords differently from the open-source distribution.
- Whether addon installer SQL files are audited before publication (Thermal Print seed has real customer data — suggests no sanitisation review).

---

## 11. Reusable Concepts for Managora

> For each concept: Concept, Purpose, Reusable lesson, Managora application, Suggested approach, Dependencies, Complexity, Priority, and Adopt/Adapt/Avoid/Investigate classification.

### 11.1 Per-event × per-channel × per-recipient notification matrix
- **Concept:** One `notification_setting` row per event key with booleans for each (channel × recipient-class) combination, plus a `{{var}}`-templated body and a `variables` CSV list. A single `mailsms($event_key, $sender_details)` call dispatches to all enabled channels.
- **Purpose:** Schools need fine-grained control over what gets sent where. Regulators require DLT template IDs for SMS in India; US schools want email-only for some events; parents may opt out of push.
- **Reusable lesson:** Decouple event triggers from channel dispatch. Per-event × per-channel × per-recipient-class toggles are the right abstraction.
- **Managora application:** All communication — fee receipts, exam results, attendance alerts, behaviour incidents, batch reminders, admission confirmations.
- **Suggested approach:** Model as `NotificationTemplate(event, channel, recipient_role, locale, body, dlt_template_id, whatsapp_template_id, active)`. Add per-user opt-out flags. Use a job queue (Sidekiq / Celery / SQS) per channel. Add AI channel for natural-language summaries. Add a per-tenant template override.
- **Dependencies:** Channel providers (SMS/Email/Push/WhatsApp/AI); event registry; user opt-out preferences; per-tenant template overrides.
- **Complexity:** Medium.
- **Priority:** Critical.
- **Classification:** **Adopt** (with modern stack: queue, per-user opt-out, AI channel).

### 11.2 Per-year enrollment (`student_session`) as the unit of scope
- **Concept:** `student_session` is a per-year record linking `student × session × class × section × hostel × transport route × pickup point`. All operational queries filter by `session_id = current_session`.
- **Purpose:** Almost every report is scoped to an academic year. Promoting students to the next class should not lose history.
- **Reusable lesson:** Use a per-year enrollment record as the FK for all operational data, not the student directly.
- **Managora application:** Attendance, fees, exams, behaviour, batches, enrollments — all scoped to (tenant × academic_year × student).
- **Suggested approach:** Composite `(student_id, academic_year_id, tenant_id)` FK on every operational table. Provide a "promote students to next session" admin action that creates new `student_session` rows copying previous year's class/section defaults. Add per-tenant academic-year calendar with multiple sessions (e.g. summer batch + fall batch for coaching institutes).
- **Dependencies:** Tenant model; academic-year model; class/section master; promotion workflow.
- **Complexity:** Low.
- **Priority:** Critical.
- **Classification:** **Adopt**.

### 11.3 Hierarchical exam model with multiple grading schemes
- **Concept:** Five-level hierarchy `exam_group → exam → schedule → marks → result → rank` with 5 grading systems (`basic_system`, `school_grade_system`, `coll_grade_system`, `gpa`, `average_passing`) and mark divisions.
- **Purpose:** Real schools have "Term 1 → Unit Test 1 → Math → 80 marks → result". A flat `exams` table cannot model this.
- **Reusable lesson:** Hierarchical assessment structure with pluggable grading schemes is necessary for any education platform.
- **Managora application:** Examination module for coaching institutes (mock tests, unit tests, full-length exams) and schools.
- **Suggested approach:** Generalise into `Assessment → AssessmentComponent → AssessmentComponentSubject → StudentScore`. Add a `grading_scheme` reference table. Add AI-powered marksheet narrative generation. Add per-tenant grading scheme customisation.
- **Dependencies:** Student model; subject model; class-section model; academic-year model.
- **Complexity:** High.
- **Priority:** High.
- **Classification:** **Adapt** (the schema is verbose; refactor into a cleaner hierarchy).

### 11.4 Forward-compatible dispatcher + provider plugins
- **Concept:** Core ships a dispatcher that calls `{$channel}gateway->{method}()` for each channel. Each channel's gateway class is provided by a separate addon. The dispatcher is guarded by a per-event toggle.
- **Purpose:** New channels/providers can be added as addon classes without core code changes.
- **Reusable lesson:** Service-locator pattern for channels — core calls `$channelRegistry->channel('whatsapp')->send(...)`, returns null if not registered.
- **Managora application:** SMS, Email, Push, WhatsApp, AI chat, payment gateways, cloud storage, live-class providers — all behind a unified registry.
- **Suggested approach:** `ChannelRegistry` service; plugins register their channel implementation on boot. Per-tenant channel configuration (e.g. tenant A uses Twilio, tenant B uses MSG91). Add AI as a first-class channel (for natural-language notifications and chatbot responses).
- **Dependencies:** Plugin system; per-tenant configuration; channel credential vault.
- **Complexity:** Medium.
- **Priority:** High.
- **Classification:** **Adopt**.

### 11.5 Module-toggle + sub-feature permission matrix
- **Concept:** `permission_group` (module-level toggle, e.g. `fees_collection`, `behaviour_records`) + `permission_category` (sub-features with `can_view/add/edit/delete` booleans) + `roles_permissions` (per-role grants).
- **Purpose:** Each addon declares its own RBAC model declaratively; core enforces uniformly.
- **Reusable lesson:** Plugins/modules declare their permission catalogue in code; runtime uses a `Policy` class per resource.
- **Managora application:** RBAC for all Managora modules — institute admin, teacher, student, parent, accountant, receptionist, franchise owner, branch manager.
- **Suggested approach:** `PermissionCatalog` that plugins declare via `Plugin::permissions(): array`. Default grants in a migration. Runtime `Policy` class per resource (Laravel-style) or decorator (NestJS-style). Add per-tenant role customisation (tenant admin can create custom roles). Add AI-assisted permission recommendations ("this role typically has X, Y, Z").
- **Dependencies:** Plugin system; tenant model; role model.
- **Complexity:** Medium.
- **Priority:** High.
- **Classification:** **Adopt** (as code, not SQL seeds).

### 11.6 Tokenised shareable PDF links
- **Concept:** `enc_lib->encrypt(implode(",", [$id1, $id2, ...]))` becomes a URL token; receiver hits a route that decrypts and serves the PDF.
- **Purpose:** Schools email parents PDF receipts/marksheets; they don't want to attach large files but also don't want a public URL.
- **Reusable lesson:** Time-windowed, intent-bound, signed URLs for document sharing.
- **Managora application:** Fee receipts, marksheets, transfer certificates, ID cards, report cards, progress reports — all shareable via signed URLs.
- **Suggested approach:** Use **signed JWTs** with short expiry (e.g. 7 days) instead of reversible AES. Bind to `{document_id, tenant_id, expiry, intent}`. Log every download. Add AI-generated cover note ("Here's your child's Term 1 marksheet...").
- **Dependencies:** PDF generation; JWT signing; audit log.
- **Complexity:** Low.
- **Priority:** High.
- **Classification:** **Adapt** (use signed JWTs, not reversible AES).

### 11.7 Co-scholastic observation framework (CBSE)
- **Concept:** `cbse_observation_parameters → cbse_observation_subparameter → cbse_observation_terms → cbse_observation_term_student_subparameter` for non-academic assessment (teamwork, discipline, etc.).
- **Purpose:** Indian K-12 boards require co-scholastic assessment alongside academic marks.
- **Reusable lesson:** Generalise into a `Rubric` system that can model any multi-criteria, per-student, per-term assessment.
- **Managora application:** Behaviour/soft-skills tracking for coaching institutes (participation, teamwork, leadership, attendance attitude); IB ATL skills for international schools; Montessori competencies.
- **Suggested approach:** `Rubric` (id, name, tenant_id) → `RubricCriterion` (id, rubric_id, name, max_score, weight) → `RubricAssessment` (rubric_id, student_id, term_id, assessor_id) → `RubricScore` (assessment_id, criterion_id, score, comment). Add AI-powered rubric scoring from observation notes.
- **Dependencies:** Student model; term model; staff model.
- **Complexity:** Medium.
- **Priority:** Medium.
- **Classification:** **Adapt** (generalise from CBSE-specific to generic rubric).

### 11.8 LMS-style course structure with paid checkout
- **Concept:** `online_courses → online_course_section → course_lesson_quiz_order → lessons/quizzes/exams/assignments`, with `course_progress`, `student_quiz_status`, `online_course_exam_attempts`, `online_course_assignment_submit`, certificate generation. Cart-based checkout with 27-gateway payment + idempotent `online_course_processing_payment` table.
- **Purpose:** Schools increasingly sell courses to non-students (MOOC-style) as a revenue stream.
- **Reusable lesson:** Course structure (section → lesson/quiz/exam/assignment → progress → certificate) with idempotent payment is a complete MOOC-in-a-box.
- **Managora application:** Course-seller module for coaching institutes and individual instructors.
- **Suggested approach:** Use SCORM/xAPI standards for content packaging. Implement content versioning so updates to a lesson don't break in-progress students. Add drip scheduling (release one section per week). Support cohort-based courses with start/end dates. Add AI-powered course outline generation, quiz generation from lesson content, and personalised learning paths.
- **Dependencies:** Payment gateway abstraction; cloud storage; user model (including guest); certificate generation.
- **Complexity:** High.
- **Priority:** Medium (depends on whether Managora targets course sellers early).
- **Classification:** **Adapt**.

### 11.9 Per-teacher OAuth for third-party services
- **Concept:** Each teacher can connect their own Zoom/Google account via OAuth; the school's global account is the fallback (`zoom_settings.use_teacher_api` flag).
- **Purpose:** Lets users connect their own third-party accounts without sharing passwords; avoids the school needing a paid Zoom license for every teacher.
- **Reusable lesson:** Per-user OAuth + per-tenant fallback for third-party services.
- **Managora application:** Zoom, Google Meet, Google Calendar, Microsoft Teams, Google Drive, Dropbox — per-teacher connections.
- **Suggested approach:** Store tokens encrypted per-user in DB (not in session). Implement a token-refresh job (cron or queue) that proactively refreshes tokens before expiry. Handle revocation gracefully ("reconnect" banner). Per-tenant fallback credential.
- **Dependencies:** OAuth2 library; secrets manager; token-refresh job; per-tenant config.
- **Complexity:** Medium.
- **Priority:** Medium.
- **Classification:** **Adapt** (persist tokens in DB, not session).

### 11.10 TOTP 2FA with per-user secrets
- **Concept:** RFC 6238 TOTP via Google Authenticator; per-user 16-char base32 secret generated via `random_bytes()`; QR code for setup; admin global toggle; per-user opt-in.
- **Purpose:** NIST 800-63B AAL2 requires a second factor.
- **Reusable lesson:** Standard TOTP implementation with per-user secrets and admin toggle is the right pattern.
- **Managora application:** Staff, tenant admin, and (optionally) student/parent 2FA.
- **Suggested approach:** Same pattern, but (a) **encrypt the secret at rest** with a per-user key derived from the user's password hash (so changing the password rotates the encryption); (b) support WebAuthn / FIDO2 as a modern alternative; (c) support backup codes; (d) per-tenant 2FA policy (mandatory for tenant admin, optional for staff).
- **Dependencies:** User model; encryption; QR code generation; backup code generation.
- **Complexity:** Low.
- **Priority:** High.
- **Classification:** **Adapt** (encrypt secrets; add WebAuthn + backup codes).

### 11.11 QR-code self-scan attendance
- **Concept:** Each student/staff ID card has a QR code (encoding `admission_no`). A web view uses `getUserMedia` to scan the QR, decodes the admission_no, looks up the student, marks attendance with `qrcode_attendance=1` flag.
- **Purpose:** Cheap hardware (any phone camera) vs biometric devices.
- **Reusable lesson:** Leverage existing ID-card QR codes for self-scan attendance — no new hardware.
- **Managora application:** Attendance for coaching institutes (students scan in/out of batch sessions); staff attendance.
- **Suggested approach:** Progressive web app (PWA) with offline QR scanning (Barcode Detection API or `quagga.js` cached for offline). Push attendance in batches when connectivity returns. **Use signed JWTs in the QR** (not raw admission_no) to prevent forgery. Add AI-powered attendance anomaly detection.
- **Dependencies:** ID card generation; PWA; signed JWT; attendance model.
- **Complexity:** Medium.
- **Priority:** Medium.
- **Classification:** **Adapt** (use signed JWTs, PWA, offline).

### 11.12 Cross-branch / cross-tenant aggregated reports
- **Concept:** Multi Branch addon's `Finance.php` iterates branches, loads each DB group, runs the same query, aggregates in PHP.
- **Purpose:** Multi-school owners want a consolidated view.
- **Reusable lesson:** Cross-tenant aggregation is a real need; multi-DB iteration is the wrong implementation.
- **Managora application:** Franchise owner dashboard ("how are all my branches doing?"); multi-institute org dashboard.
- **Suggested approach:** Replace multi-DB iteration with a **data warehouse** pattern: nightly ETL from each tenant DB into a consolidated analytics schema (Star Schema). Use a BI tool (Metabase, Superset) for queries. Real-time consolidated dashboards via read replicas + federated query engine (Trino, Presto). Add AI-powered narrative summaries ("Branch 3's fee collection is down 20% this month — likely cause: 5 students deferred").
- **Dependencies:** Data warehouse; ETL pipeline; BI tool; per-tenant data isolation in warehouse.
- **Complexity:** High.
- **Priority:** Medium (only relevant for franchise / multi-branch customers).
- **Classification:** **Adapt** (use data warehouse, not multi-DB iteration).

### 11.13 Auto-numbering with prefix + digit length
- **Concept:** `sch_settings.adm_prefix`, `adm_no_digit`, `adm_start_from`, `adm_auto_insert` flag — generates `ssadm19/20/000001`-style admission numbers. Same pattern for staff IDs.
- **Purpose:** Admission numbers follow school-specific formats.
- **Reusable lesson:** Per-tenant, per-entity sequence with prefix + suffix + digit length + padding.
- **Managora application:** Admission numbers, staff IDs, receipt numbers, invoice numbers, transfer certificate numbers — all per-tenant auto-generated.
- **Suggested approach:** `Sequence(tenant_id, entity, prefix, suffix, digits, next_value, padding_char, reset_per_year)` table. Use SQL `SELECT … FOR UPDATE` to allocate atomically. Per-tenant customisation. Add AI-powered format suggestion based on tenant's country/institute type.
- **Dependencies:** Tenant model; sequence table; atomic allocation.
- **Complexity:** Low.
- **Priority:** High.
- **Classification:** **Adopt** (generalise to per-tenant sequence).

### 11.14 Field-visibility toggles + custom-field builder
- **Concept:** `sch_settings` has 60+ boolean columns (`is_blood_group`, `student_email`, `mobile_no`, `father_pic`, …) that gate form fields. Plus a `custom_fields` table + `libraries/Form_builder.php` (40 KB) for admin-defined custom fields.
- **Purpose:** Different schools collect different fields without code changes.
- **Reusable lesson:** Per-tenant form schema with standard field toggles + custom field definitions.
- **Managora application:** Student admission form, staff profile form, enquiry form — all per-tenant customisable.
- **Suggested approach:** Replace boolean columns with a `FieldDefinition(entity, key, type, label, required, visible, default, tenant_id)` table driven by a JSON-Schema or Form-Builder library. Allow per-tenant overrides. Add AI-powered field suggestions ("coaching institutes typically collect X, Y, Z").
- **Dependencies:** Form builder library; per-tenant config; entity model.
- **Complexity:** Medium.
- **Priority:** Medium.
- **Classification:** **Adapt** (replace booleans with FieldDefinition table).

### 11.15 Audit log with role-context, IP, platform, agent
- **Concept:** `MY_Model::log()` writes to `logs(message, record_id, user_id, action, ip_address, platform, agent, time)` after every CUD operation across most models.
- **Purpose:** Schools need compliance (FERPA / GDPR / local data-protection laws).
- **Reusable lesson:** Structured audit log with user + IP + agent + action + record_id is necessary for compliance.
- **Managora application:** Compliance logging for all CUD operations across all modules.
- **Suggested approach:** Use structured event logging (OpenTelemetry logs → Loki / Elasticsearch) with searchable JSON. Retain per regulation. Add AI-powered anomaly detection ("user X accessed 500 student records in 10 minutes — flag for review").
- **Dependencies:** Logging infrastructure; retention policy; per-tenant log isolation.
- **Complexity:** Low.
- **Priority:** High.
- **Classification:** **Adopt** (with modern logging stack).

### 11.16 Multi-channel login (username / admission_no / mobile / email)
- **Concept:** `User_model::checkLogin()` accepts username OR admission_no OR mobile_number OR email — controlled by `sch_settings.student_login` JSON list.
- **Purpose:** Students/parents remember different identifiers; schools configure which are valid.
- **Reusable lesson:** Per-tenant login-identifier configuration.
- **Managora application:** Student/parent login for coaching institutes (mobile + email), schools (admission_no + email).
- **Suggested approach:** Per-tenant `LoginIdentifierConfig` table; `AuthService::resolveLoginIdentifier($input)` returns matching user. Add passwordless login (magic link via WhatsApp/Email). Add AI-powered login anomaly detection.
- **Dependencies:** User model; per-tenant config; passwordless flow.
- **Complexity:** Low.
- **Priority:** Medium.
- **Classification:** **Adapt** (add passwordless + anomaly detection).

### 11.17 Per-event × per-channel WhatsApp template integration
- **Concept:** `notification_setting.whatsapp_template_id` per event; `Whatsappgateway::sentExamResultWhatsapp()` etc. dispatch to Meta Cloud API with template + parameters.
- **Purpose:** Meta and Twilio require pre-approved templates; per-event template binding is the right abstraction.
- **Reusable lesson:** Per-event, per-channel template binding with parameter substitution.
- **Managora application:** All WhatsApp notifications.
- **Suggested approach:** Per-tenant template registry; per-event template binding; parameter substitution engine. Add AI-powered template generation ("draft a fee-reminder WhatsApp message"). Add template-approval-status tracking (Meta's approval flow).
- **Dependencies:** WhatsApp Business API account; template approval flow; per-tenant config.
- **Complexity:** Medium.
- **Priority:** Medium.
- **Classification:** **Adopt**.

### 11.18 Cron + secret-key gating
- **Concept:** All cron entry points require `sch_settings.cron_secret_key` in the URL. Cron routes: `cron/{key}` → `index()` runs 4 jobs sequentially.
- **Purpose:** Prevents unauthorised cron triggering; lets schools host cron externally.
- **Reusable lesson:** Secret-key-gated cron endpoints are a simple way to secure scheduled jobs in a self-hosted context.
- **Managora application:** For self-hosted Managora deployments (if any); for SaaS, use a proper job queue instead.
- **Suggested approach:** In a SaaS context, replace cron with a proper job queue (Sidekiq / Celery / SQS) with per-tenant job workers. Keep secret-key gating only for self-hosted deployments.
- **Dependencies:** Job queue; per-tenant workers.
- **Complexity:** Low.
- **Priority:** Low (for SaaS).
- **Classification:** **Adapt** (use queue, not cron).

### 11.19 Theme system with live toggle
- **Concept:** 6 admin themes + live toggles for dark/light, container-fluid, sidebar collapse, shadow — all configurable in `sch_settings` and applied client-side from a JSON blob.
- **Purpose:** Per-school branding + user preference.
- **Reusable lesson:** Client-side theme application from a JSON config; no page reload.
- **Managora application:** Per-tenant branding (colours, logo, layout) + per-user theme preference.
- **Suggested approach:** Per-tenant theme config + per-user preference (dark/light, density). Use CSS variables for instant theme switching. Add AI-powered theme suggestion based on tenant's logo colours.
- **Dependencies:** CSS variables; per-tenant config; per-user preference.
- **Complexity:** Low.
- **Priority:** Medium.
- **Classification:** **Adopt**.

### 11.20 ID-card QR code generation + reuse for attendance
- **Concept:** Project 1's ID-card module generates QR codes on ID cards (encoding `admission_no`). Project 2 #6's QR Attendance addon scans the existing QR codes — doesn't generate new ones.
- **Purpose:** Leverage existing output for new use cases; no new hardware.
- **Reusable lesson:** Design ID cards / receipts / certificates with embedded machine-readable tokens that other modules can reuse.
- **Managora application:** ID cards with signed JWT QR (reusable for attendance, library check-out, fee payment confirmation at counter).
- **Suggested approach:** Generate ID cards with signed JWT QR encoding `{student_id, tenant_id, expiry, intent='id_card'}`. Reuse for attendance, library, fee payment confirmation. Rotate JWT on ID card reissue.
- **Dependencies:** ID card generation; signed JWT; scanner module.
- **Complexity:** Medium.
- **Priority:** Medium.
- **Classification:** **Adopt** (with signed JWT QR).

---

## 12. Concepts to Avoid

### 12.1 Plain-text password storage
- **What:** `users.password varchar(50)` with `==` comparison in `User_model::checkLogin` lines 93, 111, 137.
- **Why avoid:** Violates every modern security baseline (OWASP, NIST 800-63B). A single DB dump exposes every student/parent password.
- **Managora alternative:** `password_hash(PASSWORD_DEFAULT)` + `password_verify()` for all user types. Migrate any legacy plain-text passwords on next login (force reset).

### 12.2 Multi-database multi-tenancy with plain-text credentials
- **What:** Multi Branch addon stores `hostname`, `username`, `password`, `database_name` in plain text in `multi_branch` table.
- **Why avoid:** Any admin with `multi_branch_overview` can read every branch's DB password. Doesn't scale past ~10 branches. Requires manual `database.php` edit per branch.
- **Managora alternative:** Single-DB multi-tenancy with `tenant_id` column on every table + row-level security (RLS). For data-residency requirements, use a tenant-resolver middleware that picks DB connection per request based on subdomain, with credentials in a secrets manager (AWS Secrets Manager, HashiCorp Vault).

### 12.3 `multi_query()` SQL installer with no rollback
- **What:** `helpers/addon_helper.php::execute_sql()` uses `mysqli::multi_query()` to execute addon SQL.
- **Why avoid:** A malicious or buggy SQL file can DROP tables, INSERT admin users, or escalate privileges. No sandboxing, no rollback on partial failure.
- **Managora alternative:** Proper migration framework (Phinx / Liquibase / Prisma migrate) with version tracking, per-statement execution, and rollback support. Plugins ship migration files, not raw SQL.

### 12.4 Hardcoded AES keys for "encrypted" tokens
- **What:** `Enc_lib` uses `$pub_key = 'ss@pubkey'` and `$pvt_key = 'ss@pvtkey'`.
- **Why avoid:** Anyone with source can decrypt any encrypted token (verification codes, share-content tokens, fee-receipt tokens, marksheet tokens).
- **Managora alternative:** Use **signed JWTs** (RS256/ES256) for tokens — no shared secret in source code. For at-rest encryption, use per-tenant data keys wrapped by a master key in a KMS.

### 12.5 SMS/WhatsApp gateway failures silently swallowed
- **What:** `catch (Exception $e) { return true; }` in `Smsgateway::sendSMS()` for Clickatell; `if ($response->IsError) { return true; }` for Twilio.
- **Why avoid:** Parents never know an SMS didn't send; admins can't see it in reports; fee reminders fail silently.
- **Managora alternative:** All channel dispatches return a `DeliveryResult` object with `status, error_message, provider_response, retry_count`. Failed deliveries are logged + retried via queue. Admin dashboard shows delivery success rate per channel.

### 12.6 Leaky addon/module boundary
- **What:** `Welcome::cbseexam()` (line 1551) loads CBSE addon classes without checking `module_lib->hasActive('sscbse')`. Route `/cbseexam` registered unconditionally → fatal error without addon.
- **Why avoid:** Core code directly references module-specific classes; missing module causes fatal errors instead of graceful 404.
- **Managora alternative:** All module-specific endpoints/routes must be registered by the module itself (via a `RouteRegistrar`), not by core. Core code never directly references module classes — always via interfaces or service-locator.

### 12.7 URL-bound AES licensing
- **What:** `Auth::addonchk('{slug}')` decrypts `addon_ver` entries with `addon_prod` keys (AES) and parses `"slug!!base_url()"`.
- **Why avoid:** Reversible AES with keys in source is not real licensing. Base_url spoofing via reverse proxy or `$_SERVER['SERVER_NAME']` edit bypasses it.
- **Managora alternative:** For SaaS, no licensing needed — entitlements are per-tenant rows in DB. For self-hosted, use **signed JWT licenses** (RS256 with vendor private key) carrying `{slug, base_url, expiry, plan}`, verified with vendor's public key. Add online revocation via short expiry (7 days) + refresh endpoint.

### 12.8 Fat-controller pattern
- **What:** `Welcome.php` 1715 lines, `Site.php` 1003, `Customlib.php` 2493, `Smsgateway.php` 3145, `Mailsmsconf.php` 78KB, CBSE `Exam.php` 1600+ lines, CBSE `Result.php` 2400+ lines.
- **Why avoid:** Unmaintainable past 1000 lines; high memory per request; hard to test.
- **Managora alternative:** Service layer + thin controllers + DTOs + typed interfaces. Each service < 500 lines. Refactor fat controllers into multiple services.

### 12.9 TEXT-stored JSON for transactional data
- **What:** `student_fees_deposite.amount_detail` is a TEXT column holding a JSON history of every payment.
- **Why avoid:** Not queryable (can't `SELECT WHERE amount > 1000`); not transactional (concurrent appends risk overwriting); prone to corruption.
- **Managora alternative:** Proper child table `fee_payment(student_fees_master_id, amount, discount, fine, payment_mode, date, description, received_by, created_at)` with indexes. Use JSON columns only for non-transactional metadata (e.g. user preferences).

### 12.10 utf8mb3 collation
- **What:** `utf8mb3 / utf8mb3_general_ci` throughout `database.sql`.
- **Why avoid:** Limits emoji and astral-plane characters in student names, chat messages, etc.
- **Managora alternative:** `utf8mb4 / utf8mb4_0900_ai_ci` (MySQL 8+) or `utf8mb4_unicode_ci` (MySQL 5.7).

### 12.11 Code duplication across payment gateways / SMS providers
- **What:** 28 payment-gateway controllers in P1 + 27 in P2 #5 = 55 near-identical files. SMS provider dispatch duplicated 9× per method in `Smsgateway.php`.
- **Why avoid:** Bug fixes require touching 55 files. Adding a new gateway requires copy-pasting an existing one.
- **Managora alternative:** Abstract payment gateway behind a `PaymentGatewayInterface` with `createCharge()`, `verifyWebhook()`, `refund()`. Each gateway is one class implementing the interface. SMS providers behind a `SmsProviderInterface`. Register in a `ProviderRegistry`.

### 12.12 Cron as a single HTTP hit running multiple jobs sequentially
- **What:** `Cron::index()` runs `autobackup`, `feereminder`, `eventreminder`, `schedulesmsemails` sequentially.
- **Why avoid:** Long-running jobs can time out; one job's failure blocks the rest.
- **Managora alternative:** Job queue (Sidekiq / Celery / SQS) with per-job workers. Each job is independent + retryable.

### 12.13 Inconsistent `is_active` typing
- **What:** `is_active` is `varchar(255)` in many tables (`students`, `feemasters`, `fee_groups_feetype`, etc.), `int` in others (`staff`, `multi_branch`).
- **Why avoid:** Confusing — `is_active='yes'` vs `is_active=1`. Index inefficiency.
- **Managora alternative:** Consistent `boolean` or `tinyint(1)` for all `is_active` / `is_published` / `is_deleted` fields.

### 12.14 `created_at ON UPDATE CURRENT_TIMESTAMP` quirk
- **What:** CBSE and Zoom tables use `created_at DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`, which means `created_at` actually reflects the last update, not creation.
- **Why avoid:** Breaks audit-trail correctness — you can't tell when a row was actually created.
- **Managora alternative:** Separate `created_at` (default `CURRENT_TIMESTAMP`, no update) and `updated_at` (default `CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`) columns on every table.

### 12.15 No outbound webhook system
- **What:** The system receives webhooks (payment gateways) but doesn't emit them.
- **Why avoid:** Managora customers will want to integrate with their own CRM / marketing automation / analytics tools.
- **Managora alternative:** Outbound webhook system with per-tenant webhook subscriptions (`WebhookSubscription(tenant_id, event, url, secret, active)`). Sign payloads with HMAC-SHA256. Retry with exponential backoff.

### 12.16 Hardcoded third-party credentials in source
- **What:** `controllers/App::zoom()` uses hardcoded `Authorization: Bearer sY6xc8tAS7Wj8-MXyXxheg` Zoom API key.
- **Why avoid:** Leaked credential; will be revoked; anyone with source has access.
- **Managora alternative:** All third-party credentials in environment variables or secrets manager. Never in source code.

### 12.17 Vendor lock-in to a single licensing server
- **What:** `Auth::addon_update_check()` posts to a remote URL controlled by QDOCS.
- **Why avoid:** If the vendor's server is down, addon updates break. If the vendor revokes access, the install stops working.
- **Managora alternative:** For SaaS, no licensing server needed — entitlements are per-tenant rows. For self-hosted, use signed JWT licenses that don't require server validation (offline verification with public key).

### 12.18 No test suite / no CI/CD
- **What:** No `tests/` directory in Project 1 or any addon. No CI configuration.
- **Why avoid:** Regressions go undetected; refactoring is risky.
- **Managora alternative:** PHPUnit / Pest tests for services; Cypress / Playwright for E2E; CI pipeline (GitHub Actions / GitLab CI) with mandatory tests on every PR.

---

## 13. New Opportunities and Recommendations

> **Recommendations and inferences labelled as such — not existing project features.**

### 13.1 AI-first communication (Managora recommendation)
**Opportunity:** The reference project's notification matrix is rule-based. Managora could add an **AI channel** alongside SMS/Email/Push/WhatsApp — an LLM that:
- Drafts personalised messages per recipient (e.g. "Hi Mrs. Sharma, Aarav's attendance is at 78% this month, below the 80% threshold...").
- Summarises multi-event notifications into a daily digest ("Today: Aarav was marked present, fee receipt #1234 generated, maths homework assigned").
- Translates messages into the parent's preferred language automatically.
- Suggests reply actions for parents ("Reply 'CONFIRM' to acknowledge" — handled by AI agent).

**Managora application:** All parent communication.

**Dependencies:** LLM provider (OpenAI / Anthropic / local); per-recipient language preference; per-tenant AI credit budget; message logging.

**Complexity:** High.

**Priority:** High (differentiator vs. rule-based competitors).

### 13.2 AI-powered question generation (Managora recommendation)
**Opportunity:** The reference project's `questions` table is populated manually by teachers. Managora could add an **AI question generator** that:
- Takes a syllabus topic + difficulty level + question type (MCQ / short answer / long answer) and generates questions.
- Generates answer keys + marking schemes.
- Generates distractors for MCQs.
- Tags questions by Bloom's taxonomy level.
- Imports questions from a textbook PDF.

**Managora application:** Examination module + Online Course module (quizzes/exams).

**Dependencies:** LLM provider; syllabus model; question bank schema; teacher review workflow.

**Complexity:** Medium.

**Priority:** High.

### 13.3 AI-powered descriptive-answer scoring (Managora recommendation)
**Opportunity:** The reference project's `onlineexam.answer_word_count` is the only descriptive-answer check. Managora could add **AI semantic scoring** that:
- Compares student answer against model answer + key concepts.
- Assigns partial marks for partially-correct answers.
- Flags answers for teacher review when confidence is low.
- Provides feedback to students ("You missed the concept of X").

**Managora application:** Online Course exams + CBSE marksheet addon equivalent.

**Dependencies:** LLM provider; rubric model; teacher review workflow.

**Complexity:** Medium.

**Priority:** Medium.

### 13.4 AI-powered attendance anomaly detection (Managora recommendation)
**Opportunity:** The reference project's `student_attendences` data is used only for reports. Managora could add an **AI anomaly detector** that:
- Flags students with attendance below a per-tenant threshold.
- Predicts students at risk of dropping out (coaching institutes care about this).
- Suggests interventions ("Aarav has missed 3 consecutive classes — send a WhatsApp reminder?").
- Detects unusual patterns (e.g. mass absenteeism on a specific day).

**Managora application:** Attendance module.

**Dependencies:** ML model; attendance data; intervention workflow; notification matrix.

**Complexity:** Medium.

**Priority:** Medium.

### 13.5 AI-powered fee-defaulter prediction (Managora recommendation)
**Opportunity:** The reference project's `student_fees_deposite` history has payment patterns. Managora could add an **AI defaulter predictor** that:
- Predicts which students are likely to default on upcoming installments.
- Suggests payment plans (e.g. "offer Aarav a 3-month EMI instead of lump sum").
- Triggers proactive WhatsApp reminders to high-risk students.

**Managora application:** Fees module.

**Dependencies:** ML model; payment history; payment plan engine; notification matrix.

**Complexity:** Medium.

**Priority:** Medium.

### 13.6 AI-powered timetable generation (Managora recommendation)
**Opportunity:** The reference project's `timetable` table is populated manually. Managora could add an **AI timetable generator** that:
- Takes teacher availability + room availability + subject requirements + class-section constraints.
- Generates an optimal timetable minimising conflicts.
- Allows teacher preferences ("no Monday morning classes").
- Auto-adjusts on teacher leave / room unavailability.

**Managora application:** Academics module.

**Dependencies:** Constraint solver (or LLM with constraint prompting); teacher availability model; room model; class-section model.

**Complexity:** High.

**Priority:** Medium.

### 13.7 AI-powered admission enquiry scoring (Managora recommendation)
**Opportunity:** The reference project's `enquiry` table has a manual `status ∈ {active, passive, dead, won, lost}`. Managora could add an **AI enquiry scorer** that:
- Scores new enquiries by conversion likelihood based on source, demographics, engagement.
- Suggests next actions ("call within 24 hours", "send WhatsApp brochure").
- Auto-assigns high-score enquiries to specific counsellors.
- Generates a daily "hot leads" report.

**Managora application:** Front Office / Admissions module.

**Dependencies:** ML model; enquiry data; counsellor assignment workflow.

**Complexity:** Medium.

**Priority:** Medium.

### 13.8 AI-powered marksheet narrative generation (Managora recommendation)
**Opportunity:** The reference project's CBSE marksheet has structured marks. Managora could add an **AI narrative generator** that:
- Generates a natural-language summary for parents ("Aarav scored 85% in Term 1, an improvement of 5% over Term 0. Strengths: Mathematics (92%). Areas for improvement: Science (78%)").
- Suggests next-term focus areas.
- Compares performance to class average.

**Managora application:** Examination module.

**Dependencies:** LLM provider; marks data; class average computation.

**Complexity:** Low.

**Priority:** Medium.

### 13.9 AI-powered chatbot for student/parent queries (Managora recommendation)
**Opportunity:** The reference project has no chatbot. Managora could add an **AI chatbot** that:
- Answers student/parent queries in natural language ("What's my child's attendance this month?", "When is the next fee due?", "What's the syllabus for next week's test?").
- Escalates to staff for complex queries.
- Logs all conversations for audit.
- Operates via WhatsApp + web chat + mobile app.

**Managora application:** Student/parent portal.

**Dependencies:** LLM provider; RAG over student data; WhatsApp / web chat integration; escalation workflow.

**Complexity:** High.

**Priority:** High (differentiator).

### 13.10 AI-powered homework evaluation assistant (Managora recommendation)
**Opportunity:** The reference project's `homework_evaluation` is manual per-student. Managora could add an **AI homework evaluator** that:
- Pre-fills evaluations for objective questions (auto-mark correct/incorrect).
- Suggests feedback comments for subjective answers.
- Flags submissions for teacher review when AI confidence is low.
- Tracks common mistakes across students ("60% of students got Q3 wrong — re-teach this concept").

**Managora application:** Academics module.

**Dependencies:** LLM provider; homework submission model; teacher review workflow.

**Complexity:** Medium.

**Priority:** Medium.

### 13.11 Unified workspace pattern (Managora recommendation)
**Opportunity:** The reference project has separate admin / student / parent panels with different navigations. Managora could build a **unified workspace** that:
- Adapts the navigation to the user's role + tenant type (coaching institute vs. school vs. college).
- Surfaces the most-likely-next-action on the dashboard ("3 fees due today", "2 unread parent messages", "1 timetable conflict to resolve").
- Allows switching between multiple institutes (for franchise owners) without logout.
- Provides a command palette (Cmd+K) for quick navigation / actions.

**Managora application:** All Managora modules.

**Dependencies:** Role-aware nav; tenant-switcher; command palette; AI-powered dashboard suggestions.

**Complexity:** High.

**Priority:** High.

### 13.12 Per-tenant AI credit budget (Managora recommendation)
**Opportunity:** The reference project has no AI. Managora's AI features will consume LLM tokens. Build a **per-tenant AI credit system** that:
- Allocates monthly credits per subscription plan.
- Tracks usage per AI feature (question generation, chatbot, narrative generation).
- Throttles when budget is exhausted (graceful degradation to rule-based fallback).
- Allows top-up purchases.
- Shows usage dashboard to tenant admin.

**Managora application:** SaaS billing.

**Dependencies:** Subscription plans; usage metering; LLM token counting; billing integration.

**Complexity:** Medium.

**Priority:** Critical (foundational for AI-first SaaS).

**Dependencies:** Subscription plan model; per-tenant credit ledger; LLM provider abstraction with token counting; billing integration (Stripe Billing / Chargebee).

### 13.13 Franchise / multi-institute org hierarchy (Managora recommendation)
**Opportunity:** The reference project's Multi Branch is flat (master + branches). Managora could model a **proper org hierarchy** that:
- Supports Organization → Institutes → Branches → Batches.
- Per-level RBAC (org admin sees all institutes; institute admin sees own + branches; branch admin sees own only).
- Per-level billing (org pays for all; or each institute pays own).
- Per-level branding (org logo on all institutes; or each institute brands own).

**Managora application:** Franchise customers.

**Dependencies:** Org hierarchy model; per-level RBAC; per-level billing; per-level branding.

**Complexity:** High.

**Priority:** Medium (depends on franchise customer segment).

### 13.14 Visual workflow builder (Managora recommendation)
**Opportunity:** The reference project's automation is hard-coded in cron. Managora could build a **visual workflow builder** that:
- Lets tenant admin drag-and-drop triggers ("when fee is due in 3 days") + conditions ("if student hasn't paid") + actions ("send WhatsApp + email + create task for accountant").
- Supports AI-assisted workflow suggestions ("I noticed you send fee reminders manually — want me to automate this?").
- Logs workflow executions.

**Managora application:** All modules.

**Dependencies:** Workflow engine; trigger registry; action registry; visual builder UI; AI workflow suggestion.

**Complexity:** High.

**Priority:** Medium.

### 13.15 Outbound webhook system (Managora recommendation)
**Opportunity:** The reference project has no outbound webhooks. Managora could add a **per-tenant webhook subscription system** that:
- Lets tenants subscribe to events ("student admitted", "fee paid", "exam result published").
- Signs payloads with HMAC-SHA256 using a per-subscription secret.
- Retries with exponential backoff on failure.
- Shows delivery history + logs.

**Managora application:** Integration with external CRM / marketing automation / analytics tools.

**Dependencies:** Event registry; webhook subscription model; HTTP client; retry queue; per-tenant config.

**Complexity:** Medium.

**Priority:** Medium.

### 13.16 Public API with OpenAPI spec (Managora recommendation)
**Opportunity:** The reference project has no formal API spec. Managora could publish a **versioned public API** that:
- Has an OpenAPI 3.1 spec.
- Supports OAuth2 + API keys.
- Has rate limiting per tenant.
- Has webhook subscriptions.
- Has SDKs in Python / JavaScript / PHP.

**Managora application:** Third-party integrations; mobile app; custom client integrations.

**Dependencies:** API gateway; OAuth2 server; rate limiter; OpenAPI spec generator; SDK generator.

**Complexity:** High.

**Priority:** Medium.

### 13.17 Mobile-first PWA instead of separate Flutter app (Managora recommendation)
**Opportunity:** The reference project ships a separate Flutter app. Managora could build a **PWA** that:
- Works offline (cached student data, attendance marking).
- Installable on home screen.
- Reuses the same codebase as the web app.
- Supports push notifications via service worker.

**Managora application:** Student / parent / teacher mobile access.

**Dependencies:** PWA framework (Next.js / SvelteKit); service worker; offline data sync; push notification.

**Complexity:** Medium.

**Priority:** Medium.

### 13.18 Per-tenant data residency (Managora recommendation)
**Opportunity:** The reference project's multi-DB is for federation, not residency. Managora could offer **per-tenant data residency** that:
- Lets tenants choose where their data is stored (e.g. India for Indian schools, EU for GDPR).
- Implemented via tenant-resolver middleware that picks DB region per request.
- Credentials in secrets manager.

**Managora application:** Compliance-conscious customers (EU schools, government schools).

**Dependencies:** Multi-region DB deployment; tenant-resolver middleware; secrets manager; per-tenant region config.

**Complexity:** High.

**Priority:** Low (only relevant for specific customer segments).

### 13.19 AI-powered syllabus / lesson-plan generation (Managora recommendation)
**Opportunity:** The reference project's `subject_syllabus` is manual. Managora could add an **AI syllabus generator** that:
- Takes a curriculum standard (CBSE / IB / state board) + subject + grade level.
- Generates a lesson-by-lesson syllabus for the academic year.
- Suggests teaching resources per lesson.
- Auto-creates homework assignments per lesson.

**Managora application:** Academics module.

**Dependencies:** LLM provider; curriculum standard model; syllabus schema; teacher review workflow.

**Complexity:** Medium.

**Priority:** Medium.

### 13.20 AI-powered parent-engagement scoring (Managora recommendation)
**Opportunity:** The reference project doesn't measure parent engagement. Managora could add an **AI engagement scorer** that:
- Tracks parent interactions (opened notification, attended PTM, replied to WhatsApp, paid fees on time).
- Scores engagement per student.
- Suggests outreach to disengaged parents.
- Correlates engagement with student performance.

**Managora application:** Parent communication module.

**Dependencies:** Engagement event logging; ML model; outreach workflow.

**Complexity:** Medium.

**Priority:** Low.

---

## 14. Prioritized Implementation Candidates

> Sorted by Priority (Critical > High > Medium > Low), then by Complexity (Low > Medium > High) within each priority.

| # | Concept | Managora Application | Expected Benefit | Complexity | Priority | Classification |
|---|---|---|---|---|---|---|
| 1 | Per-event × per-channel × per-recipient notification matrix | All communication | Decouples triggers from channels; scales to new channels (incl. AI) | Medium | Critical | Adopt |
| 2 | Per-year enrollment (`student_session`) as scope unit | All operational data | Clean academic-year model; preserves history | Low | Critical | Adopt |
| 3 | Per-tenant AI credit budget | SaaS billing | Foundational for AI-first SaaS monetisation | Medium | Critical | Adopt (new) |
| 4 | Forward-compatible dispatcher + provider plugins | SMS/Email/Push/WhatsApp/AI/Payments/Storage/Live-class | New providers added as plugin classes; per-tenant config | Medium | High | Adopt |
| 5 | Module-toggle + sub-feature permission matrix (as code) | RBAC for all modules | Plugins declare permissions declaratively; uniform enforcement | Medium | High | Adopt |
| 6 | Tokenised shareable PDF links (signed JWT) | Fee receipts / marksheets / certificates / reports | Time-windowed, intent-bound, auditable sharing | Low | High | Adapt |
| 7 | TOTP 2FA with per-user secrets (encrypted at rest) + WebAuthn | Staff + tenant admin 2FA | AAL2 compliance; modern auth | Low | High | Adapt |
| 8 | Audit log with structured event logging | All CUD operations | Compliance (FERPA / GDPR); AI anomaly detection | Low | High | Adopt |
| 9 | Auto-numbering with per-tenant sequence | Admission nos / staff IDs / receipt nos / invoice nos / TC nos | Per-tenant format; atomic allocation | Low | High | Adopt |
| 10 | Hierarchical exam model with pluggable grading schemes | Examination module | Models real exam hierarchies; supports multiple grading systems | High | High | Adapt |
| 11 | AI-first communication (AI channel + LLM-drafted messages) | All parent communication | Differentiator; personalisation; multi-language | High | High | Adopt (new) |
| 12 | AI-powered chatbot for student/parent queries | Student/parent portal | Differentiator; reduces staff load | High | High | Adopt (new) |
| 13 | Unified workspace with role-aware nav + command palette | All Managora modules | Better UX; faster workflows | High | High | Adopt (new) |
| 14 | Co-scholastic observation framework (generalised Rubric) | Behaviour / soft-skills tracking | Models any multi-criteria per-student assessment | Medium | Medium | Adapt |
| 15 | LMS-style course structure with paid checkout | Course-seller module | MOOC-in-a-box for coaching institutes / instructors | High | Medium | Adapt |
| 16 | Per-teacher OAuth for third-party services | Zoom / Google Meet / Google Calendar | Per-user connections; per-tenant fallback | Medium | Medium | Adapt |
| 17 | QR-code self-scan attendance (signed JWT + PWA) | Attendance | Cheap hardware; offline-capable | Medium | Medium | Adapt |
| 18 | Cross-tenant aggregated reports (data warehouse) | Franchise / multi-institute dashboard | Consolidated view without multi-DB iteration | High | Medium | Adapt |
| 19 | Field-visibility toggles + custom-field builder | Student / staff / enquiry forms | Per-tenant form customisation without code | Medium | Medium | Adapt |
| 20 | Per-event × per-channel WhatsApp template integration | WhatsApp notifications | Per-event template binding; parameter substitution | Medium | Medium | Adopt |
| 21 | Theme system with live toggle (per-tenant + per-user) | Branding + UX preference | Per-tenant branding; per-user dark/light | Low | Medium | Adopt |
| 22 | ID-card with signed JWT QR (reused for attendance / library / fee confirmation) | ID cards + attendance + library + fee counter | Multi-purpose QR; secure (signed) | Medium | Medium | Adopt |
| 23 | Multi-channel login (username / mobile / email / admission_no) | Student/parent login | Per-tenant login-identifier config | Low | Medium | Adapt |
| 24 | AI-powered question generation | Examination + Online Course | Differentiator; saves teacher time | Medium | Medium | Adopt (new) |
| 25 | AI-powered descriptive-answer scoring | Online Course exams | Reduces teacher grading load | Medium | Medium | Adopt (new) |
| 26 | AI-powered attendance anomaly detection | Attendance module | Proactive intervention; reduces dropout | Medium | Medium | Adopt (new) |
| 27 | AI-powered fee-defaulter prediction | Fees module | Proactive reminders; improves cash flow | Medium | Medium | Adopt (new) |
| 28 | AI-powered admission enquiry scoring | Front Office / Admissions | Improves conversion; prioritises counsellor time | Medium | Medium | Adopt (new) |
| 29 | AI-powered marksheet narrative generation | Examination module | Better parent communication; actionable insights | Low | Medium | Adopt (new) |
| 30 | AI-powered homework evaluation assistant | Academics module | Reduces teacher load; common-mistake tracking | Medium | Medium | Adopt (new) |
| 31 | AI-powered syllabus / lesson-plan generation | Academics module | Saves teacher time; standards-aligned | Medium | Medium | Adopt (new) |
| 32 | Visual workflow builder | All modules | Tenant-admin self-service automation | High | Medium | Adopt (new) |
| 33 | Outbound webhook system | External integrations | Tenant integrations with CRM / marketing / analytics | Medium | Medium | Adopt (new) |
| 34 | Public API with OpenAPI spec | Third-party integrations / mobile / custom clients | Ecosystem; integration revenue | High | Medium | Adopt (new) |
| 35 | Mobile-first PWA instead of separate Flutter app | Student / parent / teacher mobile | Single codebase; offline-capable; installable | Medium | Medium | Adopt (new) |
| 36 | Franchise / multi-institute org hierarchy | Franchise customers | Per-level RBAC + billing + branding | High | Medium | Adopt (new) |
| 37 | AI-powered timetable generation | Academics module | Saves admin time; optimal schedules | High | Medium | Adopt (new) |
| 38 | Cron + secret-key gating (only for self-hosted) | Self-hosted deployments (if any) | Secures scheduled jobs in self-hosted context | Low | Low | Adapt |
| 39 | Per-tenant data residency | Compliance-conscious customers | Data-sovereignty compliance | High | Low | Adopt (new) |
| 40 | AI-powered parent-engagement scoring | Parent communication | Identifies disengaged parents; correlation with performance | Medium | Low | Adopt (new) |

---

## 15. Evidence Limitations and Unverified Findings

### Verified in source
All findings about code structure, schema, controller methods, library functions, table columns, and integration points are **verified in source** by direct file inspection during this analysis. Key evidence:

| Claim | Evidence |
|---|---|
| 196 P1 tables | `controllers/install/database.sql` |
| 77 P2 tables | sum of per-addon `grep -c "^CREATE TABLE" installer/install/db/db.sql` |
| Plain-text student/parent passwords | `application/models/User_model.php` lines 93, 111, 137, 224 |
| Staff passwords hashed (bcrypt) | `application/libraries/Enc_lib.php::passHashEnc` + `models/Staff_model.php::checkLogin` |
| Hardcoded AES keys | `application/libraries/Enc_lib.php` lines 3-4 (`ss@pubkey` / `ss@pvtkey`) |
| Hardcoded Zoom token | `application/controllers/App.php::zoom()` |
| SaaS disabled | `application/config/saas-config.php` + `application/libraries/SaasValidation.php` |
| License = Envato + URL-bound AES | `application/core/MY_Controller.php::check_license()` + `application/libraries/Auth.php::addonchk()` |
| 28 payment gateways | `application/controllers/onlineadmission/` directory listing |
| 11 SMS providers | `application/libraries/Smsgateway.php::sendSMS()` switch chain |
| 78 languages | `application/language/` directory listing |
| Cron jobs | `application/controllers/Cron.php` |
| FCM push | `application/libraries/Pushnotification.php` |
| Audit log | `application/core/MY_Model.php::log()` |
| Addon installer runs `multi_query()` | `application/helpers/addon_helper.php::execute_sql()` |
| No AI code | grep returned 0 hits across both projects |
| 22 `module_lib->hasModule/hasActive` checks in P1 | `grep -rn "module_lib->hasModule\|module_lib->hasActive" controllers/` |
| 20 `whatsappgateway->` calls in P1's Mailsmsconf | `grep -n "whatsappgateway->" libraries/Mailsmsconf.php` |
| Leaky addon boundary: `Welcome::cbseexam()` loads CBSE classes without `hasActive` guard | `application/controllers/Welcome.php:1551` |
| Multi Branch plain-text DB credentials | `<MB>/installer/install/db/db.sql` `multi_branch` table |
| TOTP implementation | `<TFA>/application/libraries/Google_authenticator.php` + `<TFA>/application/controllers/Gauthenticate.php` |
| Zoom OAuth flow | `<ZOOM>/application/libraries/Zoom_api.php` (Guzzle + `/oauth/token`) |
| GMeet Google Calendar API | `<GMEET>/application/libraries/Google.php` |
| WhatsApp providers (Meta + Twilio) | `<WA>/application/libraries/Whatsappgateway.php` |
| CBSE 28-table schema | `<CBSE>/installer/install/db/db.sql` |
| Online Course 29-table schema + S3 | `<OC>/installer/install/db/db.sql` + `<OC>/application/libraries/Aws3.php` |
| Quick Fees 0 new tables | `<QF>/installer/install/db/db.sql` |
| WhatsApp `product_id=999999999` mismatch | `<WA>/addon_info.json` vs P1's `addons` seed (`61574315`) |
| GMeet hardcoded `location='Jabalpur'` | `<GMEET>/application/libraries/Google.php::createEvent()` |
| Thermal Print seed has real customer data | `<TP>/installer/install/db/db.sql` seed |
| Quick Fees uses raw `$_POST` | `<QF>/application/controllers/admin/Customfeesmaster.php::createInstallment()` |

### Documented by vendor
- Each addon ships a `documentation/index.html` user manual — title-only verified (e.g. "Smart School CBSE Examination - Documentation"). Contents not deeply analysed.
- Addon descriptions in `addons` seed table (product_id, name, description, article_link to `https://go.smart-school.in/{addon}`).

### Observed in interface
- **None.** No live demo was available. All UI/UX findings are inferred from view files (`application/views/admin/dashboard.php`, `application/views/layout/sidebar.php`, etc.), not from runtime observation.

### Inferred
- The exact mechanism by which `installer/install/files/application/*` lands in the host's `application/` directory was inferred from the addon contract (files match the host's directory structure) but not verified end-to-end (the file-copier script was not deeply inspected).
- Runtime behaviour of `Auth::addonchk()` when the license server returns malformed data is inferred from the code path but not verified by execution.
- The "redirect loop" prediction for uninstalled addon controllers is inferred from the code path (`Auth::addonchk` redirects to setting page which doesn't exist) but not verified by execution.

### Not verified
- **Vendor's licensing server** — not reachable from analysis environment. The exact response of the purchase-code registration endpoint was inferred from `Auth::addonchk()` and `Auth::addon_update_check()` but not observed.
- **Mobile app source** (Flutter) — not in either package. The PHP backend's API endpoints are visible, but how the mobile app consumes them is not.
- **Project 1's `documentation/` folder** — referenced in `readme.txt` ("documentation: Quick Installation Guide & User's Manual") but not present in the RAR archive.
- **Project 1's `smart_school_update_7.1.0_to_7.2.0/` delta package** — not deeply analysed.
- **10 of 11 addon `*_update_X.0_to_Y.0/` packages** — delta SQL only, not deeply analysed.
- **Whether the vendor's hosted distribution hashes passwords differently** from the open-source distribution analysed here. Cannot answer without inspecting the hosted version.
- **Whether addon installer SQL files are audited before publication.** The Thermal Print seed contains real customer data ("Mount/ Carmel /School" with a real-looking phone number), suggesting no sanitisation review — but this is an inference, not a verified process.
- **Does the Multi Branch license check pass when switched to a branch?** Not verified end-to-end. The addon's `Auth::addonchk()` uses master's `license.php` (file, not DB row), so it should pass — but the branch's `addons` table might show `current_stage=0`, confusing the in-app addon manager.
- **Does the WhatsApp `product_id=999999999` mismatch cause real installation failures?** Cannot verify without running the installer.
- **Are the 10 `*_update_X.0_to_Y.0/` packages safe to apply on top of an existing install?** Not analysed in detail.
- **Performance at scale** — no benchmarks were run. All scalability concerns are inferred from schema design (e.g. TEXT-stored JSON, multi-DB iteration) rather than measured.

### Limitations of this analysis
1. **Static analysis only** — no code execution, no live demo, no DB access.
2. **No vendor documentation** beyond `readme.txt` and per-addon HTML user manual titles.
3. **No mobile app source** — only the PHP backend.
4. **No vendor licensing server** access.
5. **Some large files sampled** rather than read line-by-line: `Customlib.php` (2493 lines), `Smsgateway.php` (3145 lines), `Mailsmsconf.php` (78 KB), CBSE `Exam.php` (1600+ lines), CBSE `Result.php` (2400+ lines).

---

## 16. Final Assessment

### What should we learn from this project?

The reference project is a **comprehensive but aging K-12 ERP** with an active addon ecosystem. Its core value is **breadth** — most operational needs of a school or coaching institute are covered, configurable without code, and translatable into 78 languages. Its core weakness is **technical debt and security** — plain-text passwords, hardcoded crypto keys, multi-database multi-tenancy with plain-text credentials, `multi_query()` installer with no rollback, CodeIgniter 3 EOL, no AI capabilities anywhere.

The most important lessons for Managora:

1. **The notification matrix is the right abstraction** — adopt it from day one, with AI as a first-class channel.
2. **`student_session` per-year enrollment is the right scope** — adopt it for all operational data.
3. **The hierarchical exam model is necessary** — adapt it (cleaner schema, multiple grading schemes, AI-powered scoring).
4. **The forward-compatible dispatcher pattern is right** — adopt it for all channels (SMS/Email/Push/WhatsApp/AI/Payments/Storage/Live-class).
5. **Multi-tenancy must be single-DB with RLS** — avoid multi-database federation with plain-text credentials.
6. **Plugin/module system must use proper migrations** — avoid `multi_query()` SQL installer.
7. **All credentials must be in a secrets manager** — avoid plain-text storage.
8. **AI is the differentiator** — the reference project has zero AI; Managora should be AI-first in communication, assessment, attendance, fees, admissions, and parent engagement.
9. **Per-tenant AI credit budget is foundational** — build it from day one.
10. **Avoid fat controllers** — service layer + thin controllers + DTOs.

### Which concepts could improve Managora?

The 40 prioritised concepts in §14 cover the full spectrum. The top 5 critical/high-priority concepts that would most improve Managora:

1. **Notification matrix with AI channel** (Critical, Medium complexity) — Adopt.
2. **Per-year enrollment scoping** (Critical, Low complexity) — Adopt.
3. **Per-tenant AI credit budget** (Critical, Medium complexity) — Adopt (new).
4. **Forward-compatible dispatcher + provider plugins** (High, Medium complexity) — Adopt.
5. **Module-toggle + sub-feature permission matrix as code** (High, Medium complexity) — Adopt.

### How should we adapt them?

- **Use modern stack** (Laravel / Next.js / NestJS / Django) — not CodeIgniter 3.
- **Use signed JWTs** for licensing and document sharing — not reversible AES.
- **Use proper migration framework** (Phinx / Liquibase / Prisma) — not `multi_query()`.
- **Use job queue** (Sidekiq / Celery / SQS) — not cron.
- **Use single-DB multi-tenancy with RLS** — not multi-database.
- **Use secrets manager** (AWS Secrets Manager / Vault) — not plain-text DB columns.
- **Use LLM providers** (OpenAI / Anthropic / local) — not rule-based only.
- **Use OpenAPI spec** for public API — not ad-hoc JSON endpoints.
- **Use PWA** for mobile — not separate Flutter app.

### What should we avoid?

1. Plain-text password storage (any user type).
2. Multi-database multi-tenancy with plain-text credentials.
3. `multi_query()` SQL installer with no rollback.
4. Hardcoded AES keys / hardcoded API tokens in source.
5. SMS/WhatsApp gateway failures silently swallowed.
6. Leaky module boundary (core directly references module classes).
7. URL-bound AES licensing.
8. Fat-controller pattern.
9. TEXT-stored JSON for transactional data.
10. utf8mb3 collation.
11. Code duplication across providers / gateways.
12. Cron as a single HTTP hit running multiple jobs.
13. Inconsistent `is_active` typing.
14. `created_at ON UPDATE CURRENT_TIMESTAMP` quirk.
15. No outbound webhook system.
16. Hardcoded third-party credentials in source.
17. Vendor lock-in to a single licensing server.
18. No test suite / no CI/CD.

### The central question, answered

**What should we learn from this project, which concepts could improve Managora, how should we adapt them, and what should we avoid?**

Learn: the notification matrix, per-year enrollment, hierarchical exams, forward-compatible dispatcher, module-toggle RBAC, tokenised sharing, CBSE observations, LMS structure, per-teacher OAuth, TOTP 2FA.

Improve Managora by: adopting the above with a modern stack (signed JWTs, job queue, single-DB RLS, secrets manager, LLM providers, OpenAPI, PWA); adding AI as a first-class citizen in communication, assessment, attendance, fees, admissions, parent engagement; building per-tenant AI credit budget from day one; building a unified workspace with role-aware nav + command palette; building a visual workflow builder; building outbound webhooks + public API.

Adapt by: using modern stack throughout; encrypting all credentials at rest; persisting OAuth tokens in DB; using signed JWTs for licensing and sharing; using proper migrations; using queue instead of cron; using single-DB multi-tenancy; using PWA instead of separate app; generalising CBSE observations to a Rubric system; abstracting providers behind interfaces.

Avoid: plain-text passwords; multi-database multi-tenancy; `multi_query()` installer; hardcoded AES keys; silent SMS failures; leaky module boundary; URL-bound AES licensing; fat controllers; TEXT-stored JSON for transactional data; utf8mb3; code duplication; cron-as-HTTP; inconsistent typing; `created_at ON UPDATE` quirk; no outbound webhooks; hardcoded credentials; vendor lock-in; no tests / no CI.

### Overall analysis confidence
- **High** for static code analysis of both packages and their intersection.
- **Moderate** for runtime behaviour (no execution, no live demo).
- **Low** for the vendor's licensing server behaviour and the mobile app's API consumption patterns.

The critical findings (plain-text passwords, plain-text DB credentials, `multi_query()` installer, no AI, no true multi-tenancy, leaky addon boundary) are **certain** — they are directly visible in source.

---

*End of Reference Project Analysis for Managora.*
