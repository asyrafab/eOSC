# eOSC+ meetings and agenda

Implemented increment: eOSC+ Plan 1 (foundation and agenda builder), 2026-10-29, on branch `feat/eosc-foundation` (not yet merged). Decision record: ADR-314. eOSC+ is core capability C10 inside iNDECS; screens are under `/osc`. This increment covers meetings, agenda items, papers and agenda publishing only. The meeting map (Plan 2, ADR-315) is described in the section "Meeting map" below and in `gis-layers.md`; there is no decision panel, attendance or external attendee access yet (later plans).

## Deployment

After backing up the target database and reviewing the additive migration, run from the iNDECS root with the correct environment configuration:

```sh
php artisan migrate --path=database/migrations/2026_10_29_000000_create_osc_meetings_and_agenda.php --force
php artisan db:seed --class=OscSeeder --force
```

The migration creates `core.osc_meetings`, `core.osc_meeting_members`, `core.osc_agenda_items`, `core.osc_agenda_item_papers`, `core.osc_agenda_versions` and `core.osc_department_settings`. `OscSeeder` adds the roles `osc_secretary` and `osc_steno`, the permissions `osc.meeting.view`, `osc.meeting.manage`, `osc.agenda.propose`, `osc.agenda.endorse` and their grants. It creates no meetings or user accounts and can be rerun without duplicates. Do not rerun the full DatabaseSeeder on the consolidated development database just to enable this increment.

Rollback refuses to drop the tables once any meeting exists; use a reviewed data-preserving forward fix instead.

Papers can be up to 50 MB. PHP must accept that: set `upload_max_filesize` and `post_max_size` to at least `50M` (`post_max_size` slightly larger is safer) in the PHP configuration used by the web server, and allow the same body size in the web server or reverse proxy. Check with `php -i | grep -E "upload_max_filesize|post_max_size"` for the CLI and a phpinfo check for the web SAPI. With lower limits an upload fails before it reaches iNDECS.

Times: every date and time in eOSC+ (meeting time, cut-offs, published-at labels) is entered and shown in Malaysian local time (`Asia/Kuala_Lumpur`, UTC+8), set by `DISPLAY_TIMEZONE` (config `app.display_timezone`). The database stores UTC. Do not change the application timezone (`config/app.php`) for eOSC+.

## Granting roles

A staff account has one role in `users.role`. Set it to `osc_secretary` or `osc_steno` for the people who run the meeting. Both receive `osc.meeting.view` and `osc.meeting.manage` and are called "managers" below. They can sign in with that role alone and land on `/osc`.

Who can open `/osc` (only active staff accounts, `account_type = staff`, `is_active = true`):

- holders of `osc.meeting.view` (the roles in the table below, the OSC roles and `system_admin`);
- any staff account that is the chairperson, secretary, STENO or a member of a meeting, or the presenter or proposer of an agenda item, whatever its role. Such a person can sign in even with no other iNDECS access and lands on `/osc`.

Everyone else gets 403 on `/osc`, and an account with no other iNDECS workspace is refused at sign-in. Opening `/osc` does not show every meeting: each meeting and item is still checked individually (see "What people see").

Existing roles already get grants from the seeder:

| Role | Department | Permissions |
|---|---|---|
| `planning_officer` | JPBD | view, propose |
| `senior_planning_officer` | JPBD | view, propose, endorse |
| `building_officer` | JBANG | view, propose |
| `senior_building_officer` | JBANG | view, propose, endorse |
| `system_admin` | all | view, manage, propose, endorse |

The `senior_*` roles act as heads of department for endorsement. Any other role needs a reviewed new grant in `core.role_permissions`; do not edit grants ad hoc.

## Creating a meeting

A manager opens `/osc` and chooses to create a meeting. Enter the meeting number (unique per meeting type), date and time in Malaysian local time (must be in the future), venue, and two cut-offs, both on or before the meeting time:

- **Tutup cadangan agenda** (proposal cut-off): after this, officers cannot propose items. Managers still can.
- **Tutup kertas kerja** (paper cut-off): accepted items without a paper after this time are flagged as missing a paper.

Choose the chairperson (set per meeting, not a fixed role), secretary, optional STENO and the members. All must be active staff. Any active staff account may be chosen; being chosen is enough to open the meeting in `/osc` once its agenda is published. Members are set when the meeting is created. A new meeting starts as a draft.

### Changing members before a screen exists

Plan 1 has no screen for changing members (planned for Plan 3). Until then an operator makes the change on request from the meeting secretary, through the audited service method, never by editing `core.osc_meeting_members` directly:

1. Record the request (who asked, the meeting number, the full new list) in the change ticket.
2. Check that every person is an active staff account: `SELECT id, name, account_type, is_active FROM users WHERE id IN (...);` The service does not repeat the screen's staff check, so do not continue if any row is not `staff` and `true`.
3. Run `php artisan tinker` on the application server with the production environment and call the service as the requesting secretary (an account holding `osc.meeting.manage`):

```php
$meeting = App\Models\Osc\Meeting::query()->where('meeting_type', 'jawatankuasa_osc')->where('number', 'Bil. 21/2026')->firstOrFail();
$actor = App\Models\User::query()->where('email', 'secretary@mbsp.gov.my')->firstOrFail();
app(App\Services\Osc\MeetingService::class)->setMembers($meeting, [
    'chair_id' => 12, 'secretary_id' => 34, 'steno_id' => 56, // steno_id may be null
    'member_ids' => [78, 90],
], $meeting->row_version, $actor);
```

The list replaces the whole membership, so give the chairperson, secretary, STENO and every member each time. The call uses the meeting's current `row_version`; if someone changed the meeting in between it is refused ("Rekod telah berubah") and you rerun step 3. It is refused for closed or minutes-confirmed meetings. It writes an `osc_meeting_members_changed` audit row with the before and after lists and the secretary as actor; quote that audit row id in the ticket.

## Proposing items

- **Linked item:** on a Bangunan or PKM application page the officer uses the button "Bentang ke mesyuarat OSC", picks a meeting and the purpose, and names the presenter. The same application can be tabled again at a later meeting (repeated tabling).
- **Standalone item:** from the meeting screen, for matters with no application. The officer's own department is used. A manager must choose the department.

Officers need `osc.agenda.propose` for the item's department. Proposals are refused after the proposal cut-off (managers excepted) and on meetings that are in progress, closed or minutes confirmed.

## Endorsement by the head of department (optional)

Endorsement is off by default. To switch it on for a department, insert or update a row in `core.osc_department_settings`:

```sql
INSERT INTO core.osc_department_settings (department_id, requires_endorsement, created_at, updated_at)
VALUES ('JPBD', true, now(), now())
ON CONFLICT (department_id) DO UPDATE SET requires_endorsement = true, updated_at = now();
```

Set `requires_endorsement` back to false to switch it off. There is no screen for this yet. When on, that department's new items wait for a head (a `senior_*` role holder with `osc.agenda.endorse` for that department) to press "Sokong" before the secretary can accept them. When off, items go straight to the secretary.

## Secretary: accept, return, order, publish

Managers review proposed items on the meeting screen:

- **Terima** accepts the item onto the agenda.
- **Kembalikan** sends it back to the department with a required reason, which the proposer sees.
- Accepted items are ordered with the reorder action.
- **Terbitkan agenda** publishes. Each publish creates a new version (1, 2, ...) that stores the ordered items and a list of changes against the previous version. Members are emailed a link only; the agenda and papers are never in the email.

Agenda changes are refused once the meeting is closed or minutes confirmed. Every change carries the row version, so a stale screen is rejected and must be reloaded.

## Papers

The item's presenter, proposer or a manager can upload a paper. Rules: PDF only, up to 50 MB, virus-scanned by ClamAV. The file is stored first and then scanned; if the scan detects a virus, the scanner is unavailable, or it errors or times out, the stored file is deleted and the upload never becomes a version (there is no bypass). Each upload is a new version and old versions are kept with a SHA-256 checksum. Every view of a paper is audited. Configure ClamAV as described in the document-verification runbook.

## What people see

| Person | Sees |
|---|---|
| Manager (secretary, STENO, system admin) | Everything for all meetings |
| Meeting member | Only the latest published agenda and the papers of items in it, once the meeting is no longer a draft |
| Presenter or proposer | Their own items, whatever the status, including a return reason |
| Head of department | All items of their own department, and the Sokong action |

The `/osc` list shows the same meetings: managers see all; others see meetings where they are a member (once published), presenter or proposer, or where their department has an item they may endorse. A head of department also sees how many items are waiting for their endorsement.

## Staff accounts only

Until Plan 4 (attendee access) ships, members, chairperson and presenters must be staff accounts. External attendees (PSP, owner, outside agencies) cannot be added or log in. Plan meetings with staff only.

## Validation

`php artisan test` covers meetings, linked and standalone proposals, cut-offs, endorsement, accept/return, versioned publishing and changes, papers, visibility, closed-meeting refusals, sign-in for eOSC+-only staff and members, and cut-offs in Malaysian local time. Use the configured test database, never a production connection.

## Meeting map

Added by Plan 2 (branch `feat/eosc-map`, not yet merged). Setup, layer replication and the legacy import are in `docs/runbooks/gis-layers.md`; this section is the user guide.

- **Open the map:** on a meeting page use the "Peta" button (`osc.meetings.map`). Anyone who can view the meeting can open it. It shows each agenda item's site over lot, zoning and utility layers, with past OSC decisions (MJKP) as points, and the item's layout plan placed on the map. Reference data is copied from MBSP ArcGIS and can be up to one refresh old; the map never calls ArcGIS itself.
- **Site boundary:** the officer who registers the application draws the boundary on the registration page. Without a boundary the map falls back to the matched lot, or shows the item without a location.
- **Layout plan:** a manager, the item's presenter or its proposer opens the item's layout page (`osc.layout.edit`), uploads a JPEG or PNG (up to 20 MB) and marks three reference points. The plan then appears in place on the map. Legacy plans from the old MJKP system were placed automatically where a world file existed.
- **Basemap:** Esri imagery or street map, or the MBSP orthophoto once `ORTHOPHOTO_TILE_URL` is set. If imagery cannot load, a notice appears and the local layers keep working.
- **Refresh layers:** managers see the last update time and status of each layer on `/osc` and can start a refresh with "Kemas kini lapisan". A queue worker must be running. Do this at least yearly.
