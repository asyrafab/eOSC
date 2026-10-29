# eOSC+ Plan 1: Foundation and Agenda Builder — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Staff can create OSC meetings, departments put applications (linked or standalone) forward with papers, and the secretary orders and publishes a versioned agenda that members read on their phones.

**Architecture:** eOSC+ is core capability C10 inside the iNDECS Laravel app. Tables live in the PostgreSQL `core` schema with prefix `osc_`; services own every write (row-version checks, append-only `audit_logs`); thin controllers under the `/osc` route prefix render Blade views in the existing workspace shell.

**Tech Stack:** PHP 8.3, Laravel (existing app), PostgreSQL 18 + PostGIS (`indecs_test` for tests), PHPUnit feature tests, Blade with `public/css/indecs.css`, Laravel mail notifications, existing `DocumentScanner` contract (ClamAV).

**Spec:** `docs/eosc/README.md` (design v1.1, approved 2026-10-29). Read sections 3–8.1 before starting.

## Global Constraints

- Follow `AGENTS.md` and `docs/DESIGN-DIRECTION.md` exactly: tokens only (no hard-coded colour, font, spacing), Lucide icons via `<x-icon>`, Bahasa Melayu interface, reuse existing `indecs.css` classes, no Tailwind.
- Tables: `core.osc_*`. Routes: prefix `/osc`, names `osc.*`. Permissions: `osc.*`. Recorded in ADR-314 (Task 1).
- Product name shown to users: **eOSC+**.
- Every state change writes an `AuditLog` row (`auditable_type`, `auditable_id`, `event`, `actor_id`, `before_values`, `after_values`, `ip_address`). Never update or delete audit rows.
- Every mutable record carries `row_version`; writes take `int $expectedVersion` and fail with the message `Rekod telah berubah. Muat semula sebelum meneruskan.` on mismatch (same text as `WorkflowService`).
- One agenda item = one application (design section 5). An application may be tabled many times, at different meetings or the same meeting.
- Tabling purposes: `permohonan` (Permohonan), `rayuan` (Rayuan), `pertimbangan_semula` (Pertimbangan semula), `pindaan` (Pindaan), `lain_lain` (Lain-lain).
- Staff accounts only in this plan; no OTP, no public access.
- Tests run against PostgreSQL (`php artisan test`), never SQLite. `php vendor/bin/pint --test` must pass.
- Conventional Commits.

## Review Focus

1. **Two people editing the same agenda at once** (secretary and STENO reorder or publish together): the second write must be refused with the row-version message, never silently overwrite. Test in Task 4.
2. **An officer proposing an application outside their department or district scope**: must get 403 and create nothing. Test in Task 3.
3. **A paper that fails the virus scan or the scanner is down**: upload refused, no paper row, no file left on disk. Test in Task 5.
4. **A proposal arriving after the proposal cut-off**: an officer is refused with `Tarikh tutup cadangan agenda telah berlalu.`; the secretary can still add it. Test in Task 3.
5. **A staff user who is not a member of the meeting opens its agenda or a paper**: 403, and draft agendas are never shown to non-managers. Test in Task 6.

---

## File structure

| File | Responsibility |
|---|---|
| `docs/adr/ADR-314-eosc-naming-and-c10-schema.md` | Naming decision for routes, tables, permissions |
| `database/migrations/2026_10_29_000000_create_osc_meetings_and_agenda.php` | All `core.osc_*` tables for this plan |
| `database/seeders/OscSeeder.php` | Roles, permissions, grants for eOSC+ (called from `DatabaseSeeder`) |
| `app/Enums/Osc/MeetingStatus.php`, `MeetingRole.php`, `AgendaItemStatus.php`, `TablingPurpose.php` | Enums with BM `label()` |
| `app/Models/Osc/Meeting.php`, `MeetingMember.php`, `AgendaItem.php`, `AgendaItemPaper.php`, `AgendaVersion.php`, `DepartmentSetting.php` | Eloquent models |
| `app/Services/Osc/MeetingService.php` | Create meetings, manage members |
| `app/Services/Osc/AgendaService.php` | Propose, endorse, accept, return, reorder, publish |
| `app/Services/Osc/AgendaPaperService.php` | Scanned, hashed, versioned paper upload and download |
| `app/Services/Osc/OscAccess.php` | Every "may this user see/do this" rule for eOSC+ |
| `app/Notifications/Osc/AgendaPublished.php` | Mail to meeting members on publish |
| `app/Http/Middleware/EnsureOscAccess.php` | `osc.access` alias |
| `app/Http/Controllers/Osc/MeetingController.php`, `AgendaItemController.php`, `AgendaPaperController.php` | HTTP layer |
| `app/Http/Requests/Osc/*.php` | Validation |
| `resources/views/osc/*.blade.php` | Screens |
| `tests/Feature/Osc/*.php` | Feature tests |

---

### Task 1: Naming ADR, schema, enums, models, permissions

**Files:**
- Create: `docs/adr/ADR-314-eosc-naming-and-c10-schema.md`
- Modify: `docs/DESIGN-DIRECTION.md` (section 1 table: add a row), `docs/adr/README.md` (list ADR-314)
- Create: `database/migrations/2026_10_29_000000_create_osc_meetings_and_agenda.php`
- Create: `database/seeders/OscSeeder.php`; Modify: `database/seeders/DatabaseSeeder.php` (call it last)
- Create: the four enums and six models listed above
- Modify: `app/Models/User.php` (add `canAccessOsc()`, `canPerformOsc()`)
- Test: `tests/Feature/Osc/OscSchemaTest.php`

**Interfaces:**
- Produces:
  - Enums (string-backed, each with `label(): string`):
    - `MeetingStatus`: `Draft='draft'` (Draf), `AgendaPublished='agenda_published'` (Agenda diterbitkan), `InProgress='in_progress'` (Sedang berlangsung), `Closed='closed'` (Selesai), `MinutesConfirmed='minutes_confirmed'` (Minit disahkan)
    - `MeetingRole`: `Chair='chair'` (Pengerusi), `Secretary='secretary'` (Setiausaha), `Steno='steno'` (STENO), `Member='member'` (Ahli)
    - `AgendaItemStatus`: `Proposed='proposed'` (Dicadangkan), `Endorsed='endorsed'` (Disokong ketua jabatan), `Accepted='accepted'` (Dalam agenda), `Returned='returned'` (Dikembalikan)
    - `TablingPurpose`: values in Global Constraints
  - `User::canPerformOsc(string $permission, ?string $departmentId = null): bool` (active staff + `CorePermissionService::userCan($this, $permission, $departmentId ?? $this->department_id, null)`)
  - `User::canAccessOsc(): bool` (`canPerformOsc('osc.meeting.view')`)
  - Models in namespace `App\Models\Osc`, ULID keys where the table uses ULID, relations: `Meeting::members()`, `Meeting::items()` (ordered by `position` nulls last, then `created_at`), `Meeting::versions()`, `AgendaItem::meeting()`, `AgendaItem::application()`, `AgendaItem::papers()` (ordered by `version` desc), `AgendaItem::previousItem()`, `AgendaItem::presenter()`

- [ ] **Step 1: Write ADR-314** in the ADR template from `docs/adr/README.md`. Decision: eOSC+ is core C10; tables `core.osc_*`; routes `/osc`, names `osc.*`; permissions `osc.meeting.view`, `osc.meeting.manage`, `osc.agenda.propose`, `osc.agenda.endorse`; roles `osc_secretary` (Setiausaha OSC), `osc_steno` (STENO OSC); product name eOSC+. Add the row to `DESIGN-DIRECTION.md` section 1: `| eOSC+ | eOSC+ (Mesyuarat OSC) | eOSC+ | /osc | core.osc_ | — | Bil. 21/2026 |`. Add ADR-314 to the list in `docs/adr/README.md`.

- [ ] **Step 2: Write the failing test** `tests/Feature/Osc/OscSchemaTest.php`

```php
public function test_osc_tables_permissions_and_roles_exist(): void
{
    $this->seed();
    foreach (['osc_meetings', 'osc_meeting_members', 'osc_agenda_items', 'osc_agenda_item_papers', 'osc_agenda_versions', 'osc_department_settings'] as $table) {
        $this->assertTrue(Schema::hasTable('core.'.$table), $table);
    }
    foreach (['osc.meeting.view', 'osc.meeting.manage', 'osc.agenda.propose', 'osc.agenda.endorse'] as $permission) {
        $this->assertDatabaseHas('core.permissions', ['name' => $permission]);
    }
    $secretary = User::factory()->create(['role' => 'osc_secretary', 'department_id' => 'JPBD']);
    $this->assertTrue($secretary->canPerformOsc('osc.meeting.manage'));
    $officer = User::factory()->create(['role' => 'building_officer', 'department_id' => 'JBANG']);
    $this->assertTrue($officer->canPerformOsc('osc.agenda.propose'));
    $this->assertFalse($officer->canPerformOsc('osc.meeting.manage'));
    $this->assertSame('Pertimbangan semula', TablingPurpose::PertimbanganSemula->label());
}
```

- [ ] **Step 3: Run it to verify it fails**

Run: `php artisan test --filter=OscSchemaTest`
Expected: FAIL (table `core.osc_meetings` missing).

- [ ] **Step 4: Write the migration.** Columns (all tables `timestamps()` unless noted):
  - `core.osc_meetings`: `ulid id` PK; `meeting_type string(30)` default `jawatankuasa_osc`; `number string(30)`; `scheduled_at timestampTz`; `venue string`; `proposal_cutoff_at timestampTz`; `paper_cutoff_at timestampTz`; `status string(30)` default `draft`; `agenda_version unsignedInteger` default 0; `row_version unsignedInteger` default 1; `created_by` FK users restrict. Unique `(meeting_type, number)`.
  - `core.osc_meeting_members`: `id`; `meeting_id` ULID FK cascade; `user_id` FK users restrict; `role string(20)`. Unique `(meeting_id, user_id, role)`. Partial unique index: one `chair` per meeting.
  - `core.osc_agenda_items`: `ulid id` PK; `meeting_id` ULID FK restrict; `application_id` ULID nullable FK `applications` restrict; `department_id string(20)` FK departments; `application_type_id string(20)` nullable FK application_types; `standalone_type string(100)` nullable (e.g. `Penamaan Jalan & Taman`); `title string(1000)`; `reference string` nullable; `purpose string(30)`; `presenter_id` FK users restrict; `proposed_by` FK users restrict; `status string(20)` default `proposed`; `position unsignedInteger` nullable; `return_reason text` nullable; `previous_item_id` ULID nullable self-FK; `row_version` default 1. Check constraint: `application_id IS NOT NULL OR standalone_type IS NOT NULL`.
  - `core.osc_agenda_item_papers`: `ulid id` PK; `agenda_item_id` ULID FK restrict; `version unsignedInteger`; `original_name`; `path`; `mime_type string(100)`; `size_bytes unsignedBigInteger`; `sha256 char(64)`; `uploaded_by` FK users restrict; `created_at` only. Unique `(agenda_item_id, version)`.
  - `core.osc_agenda_versions`: `id`; `meeting_id` ULID FK restrict; `version unsignedInteger`; `items jsonb` (ordered list of `{id, title, department_id, purpose}`); `changes jsonb` (`{added:[ids], removed:[ids], reordered:bool}`); `published_by` FK users restrict; `published_at timestampTz`. Unique `(meeting_id, version)`.
  - `core.osc_department_settings`: `department_id string(20)` PK FK departments; `requires_endorsement boolean` default false; `updated_by` FK users nullable; timestamps.
  - `down()`: refuse with `RuntimeException` if any `core.osc_meetings` row exists; otherwise drop in reverse order (same guard pattern as `2026_10_28_000000_create_core_document_verification.php`).

- [ ] **Step 5: Write `OscSeeder`** using the `updateOrInsert` pattern of `DatabaseSeeder::seedCorePermissions()`. Roles `osc_secretary`, `osc_steno`. Permissions with BM labels:
  - `osc.meeting.view`: Akses eOSC+
  - `osc.meeting.manage`: Urus mesyuarat dan agenda OSC
  - `osc.agenda.propose`: Cadang perkara ke agenda OSC
  - `osc.agenda.endorse`: Sokong cadangan agenda jabatan

  Grants (`application_type_id` null):
  - `osc_secretary` and `osc_steno`: view and manage, department null.
  - `planning_officer` and `senior_planning_officer`: view and propose, department `JPBD`.
  - `building_officer` and `senior_building_officer`: view and propose, department `JBANG`.
  - `senior_planning_officer`: endorse, department `JPBD`.
  - `senior_building_officer`: endorse, department `JBANG`.
  - `system_admin`: all four, department null.

  Call it at the end of `DatabaseSeeder::run()`.

- [ ] **Step 6: Write enums, models, and the two `User` methods** per Interfaces. Models: `$table = 'core.osc_…'`, `HasUlids` where the key is ULID, enum casts for `status`, `role`, `purpose`; `AgendaItemPaper::UPDATED_AT = null`.

- [ ] **Step 7: Run the test**

Run: `php artisan test --filter=OscSchemaTest`
Expected: PASS. Then run `php artisan migrate:rollback --step=1 --pretend` to confirm the down path compiles.

- [ ] **Step 8: Commit**

```bash
git add docs/adr/ADR-314-eosc-naming-and-c10-schema.md docs/adr/README.md docs/DESIGN-DIRECTION.md database/ app/Enums/Osc app/Models/Osc app/Models/User.php tests/Feature/Osc/OscSchemaTest.php
git commit -m "feat(osc): add eOSC+ meeting and agenda schema with permissions (ADR-314)"
```

---

### Task 2: Meetings and members

**Files:**
- Create: `app/Services/Osc/MeetingService.php`, `app/Services/Osc/OscAccess.php`
- Test: `tests/Feature/Osc/MeetingServiceTest.php`

**Interfaces:**
- Consumes: Task 1 models, enums, `User::canPerformOsc()`.
- Produces:
  - `MeetingService::create(array $data, User $actor): Meeting`. `$data` keys: `number`, `scheduled_at`, `venue`, `proposal_cutoff_at`, `paper_cutoff_at`, `chair_id`, `secretary_id`, `steno_id` (nullable), `member_ids` (int[]). Creates the meeting with status `draft`, adds members with their roles, writes audit event `osc_meeting_created`.
  - `MeetingService::setMembers(Meeting $meeting, array $data, int $expectedVersion, User $actor): Meeting`, with the same member keys as `create`. Replaces members, increments `row_version`, writes audit `osc_meeting_members_changed` with before and after lists. Refused when status is `closed` or `minutes_confirmed`.
  - `OscAccess` is a `final` class of **static** methods (called as `OscAccess::canManage($actor)` throughout):
  - `OscAccess::canManage(User $u): bool` (`$u->canPerformOsc('osc.meeting.manage')`)
  - `OscAccess::isMember(User $u, Meeting $m): bool` (any role in `osc_meeting_members`)
  - `OscAccess::canViewMeeting(User $u, Meeting $m): bool`: manager, or member when status is not draft, or presenter of any item in the meeting.

- [ ] **Step 1: Write the failing tests**

```php
public function test_secretary_creates_meeting_with_chair_and_members(): void
// assert status Draft, number 'Bil. 21/2026', one Chair member = chair_id, members count = 1 chair + 1 secretary + 1 steno + 2 members,
// audit_logs has event 'osc_meeting_created' with actor = secretary

public function test_only_one_chair_per_meeting_and_chair_is_any_user(): void
// chair_id may be a user with role 'osc_secretary' (Setiausaha Bandaraya case); setMembers with a new chair replaces the old one

public function test_officer_without_manage_permission_cannot_create(): void
// building_officer → AuthorizationException (abort 403); assertDatabaseCount('core.osc_meetings', 0)

public function test_set_members_with_stale_version_is_refused(): void
// call setMembers with expectedVersion 0 → ValidationException message 'Rekod telah berubah. Muat semula sebelum meneruskan.'

public function test_member_cannot_view_draft_but_can_view_published(): void
// OscAccess::canViewMeeting false while Draft; set status AgendaPublished → true; non-member staff → false
```

- [ ] **Step 2: Run them to verify they fail**

Run: `php artisan test --filter=MeetingServiceTest`
Expected: FAIL (class `MeetingService` not found).

- [ ] **Step 3: Implement `MeetingService` and `OscAccess`.** Each write runs in `DB::transaction` with `lockForUpdate` on the meeting, mirroring `WorkflowService::transition()`. Authorisation uses `abort_unless(OscAccess::canManage($actor), 403)`.

- [ ] **Step 4: Run tests**

Run: `php artisan test --filter=MeetingServiceTest`
Expected: PASS.

- [ ] **Step 5: Commit** `feat(osc): create meetings and manage members`

---

### Task 3: Putting items forward (linked, standalone, repeated) and endorsement

**Files:**
- Create: `app/Services/Osc/AgendaService.php`
- Test: `tests/Feature/Osc/AgendaProposalTest.php`

**Interfaces:**
- Consumes: Task 2 `OscAccess`; `User::canAccessApplication(Application)`; `Application` model.
- Produces:
  - `AgendaService::proposeLinked(Meeting $meeting, Application $application, TablingPurpose $purpose, User $presenter, User $actor): AgendaItem`. Title and reference are copied from the application (`title`, `application_reference`); `department_id` comes from `responsible_department_id` and `application_type_id` from the application. `previous_item_id` is set to the most recent earlier item for the same `application_id`, or null. Audit event `osc_item_proposed`.
  - `AgendaService::proposeStandalone(Meeting $meeting, array $data, User $actor): AgendaItem`. `$data` keys: `standalone_type`, `title`, `reference` (nullable), `purpose`, `presenter_id`. `department_id` is the actor's department.
  - `AgendaService::endorse(AgendaItem $item, int $expectedVersion, User $actor): AgendaItem`. Moves `proposed` to `endorsed`. Requires `osc.agenda.endorse` for the item's department. Audit `osc_item_endorsed`.
  - `AgendaService::requiresEndorsement(string $departmentId): bool` reads `core.osc_department_settings`; a missing row means false.

**Rules (each needs a test):**
- Proposing requires `osc.agenda.propose` scoped to the item's department, otherwise 403.
- A linked item also requires `$actor->canAccessApplication($application)`, otherwise 403. The district scope is respected.
- After `proposal_cutoff_at`, a non-manager is refused with the ValidationException message `Tarikh tutup cadangan agenda telah berlalu.` A manager may still propose.
- Proposing to a meeting in status `in_progress`, `closed` or `minutes_confirmed` is refused with `Mesyuarat ini tidak lagi menerima cadangan.`
- The same application may be proposed twice, including to the same meeting. The second item's `previous_item_id` is the first item's id.
- The presenter must be an active staff user, otherwise ValidationException on `presenter_id`.

- [ ] **Step 1: Write the failing tests**

```php
public function test_building_officer_tables_own_application_linked(): void
// item status Proposed, title/reference copied, department JBANG, purpose Permohonan, audit 'osc_item_proposed'

public function test_officer_cannot_table_application_outside_scope(): void
// JPBD planning_officer proposes a JBANG application → 403; JBANG officer with district 'SPU' proposes SPS application → 403; no rows

public function test_standalone_item_for_type_without_module(): void
// planning_officer proposes standalone 'Penamaan Jalan & Taman' → application_id null, department JPBD

public function test_repeat_tabling_links_previous_item(): void
// propose Permohonan, then Rayuan for the same application in the same meeting → 2 items, second->previous_item_id = first->id

public function test_cutoff_blocks_officer_but_not_secretary(): void
// travel past proposal_cutoff_at; officer → message 'Tarikh tutup cadangan agenda telah berlalu.'; secretary → succeeds

public function test_closed_meeting_refuses_proposals(): void

public function test_endorsement_required_when_department_setting_on(): void
// set requires_endorsement true for JBANG; building_officer cannot endorse (403); senior_building_officer endorses → Endorsed
```

- [ ] **Step 2: Run them to verify they fail**

Run: `php artisan test --filter=AgendaProposalTest`
Expected: FAIL.

- [ ] **Step 3: Implement the five methods** in `AgendaService` with transaction, lock, audit and row-version handling as in Task 2.

- [ ] **Step 4: Run tests**

Run: `php artisan test --filter=AgendaProposalTest`
Expected: PASS.

- [ ] **Step 5: Commit** `feat(osc): put linked and standalone items forward with optional endorsement`

---

### Task 4: Secretary builds and publishes the agenda

**Files:**
- Modify: `app/Services/Osc/AgendaService.php`
- Create: `app/Notifications/Osc/AgendaPublished.php`
- Test: `tests/Feature/Osc/AgendaPublishingTest.php`

**Interfaces:**
- Consumes: Task 3.
- Produces:
  - `AgendaService::accept(AgendaItem $item, int $expectedVersion, User $actor): AgendaItem`
    - Manager only.
    - From `proposed`, or from `endorsed` when the department requires endorsement. Accepting a `proposed` item from such a department fails with `Cadangan ini perlu disokong ketua jabatan terlebih dahulu.`
    - Sets `position` to max + 1.
    - Audit `osc_item_accepted`.
  - `AgendaService::returnToDepartment(AgendaItem $item, int $expectedVersion, string $reason, User $actor): AgendaItem`
    - Manager only; `$reason` is required.
    - Sets status `returned`, `position` null, `return_reason`.
    - Audit `osc_item_returned`.
  - `AgendaService::reorder(Meeting $meeting, array $orderedItemIds, int $expectedMeetingVersion, User $actor): Meeting`
    - The ids must equal exactly the set of accepted items, otherwise ValidationException `Susunan agenda tidak lengkap.`
    - Rewrites `position` from 1 and increments the meeting's `row_version`.
    - Audit `osc_agenda_reordered`.
  - `AgendaService::publish(Meeting $meeting, int $expectedMeetingVersion, User $actor): AgendaVersion`
    - Requires at least one accepted item, otherwise `Agenda tiada perkara untuk diterbitkan.`
    - Increments `agenda_version` and writes an `osc_agenda_versions` row with `items` (accepted items in position order) and `changes` compared with the previous version.
    - Sets status `agenda_published` if it was `draft`.
    - Notifies every meeting member except the actor with `AgendaPublished`.
    - Audit `osc_agenda_published`.
  - `AgendaPublished` (mail): subject `Agenda Mesyuarat OSC {number} diterbitkan (versi {n})`. The body lists the item count and includes a link to `route('osc.meetings.show', $meeting)`. **It carries no paper attachments** (design S7).

- [ ] **Step 1: Write the failing tests**

```php
public function test_accept_order_and_publish_creates_version_one(): void
// accept 3 items, reorder [c,a,b], publish → agenda_version 1, items[0]['id'] = c, status AgendaPublished,
// Notification::assertSentTo(member, AgendaPublished::class); assertNotSentTo(secretary)

public function test_republish_records_changes(): void
// publish v1 with a,b; return b with reason; accept d; publish → v2 changes == ['added'=>[d],'removed'=>[b],'reordered'=>false]

public function test_concurrent_reorder_is_refused(): void
// two reorder calls with the same expectedMeetingVersion → second throws 'Rekod telah berubah. Muat semula sebelum meneruskan.'; positions match the first call

public function test_reorder_must_include_every_accepted_item(): void

public function test_endorsement_department_item_cannot_be_accepted_unendorsed(): void

public function test_publish_with_no_items_is_refused(): void

public function test_return_requires_reason_and_removes_position(): void

public function test_notification_has_no_attachments(): void
// toMail()->attachments === [] and rawAttachments === []
```

- [ ] **Step 2: Run them to verify they fail**

Run: `php artisan test --filter=AgendaPublishingTest`
Expected: FAIL.

- [ ] **Step 3: Implement the four methods and the notification.** `changes.reordered` is true when the relative order of ids present in both versions differs.

- [ ] **Step 4: Run tests**

Run: `php artisan test --filter=AgendaPublishingTest`
Expected: PASS.

- [ ] **Step 5: Commit** `feat(osc): accept, order and publish versioned agendas`

---

### Task 5: Papers

**Files:**
- Create: `app/Services/Osc/AgendaPaperService.php`
- Test: `tests/Feature/Osc/AgendaPaperTest.php`

**Interfaces:**
- Consumes: `App\Contracts\DocumentScanner::assertClean(string $path): void`; Tasks 2–4.
- Produces:
  - `AgendaPaperService::upload(AgendaItem $item, UploadedFile $file, User $actor): AgendaItemPaper`
    - Allowed actors: the item's presenter, its proposer, or a manager. Anyone else gets 403.
    - Accepts PDF only (MIME checked from content, as in `ApplicationDocumentService`), up to 50 MB. Otherwise ValidationException on `paper`.
    - Stores on the `local` disk under `osc/papers/{meeting_id}/{item_id}/v{n}.pdf` and scans the file.
    - If the scan fails or the scanner is unavailable: delete the file, create no row, and raise ValidationException `Dokumen gagal imbasan keselamatan dan tidak diterima.`
    - Version is max + 1.
    - Audit `osc_paper_uploaded` with the sha256.
  - `AgendaPaperService::download(AgendaItemPaper $paper, User $actor): StreamedResponse`
    - Requires `OscAccess::canViewMeeting`; managers may also download papers of draft meetings.
    - Re-checks the sha256 and aborts with 409 on mismatch.
    - Audit `osc_paper_viewed` (design S8).
  - `AgendaItem::isMissingPaper(): bool` is true when the item has no papers and `now()` is after the meeting's `paper_cutoff_at`.

- [ ] **Step 1: Write the failing tests**

```php
public function test_presenter_uploads_versioned_scanned_pdf(): void
// bind a scanner test double that passes (as tests/Feature/BangunanDocumentVerificationTest does); upload twice → versions 1,2; sha256 stored; audit has 'osc_paper_uploaded'

public function test_infected_or_unscannable_paper_is_refused_and_removed(): void
// double throws → message 'Dokumen gagal imbasan keselamatan dan tidak diterima.'; assertDatabaseCount('core.osc_agenda_item_papers', 0); Storage::disk('local')->assertDirectoryEmpty('osc/papers')

public function test_non_pdf_is_refused(): void

public function test_other_officer_cannot_upload(): void

public function test_download_audited_and_checksum_verified(): void
// member downloads after publish → 200 and audit 'osc_paper_viewed'; tamper the stored file → 409

public function test_missing_paper_flag_after_cutoff(): void
```

- [ ] **Step 2: Run them to verify they fail**

Run: `php artisan test --filter=AgendaPaperTest`
Expected: FAIL.

- [ ] **Step 3: Implement the service** following the store, scan and rollback sequence of `ApplicationDocumentService::upload()`.

- [ ] **Step 4: Run tests**

Run: `php artisan test --filter=AgendaPaperTest`
Expected: PASS.

- [ ] **Step 5: Commit** `feat(osc): versioned, scanned agenda papers with audited download`

---

### Task 6: Screens and routes

**Files:**
- Create: `app/Http/Middleware/EnsureOscAccess.php`; Modify: `bootstrap/app.php` (alias `osc.access`)
- Create: `app/Http/Controllers/Osc/MeetingController.php`, `AgendaItemController.php`, `AgendaPaperController.php`
- Create: `app/Http/Requests/Osc/StoreMeetingRequest.php`, `StoreAgendaItemRequest.php`, `ReturnAgendaItemRequest.php`, `ReorderAgendaRequest.php`
- Modify: `routes/web.php`
- Create: `resources/views/osc/index.blade.php`, `create.blade.php`, `show.blade.php`, `propose.blade.php`
- Modify: `resources/views/layouts/app.blade.php` (sidebar link), `resources/views/bangunan/show.blade.php` and `resources/views/pkm/show.blade.php` (button)
- Test: `tests/Feature/Osc/OscScreensTest.php`

**Interfaces:**
- Consumes: Tasks 2–5 services; `OscAccess`.
- Produces routes, all inside `Route::middleware(['auth', 'osc.access'])->prefix('osc')->name('osc.')`:

| Method | URI | Name | Action |
|---|---|---|---|
| GET | `/` | `osc.meetings.index` | Meetings the user may view, newest first |
| GET | `/meetings/create` | `osc.meetings.create` | Manager only |
| POST | `/meetings` | `osc.meetings.store` | `MeetingService::create` |
| GET | `/meetings/{meeting}` | `osc.meetings.show` | Manager: builder view; member: published agenda |
| GET | `/meetings/{meeting}/items/create` | `osc.items.create` | Query `application` (ULID, optional) pre-fills a linked item |
| POST | `/meetings/{meeting}/items` | `osc.items.store` | `proposeLinked` or `proposeStandalone` |
| POST | `/items/{item}/endorse` | `osc.items.endorse` | |
| POST | `/items/{item}/accept` | `osc.items.accept` | |
| POST | `/items/{item}/return` | `osc.items.return` | |
| POST | `/meetings/{meeting}/reorder` | `osc.meetings.reorder` | |
| POST | `/meetings/{meeting}/publish` | `osc.meetings.publish` | |
| POST | `/items/{item}/papers` | `osc.papers.store` | |
| GET | `/papers/{paper}` | `osc.papers.show` | Inline PDF (`Content-Disposition: inline`) |

**Screen content (BM copy is fixed; layout uses existing classes):**
- Sidebar link `eOSC+` with icon `gavel`, shown when `canAccessOsc()`.
- `index`: heading "Mesyuarat OSC", table of number, date, venue and status badge, and a "Mesyuarat baharu" button for managers.
- `show` for managers has three panels:
  - "Cadangan" (proposed and endorsed items, with Sokong / Terima / Kembalikan buttons)
  - "Agenda" (accepted items in order, with up/down controls posting to reorder, a missing-paper badge `Kertas belum diterima` using `status-attention`, and a "Terbitkan agenda" button)
  - "Versi" (published versions with their changes)
- `show` for members: published agenda only, as a single-column list on phones. Each row shows the number, title, reference, purpose, department, presenter and a paper link. Shows "Versi {n} · diterbitkan {date}".
- `propose`: purpose select; linked fields are read-only from the application; standalone fields are `standalone_type`, `title`, `reference`, presenter select.
- Bangunan and PKM application pages get a button "Bentang ke mesyuarat OSC" (icon `calendar-plus`), shown when the user can propose for that department. It links to `route('osc.meetings.index', ['application' => $application->id])`. When `application` is present, `index` shows a "Pilih mesyuarat" panel listing meetings in status `draft` or `agenda_published` with `scheduled_at` in the future. Each meeting links to `osc.items.create` with the same `application` query.
- Every action form carries the hidden `expected_version`.

- [ ] **Step 1: Write the failing tests**

```php
public function test_secretary_full_flow_through_http(): void
// create meeting → officer proposes via osc.items.store → secretary accepts, reorders, publishes; member GET show sees title and 'Versi 1'

public function test_non_member_staff_gets_403_on_meeting_and_paper(): void

public function test_member_cannot_see_draft_agenda_or_proposals(): void
// member GET show while Draft → 403; after publish, page does not contain a returned item's title

public function test_sidebar_and_application_button_visibility(): void
// building_officer sees 'eOSC+' and 'Bentang ke mesyuarat OSC' on bangunan.show; a user without osc permissions sees neither

public function test_pages_render_without_hard_coded_colours(): void
// rendered HTML of osc pages contains no '#[0-9a-fA-F]{3,6}' inside style attributes
```

- [ ] **Step 2: Run them to verify they fail**

Run: `php artisan test --filter=OscScreensTest`
Expected: FAIL.

- [ ] **Step 3: Implement the middleware, requests, controllers, routes, views and the two buttons.** Controllers only validate, call the services, and redirect back with `session('status')` messages in BM.

- [ ] **Step 4: Run the full suite and the style check**

Run: `php artisan test` then `php vendor/bin/pint --test` then `php artisan view:cache`
Expected: all pass, no Pint changes, views compile.

- [ ] **Step 5: Check the screens in a browser at desktop width and at 375 px** (the main screen and a phone), in both light and dark mode. Expected: no horizontal scroll, and the action buttons are reachable.

- [ ] **Step 6: Commit** `feat(osc): eOSC+ meeting, agenda builder and member agenda screens`

---

### Task 7: Documentation

**Files:**
- Create: `docs/runbooks/eosc-agenda.md`
- Modify: `docs/eosc/README.md` (status line), `docs/eosc/plans/README.md` (plan 1 status), `progress.md`; mirror `docs/eosc/README.md` and `docs/eosc/plans/` to `C:\Users\Asyra\Documents\eOSC+\docs\` (owner keeps both copies updated)

- [ ] **Step 1: Write the runbook.** Cover:
  - How to grant `osc_secretary` and `osc_steno` roles.
  - How to switch on department endorsement (a row in `core.osc_department_settings`).
  - How to create a meeting and the cut-offs.
  - What members see.
  - That only staff accounts can attend until Plan 4 ships.

- [ ] **Step 2: Update the status lines and `progress.md`** with the test count from the last full run, then copy the docs to the eOSC+ mirror.

- [ ] **Step 3: Commit** `docs(osc): agenda runbook and plan 1 progress`
