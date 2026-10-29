# eOSC+ — OSC meeting room for iNDECS

> Mirror copy. The canonical version is `C:\Users\Asyra\Documents\iNDECS\docs\eosc\README.md`; both are updated together.

> **Status:** Design v1.1 **approved by the owner 2026-10-29** (builder check done). Plan 1 (foundation and agenda) built on branch `feat/eosc-foundation`, not yet merged. Next: Plan 2 (map).
> Owner: JTIK (builds). Operated by OSC, Jabatan Perancang Bandar and Jabatan Bangunan.
> Decision records: ADR-212 (concurrent modules), ADR-213 (eOSC+ inside iNDECS) in `docs/bangunan/16-decision-log.md`.
> Mirror copy: `C:\Users\Asyra\Documents\eOSC+\docs\eosc-plus-design.md`. Both are updated together.
>
> Marking: **[owner]** = confirmed by the owner. **[call]** = the designer's call, open to owner correction
> (listed in section 18). **[builder]** = left to the builder. **[open]** = owner will settle later.

---

## 1. What this is and why

eOSC+ is the screen the **Jawatankuasa OSC** meeting runs on. Applications are tabled, discussed on a map
with their papers and plans, and decided by the chairperson in the system. Members follow and comment on
their own phones. The PSP or Owner can be brought in by video call when the committee needs clarification.

It is built **inside iNDECS** as the meeting-room screens of core capability **C10 Meetings and decisions**
(`docs/bangunan/11-indecs-core-roadmap.md`) **[owner]**. It replaces the legacy eOSC / MJKP system
(`O:\Projects\mjkp`: PHP circa 2010, Access `mpsp_mjkp.mdb`, MapGuide 6.5 ActiveX viewer).

The legacy system still works, but:

| Problem today | Consequence |
|---|---|
| One table (`Kertas_kerja`) is hard-wired to Kebenaran Merancang fields | Pelan Bangunan, Penamaan Jalan & Taman and future types cannot be added |
| Attendees only watch the main screen | They cannot open papers, download, comment or look at additional data |
| Lots and satellite imagery are years old; no reference layers | Members switch to Google Maps during the meeting; the map adds nothing |
| Departments email or hand title lists to the STENO a week ahead; she builds the agenda; full papers arrive as late as the day before | Manual, error-prone agenda work |
| STENO records decisions by hand and drafts minutes for verification at the next meeting | Slow minutes; decisions recorded as a status code plus one comment |

**Success looks like:** the STENO no longer types the agenda or handwrites decisions; nobody opens Google
Maps during the meeting; every member can read the paper on their phone; the chairperson's decision is
recorded, signed and (where appropriate) released to the applicant the same day; no paper leaks without a
trace.

## 2. Goals, non-goals, anti-goals

**Goals**

1. Any application type can be tabled, including types whose iNDECS module does not exist yet.
2. The agenda is built from what departments put forward in the system, not from emailed lists.
3. A map that is current and useful enough that members stay in eOSC+.
4. The chairperson's decision is recorded in the system, in a fixed sequence, and is final when given.
5. Attendees, internal and external, use eOSC+ on their own phones, securely.
6. The PSP or Owner can be called in by video for clarification.
7. The meeting runs even when ArcGIS Enterprise is down.

**Non-goals**

- eOSC+ is not where applications are registered or assessed; that stays in the iNDECS department modules.
- eOSC+ does not draw application boundaries (section 9.3).
- eOSC+ is not the master GIS; ArcGIS Enterprise stays the master.
- No native mobile app in an app store.

**Anti-goals (failure even if it technically works)**

- A meeting stalls because the ArcGIS portal, the video server or one module is unavailable.
- Papers or plans leak and nobody can tell from whom.
- A PSP or Owner sees another applicant's papers.
- The chairperson can press a decision button before the presenting officer's proposal exists, or a decision
  is changed after it was given.
- AI text is presented as a decision (AGENTS.md: AI suggests, the officer decides).
- eOSC+ becomes a separate system with its own login, database or look.

## 3. Constraints

- **Inside iNDECS** **[owner]**: Laravel, PostgreSQL 16+ with PostGIS, iNDECS identity (C1), RBAC (C2),
  append-only audit and history (ADR-306), documents (C4), letters and e-approval (C9), notifications (C8).
  Follow `AGENTS.md` and `docs/DESIGN-DIRECTION.md` exactly: Calm Intelligence tokens, Google Sans, Lucide
  icons, Bahasa Melayu interface, light/dark by token only, existing classes in `public/css/indecs.css`.
- **Core, not a department module.** Meetings serve every department, so eOSC+ tables live in the `core`
  schema. Department data stays in department extension tables; cross-module effects go through domain
  events (C11).
- **Naming.** `DESIGN-DIRECTION.md` section 1 has no route or table prefix for meetings. The builder records
  one in an ADR before writing code **[builder]**; the product name shown to users is **eOSC+**.
- **Never depend on ArcGIS Enterprise during a meeting** **[owner]** (portal at `spgis.mbsp.gov.my`
  disconnects often).
- **Concurrent delivery** **[owner]** (ADR-212): eOSC+ is built at the same time as the department modules.
  It must work before PKM, Kejuruteraan, Landskap or a Penamaan Jalan module are on the workflow engine.
- **Web only** **[owner]**: a responsive web application, installable to the home screen.
- **Decisions are written before code** in `docs/adr/` (AGENTS.md).

## 4. People and roles

| Role | Who | Signs in with | Can do |
|---|---|---|---|
| Pengerusi (chairperson) | Usually the Datuk Bandar; may be the Setiausaha Bandaraya or another member **[owner]** | Staff account | Give decisions; comment; everything a member can |
| Setiausaha OSC (meeting admin) | OSC secretary | Staff account | Create meetings, invite, manage seats and registrations, build and publish the agenda, run the main screen |
| STENO | OSC stenographer | Staff account | Build agenda (with the secretary) **[call]**; write discussion notes per item; prepare minutes |
| Pegawai pembentang (presenting officer) | Officer of Perancang, Bangunan or another department | Staff account | Put items forward; attach papers; present; write the proposed decision and syarat kelulusan |
| Pegawai ditugaskan (assigned officer) | Officer assigned by the secretary or the department | Staff account | Write the proposed decision and syarat on an item in place of the presenting officer **[owner]** |
| Ahli mesyuarat (member) | Council members and internal officers | Staff account, or invitation + phone OTP if they have none **[call]** | Read the agenda and papers, view the map, comment |
| Wakil jabatan teknikal | Officers of internal and external technical departments | Department invitation link + phone OTP **[owner]** | Read the items allowed to them, comment |
| PSP / Owner | Applicant's representative | Single-use video link | Join a video call for their own item only |

The chairperson is **set per meeting**, not hard-coded to a person **[owner]**. One person may hold several
roles in a meeting (e.g. an officer presents one item and is a member for the rest).

## 5. Concepts

| Concept | Meaning |
|---|---|
| Meeting type | A kind of committee with its own default members, chair, quorum rule and application types. Only Jawatankuasa OSC is needed (its decision is final **[owner]**); other committees can be added later on the same model **[call]** |
| Meeting (Mesyuarat) | One sitting: number (e.g. Bil. 21/2026), date, time, venue, chair, secretary, status |
| Agenda item (Perkara) | One thing tabled: **one agenda item = one application [owner]** (combined developments are not tabled as one item for now). Either **linked** to an iNDECS application, or **standalone** (title, department, application type, location, paper) for types with no iNDECS module yet, e.g. Penamaan Jalan & Taman **[call]**. A standalone item can be linked to an application later |
| Paper (Kertas) | Kertas pembentangan plus attachments (plans, photos, reports) for an item, versioned |
| Proposed decision | Written by the presenting or assigned officer: decision text and syarat kelulusan |
| Tabling purpose (Tujuan pembentangan) | Why the application is on the agenda this time: e.g. Permohonan, Rayuan, Pertimbangan semula, Pindaan; the list is configurable **[call]** |
| Decision (Keputusan) | Given by the chair: **Lulus**, **Lulus dengan Syarat**, **Tolak** or **Tangguh** **[owner]**. **The OSC decision is the final decision on the application [owner]**; no later meeting confirms it |
| Comment (Ulasan ahli) | A member's own written comment on an item, attributed to them |
| Discussion note (Catatan perbincangan) | STENO's note per item |
| Invitation | A personal link (one person) or a department link (one department, several officers) for one meeting |
| Seat (Tempat) | A department's allowance of registrations on its link (section 10) |
| Signed plan | The digital layout plan stamped with the chair's signature when a decision is given (section 8.3) |

## 6. How a meeting runs

```
Before                          During                                   After
------                          ------                                   -----
Secretary creates meeting   →   Secretary opens meeting (main screen)  →  Meeting closed
Departments put items forward   For each item:                           Draft minutes generated (F13)
Secretary orders, publishes       Officer presents (map, paper, plans)   Minutes verified at next meeting
Invitations sent                  PSP/Owner video call if needed         Decisions flow to iNDECS cases
Attendees register (OTP)          Members comment, STENO notes           Signed plans released where approved
Attendees pre-read, comment       Officer enters proposed decision       Dashboard updated (F14)
                                  Chair gives decision → final, signed
```

**Meeting states [call]:** Draf → Agenda diterbitkan → Sedang berlangsung → Selesai → Minit disahkan.

**Agenda item states [call]:** Dicadangkan (put forward) → Dalam agenda → Sedang dibentang →
Keputusan direkod. An item put forward but not accepted onto the agenda returns to the department.

**Repeated tabling [owner]:** an application can be tabled several times (first application, appeal,
reconsideration and so on) at different meetings or even the same meeting. Each tabling is its own agenda
item with its own purpose, paper and decision; the application keeps the full history, and its current
decision is the latest one given **[call]**. The map's application layer and the item screen show earlier
tablings of the same application.

**Tangguh** **[owner]**: rare, used when an application cannot be decided at that meeting. A Tangguh item is
offered automatically for a later meeting's agenda, carrying its history **[call]**.

## 7. Features and priorities

All 14 features are in scope **[owner]**. **F1, F7 and F9 are built first** **[owner]**, together with the
attendee access they need (section 10).

| # | Feature | When | Priority |
|---|---|---|---|
| F1 | Agenda builder | Before | **First** |
| F2 | Auto-drafted kertas pembentangan (AI suggests, officer edits) | Before | |
| F3 | Pre-read on mobile with per-item comments before the meeting | Before | |
| F4 | Technical-department ulasan tracker per item | Before | |
| F5 | Offline pack: papers and map tiles cached on devices | Before | |
| F6 | "Follow the presenter": main screen drives phones; free browsing with tap-to-rejoin | During | |
| F7 | Map | During | **First** |
| F8 | Plan viewer: plans in the browser, revisions side by side, markup | During | |
| F9 | Chairperson's decision panel, with attendance, quorum, conflict-of-interest declarations | During | **First** |
| F10 | STENO per-item notes and optional item timer | During | |
| F11 | Video call to PSP or Owner | During | |
| F12 | Decisions advance iNDECS workflow and trigger letters (SKP etc.) | After | |
| F13 | Draft minutes from decisions, comments and notes | After | |
| F14 | Decision dashboard: by month, type, area; submission-to-decision time; replaces MJKP reports | After | |

## 8. Priority features in detail

### 8.1 F1 Agenda builder

**Today:** Perancang and Bangunan each send the STENO a list of titles about a week before; she builds the
agenda; full papers can arrive the day before.

**In eOSC+:**

1. The secretary creates a meeting: type, number, date, time, venue, chair, cut-off date for putting items
   forward, cut-off for papers **[call]**.
2. Officers put items forward for a chosen meeting:
   - **Linked item:** from the application's **Mesyuarat** tab in the iNDECS workspace (the tab already exists
     in the 17-tab shell). Title, location, boundary and documents come from the application.
   - **Standalone item:** from eOSC+ directly, for types without a module: title, department, application
     type, reference, location (boundary drawn by the presenting officer, section 9.3), paper.
3. **Who may put an item forward, and whether a head of department must approve it first, is [open]**: the
   owner will settle it later. Build it as permissions plus an optional per-department approval step that can
   be switched on without code changes **[call]**.
4. The secretary sees one combined list, accepts or returns items, orders them (grouped by department and
   type by default), and publishes. Publishing notifies invitees. Re-publishing after a change creates a new
   agenda version and tells attendees what changed **[call]**.
5. A title may be on the agenda before its paper. Items without a paper by the paper cut-off are flagged on
   the secretary's screen and the attendee agenda **[call]**.

### 8.2 F7 Map

**Layers, top to bottom [owner]:**

| # | Layer | Content | Source |
|---|---|---|---|
| 1 | Application | Boundary of each item on the agenda (highlighted) and past applications and decisions nearby (muted) | iNDECS applications and standalone items; legacy MJKP records migrated as points (section 14) |
| 2 | Scanned layout imagery | Scanned layout plans placed in position on the map, with a transparency slider | Uploaded and positioned at registration (section 9.3); legacy MJKP georeferenced images (~4,300 `.jgw`, ~280 `.tfw`) migrated |
| 3 | Cadastral lot | Lot boundaries and numbers | Replicated from ArcGIS Enterprise |
| 4 | Imagery | Current satellite / aerial imagery | Live: Esri World Imagery **[call]**; fallback: MBSP's own orthophoto stored locally **[call]** |
| 5 | Additional | Zoning, roads, utilities and others, each switchable | Replicated from ArcGIS Enterprise; exact list **[open]** |

**Replication [owner]:** layers 3 and 5 are copied from ArcGIS Enterprise into iNDECS PostGIS **about once a
year**, plus an on-demand **"Kemas kini lapisan"** action for an administrator. The meeting reads only the local
copy. A refresh writes a new snapshot and switches over only when complete; the previous snapshot is kept so a
failed refresh never leaves the map empty **[call]**. The refresh method (feature-service query export, file
geodatabase export, or ArcGIS Pro script) is **[builder]**.

**Tools [owner-approved feature list]:** zoom to item, layer switcher, transparency per layer, measure
distance and area, swipe between imagery and scanned layout (or old and new imagery), open Google Street View
at the site in a new tab **[call]**, tap a feature to see its details and open the paper.

**Behaviour:** when the presenter moves to an item, the main screen zooms to its boundary (F6 later makes the
phones follow). On phones the map is full-screen with a bottom sheet for item details **[call]**.

**Offline:** if the internet drops, the map keeps working from the local layers and the orthophoto; only live
imagery and Street View are lost **[call]**.

### 8.3 F9 Chairperson's decision panel

**Sequence [owner]:** presenting officer first, then chairperson. Buttons are only available in that order.

1. The **presenting or assigned officer** enters the proposed decision: decision type, decision text and
   syarat kelulusan. Conditions are picked from a standard conditions library plus free text **[call]**;
   condition types replace the legacy split of "Lulus Dgn Syarat Tambahan" and "Lulus Dgn Syarat Teknikal"
   **[call]**. Who maintains the library is **[open]**.
2. Only then do the chair's four buttons become active: **Lulus**, **Lulus dengan Syarat**, **Tolak**,
   **Tangguh** **[owner]**. If the chair wants a different decision from the one proposed **[call]**, the chair
   marks the intended decision, the officer revises the proposal on the spot (e.g. adds syarat when the chair
   wants Lulus dengan Syarat instead of the proposed Tolak), and only then can the chair give it. The
   officer-then-chair order holds and nothing is completed after the decision.
3. Pressing a button asks the chair to re-confirm their identity (C9 MFA re-prompt) **[call]** and shows a
   one-screen summary before the final press **[call]**.
4. **The decision is final the moment it is given** **[owner]**. It cannot be edited. A correction is a new,
   attributed record that references the original, never an overwrite (ADR-306) **[call]**.
5. **Signed digital plan [owner]:** on Lulus or Lulus dengan Syarat, the chair's signature is applied to the
   item's digital layout plan. Using C9 e-approval: chair's registered signature specimen, document hash,
   verification QR, public verify page **[call]**. Tolak and Tangguh produce no stamp **[call]**. Every person
   who may chair holds a registered signature specimen.
6. **Immediate release:** some approvals are released to the Owner / PSP straight after the meeting. Which
   ones, and who decides, is **[open]** (to be settled during development). Build release as an explicit
   action with its own permission, not automatic **[call]**.
7. Members' comments and STENO notes on the item stay visible on the panel throughout.

**Who sees members' comments [owner]:** every attendee sees every other attendee's comments, before and
during the meeting, including external technical departments. PSP / Owner have no access to comments (they
only join the video call). Minutes (F13) may quote any comment. **[call]** STENO notes are visible to staff
attendees only until the minutes are drafted.

**Also on the panel:** attendance (from registrations and sign-ins, confirmed by the secretary), quorum
indicator using the meeting type's rule (rule **[open]**, configurable), and conflict-of-interest declarations
per item: a member who declares is marked and excluded from that item's comments **[call]**.

**After the decision:** for a linked item, a domain event moves the application's iNDECS workflow (F12) and the
decision appears on the application's **Keputusan** tab. **The eOSC+ decision is the only authoritative
committee decision from the first live meeting [call].** Until full F12 is built, a minimal version ships with
F9: the decision is written to the application's record, and the department module's own committee-decision
entry for that application becomes read-only, showing the eOSC+ decision. Officers never re-enter it. For a standalone item, the decision is recorded on the
item and carried over when it is later linked to an application **[call]**.

## 9. Other features (summary for planning)

### 9.1 Before the meeting

- **F2** Draft kertas pembentangan from case data through the iNDECS internal AI layer (C14). Shown as an AI
  block, edited and saved by the officer; never published without officer action.
- **F3** Pre-read: published agenda and papers on phones; members leave comments per item before the meeting.
- **F4** Ulasan tracker: per item, which consulted technical departments have or have not returned comments
  (from the application's Ulasan Teknikal tab for linked items; entered by the officer for standalone items).
- **F5** Offline pack: before the meeting, the installed web app caches the meeting's papers and map tiles,
  encrypted, and wipes them when access expires (S11).

### 9.2 During the meeting

- **F6** Follow the presenter: the secretary's main screen broadcasts the current item and map view; phones
  follow until the user browses away; a "Kembali ke pembentang" button rejoins.
- **F8** Plan viewer: PDF and images in the browser; DWG converted to PDF or image on upload **[builder]**;
  revisions side by side; markup visible to the meeting only.
- **F10** STENO notes per item; optional per-item timer shown to the secretary and chair.
- **F11** Video call: section 11.

### 9.3 Boundaries and positioned layout plans (in iNDECS registration, used by eOSC+)

**[owner]** Application boundaries are drawn at registration in iNDECS, on the application, by the
registration officer **or** by the PSP / Owner when they submit. eOSC+ only displays them. This needs a
map-drawing tool in core C13 GIS, used by every module's registration form and by the public submission path.
**[call]** Standalone items get their boundary drawn by the presenting officer when put forward. **[call]**
Scanned layout plans are positioned the same way, with a simple control-point tool (pick matching points on the
plan and on the map).

### 9.4 After the meeting

- **F12** Decisions on linked items publish domain events; each department module's workflow consumes them and
  generates its letters through C9 (e.g. SKP series for Bangunan).
- **F13** Draft minutes assembled from decisions, proposed decision text, syarat, members' comments and STENO
  notes; edited by the STENO; marked "Disahkan" when the next meeting verifies them.
- **F14** Dashboard of decisions by month, type, department, area (mukim / DUN), and submission-to-decision
  time, including migrated MJKP history.

## 10. Attendee access and leak protection

**Invitations [owner]:** before each meeting, attendees receive an invitation by email or letter with a unique
link or QR code. Opening it, they register for that meeting with their **own phone number**, confirmed by
**OTP**. eOSC+ is web-based.

**Two kinds of link:**

| Link | For | Rule |
|---|---|---|
| Personal | One named person (e.g. a council member without a staff account) | Tied to the phone number the secretary entered; OTP goes only to that number **[call]** |
| Department | One technical department, internal or external | Officers register with name, jawatan, district (SPU, SPT, SPS, or none for headquarters / senior officers; not compulsory) and own phone + OTP **[owner]** |

**Department seats [owner]:** each department link has a seat allowance, **default 1**. The secretary can set a
different allowance per department (e.g. 3 for a department that sends one officer per district), and the
setting is remembered for that department's future invitations **[call]**. When the seats are full, the link
offers **"Mohon tambahan tempat"**: the extra officer registers with OTP and waits until the **secretary
approves**. The secretary can **remove any registration** at any time, ending that person's access at once.

**MBSP staff** sign in with their staff account instead of OTP **[call]**.

**Safeguards [call, recommended to the owner]:**

| # | Safeguard |
|---|---|
| S1 | Personal links bound to a phone; department links bound to seats and secretary approval; the department is notified of every registration under its link |
| S2 | A registration binds to one device; a new device needs a fresh OTP |
| S3 | Access from invitation until a set number of days after the meeting **[open: how many]**, then expires; secretary can revoke anyone instantly |
| S4 | Each attendee sees only their meeting, and only the items allowed to them (an external department sees the items it was consulted on) |
| S5 | Visible watermark on every screen, paper and plan: viewer name, masked phone, date and time |
| S6 | View in the browser by default; download is a per-role permission (internal yes, external no by default); downloads carry the watermark |
| S7 | Invitations carry only the link, never papers |
| S8 | Every view, download, comment and decision written to the append-only audit log |
| S9 | Staff accounts for staff; OTP only for those without one |
| S10 | PSP / Owner see only their own item through the video link; never the agenda or other papers |
| S11 | Offline pack encrypted on the device, wiped when access expires |
| S12 | OTP rate limiting, short session timeouts, phone numbers handled under PDPA |

OTP is sent by SMS through the core notification adapter (C8); the SMS gateway is shared with iNDECS
**[call]**; its procurement is **[open]**.

## 11. Video call to PSP or Owner (F11)

- **Technology [call]:** a self-hosted **Jitsi Meet** server at MBSP, opened from inside eOSC+. No per-user
  licence; calls stay in-house. If MBSP already licenses Microsoft Teams or Google Meet, the owner may choose
  that instead; the integration point is the same (create a room, hand out a link).
- The secretary or presenting officer starts a call for one item. The PSP / Owner receives a **single-use link**
  by SMS or email, waits in a **waiting room**, and is admitted by the secretary. They can share their screen.
  The link dies when the item closes (S10).
- The main screen shows the call beside the map. Recording is off by default **[call]**.
- If the video server is down, the meeting carries on; the item can be marked for Tangguh or for a phone call.

## 12. Screens by role

| Role | Main view |
|---|---|
| Secretary (main screen, PC on the big display) | Agenda list on the left; map, paper and plans in the centre; item controls (present, start call, open panel) |
| Secretary (admin) | Meetings, invitations and seats, registrations (approve/remove), agenda builder |
| Chairperson (tablet or phone) | Current item summary, proposed decision and syarat, members' comments, the four decision buttons (active only when allowed) |
| Presenting officer | Their items, paper upload, proposed decision and syarat form |
| STENO | Current item, notes field, timer, minutes draft |
| Member / technical department (phone) | Agenda, current item following the presenter, map, paper, comment box |
| PSP / Owner | Video call only |

All screens follow the iNDECS workspace shell and Calm Intelligence direction; the phone layouts use the
existing responsive patterns **[builder]**.

## 13. Integration

| With | How |
|---|---|
| iNDECS department modules | Items put forward from the application's Mesyuarat tab; decisions returned as domain events to the Keputusan tab and workflow (C3, C11) |
| C9 letters and e-approval | Chair's signature, hash, QR verification on the signed plan; module letters after the decision |
| C8 notifications | Invitations, agenda published, OTP SMS, call links |
| C13 GIS | Boundary drawing and plan positioning at registration; PostGIS layers |
| ArcGIS Enterprise | Source of replicated layers only; never called during a meeting |
| OSC3Plus | No direct link in the first release; applications reach eOSC+ through their iNDECS modules **[call]** |
| Sistem Bangunan 2.0 | Its `PmnKeputusanOSC` field is replaced by eOSC+ decisions at Bangunan cut-over. Because the OSC decision is final **[owner]**, there is no separate YDP decision in eOSC+; legacy `PmnKeputusanYDP` values are migrated as history only **[call]** |

## 14. Legacy migration

| Source | Content | Into eOSC+ |
|---|---|---|
| `O:\Projects\mjkp\DB\mpsp_mjkp.mdb`, table `Kertas_kerja` | 7,694 historical records (sync log 2026-10-28), of which 2,288 have no XY; title, status, comment, mukim, seksyen, attachment names | Historical meetings and items with decisions mapped: 1 → Lulus dengan Syarat (tambahan), 2 → Lulus dengan Syarat (teknikal), 3 → Tangguh, 4 → Tolak |
| `O:\Projects\mjkp\web\doc`, `plan`, `gambar` | ~5,700 papers, ~15,000 plans, ~13,000 photos | Attachments of the historical items, with checksums |
| Georeferenced `.jgw` / `.tfw` images | ~4,300 + ~280 | Layer 2 scanned layout imagery |
| `O:\Projects\mjkp\Data\DWG`, `SHP` | Drawings and site shapefiles | Evaluated; site shapes become boundaries where they match **[builder]** |
| `sync_mjkp.py` | MDB → ArcGIS feature-layer sync | Retired when eOSC+ holds the records |

Records without XY (the sync already exports them as `missing_xy_*.csv`) are migrated without a location and
listed for cleanup **[call]**. The coordinate system of the legacy XY is determined from the shapefile `.prj`
files **[builder]**. Migration is rehearsable and repeatable without duplicates, as in the PKM backlog's
migration rules.

## 15. Non-functional requirements

- **Availability:** the meeting survives loss of ArcGIS Enterprise (always), loss of the video server (call
  only lost) and loss of internet (live imagery and Street View lost; everything else works from the local
  network) **[call]**.
- **Scale [call]:** up to about 60 people connected to one meeting, one meeting at a time; agenda of up to about
  50 items. Builder to confirm with OSC.
- **Responsiveness:** moving to the next item updates the main screen and following phones within about two
  seconds on the meeting-room network **[call]**.
- **Devices:** current Chrome, Edge and Safari on Android and iOS phones, tablets and the main-screen PC.
- **Language:** Bahasa Melayu interface; documents in their original language.
- **Audit and integrity:** append-only audit and history; decisions immutable once given.
- **Privacy:** phone numbers and attendee data under PDPA; retention period **[open]**.
- **Accessibility:** keyboard focus and contrast per DESIGN-DIRECTION; text not below 12px; large touch
  targets for the decision buttons.

## 16. Suggested build order

1. **Foundation:** meeting types, meetings, agenda items (linked and standalone), invitations, seats, staff and
   OTP sign-in, safeguards S1–S9 and S12, audit.

   **Core dependencies [call].** Under ADR-212 a core capability is built by the first module that needs it, so
   the eOSC+ work builds the parts it needs, inside `core`, generically: C13 boundary drawing and PostGIS
   geometry, the C9 signature/hash/QR and staff MFA re-prompt, the C8 SMS adapter, and C11 domain events.
   Where one is not ready, these reduced first versions are acceptable:
   - F9 records final decisions before the signed plan exists; signing is added when C9 is ready, and no
     immediate release happens until then.
   - An application with no drawn boundary is shown by its lot geometry from the replicated cadastral layer,
     or as a point.
   - Until the SMS gateway exists, meetings run with staff-account attendees only.
2. **First priorities:** F1 agenda builder, F7 map (with replicated layers and the migration of MJKP points),
   F9 decision panel with signed plan.
3. Then F3, F6, F8, F10, F11 (the meeting experience).
4. Then F12, F13, F14, F4, F5, F2.
5. Legacy migration rehearsals run alongside steps 2–3 so the map shows history from the first live meeting.

The builder may reorder within these steps **[builder]**.

## 17. Open items

**Owner will settle [open]:**

| Item | Needed by |
|---|---|
| Who puts an item forward, and whether a head of department approves it first | Before F1 goes live |
| Which approvals are released to the Owner / PSP immediately after the meeting, and who decides | During development of F9 |
| Legal standing of the chair-signed digital plan (legal unit) | Before the first live signed release |
| Quorum rule per meeting type | Before F9 goes live |
| Full list of additional map layers | Before the first layer replication |
| Who maintains the standard conditions library | Before F9 goes live |
| How many days after the meeting attendee access lasts; retention of attendee data | Before the first live meeting |
| SMS gateway procurement (shared with C8) | Before invitations go live |
| Meeting-room hardware and Wi-Fi | Before the first live meeting |

**Left to the builder [builder]:** route and table prefix (by ADR), table design, map library, layer refresh
method, DWG conversion, screen layouts within the design rules, coordinate system of legacy data, ordering
within the build steps.

## 18. Designer's calls for the owner to check

These were decided by the designer, not the owner. Each can be changed.

1. Standalone agenda items for types without an iNDECS module, linkable later.
2. Meeting types as configuration (Jawatankuasa OSC first, Mesyuarat YDP and others later).
3. Agenda versioning on re-publish; missing-paper flags; paper cut-off.
4. Tangguh items offered automatically for a later meeting.
5. Chair may choose a different decision from the officer's proposal; MFA re-prompt and a summary before the
   final press; corrections as new linked records.
6. C9 e-approval for the signed plan (signature specimen, hash, QR); no stamp for Tolak or Tangguh.
7. Immediate release as an explicit permissioned action.
8. Condition types replace the two legacy "Lulus" variants.
9. Conflict-of-interest members excluded from that item's comments.
10. Department seat allowance remembered per department; personal links tied to a phone.
11. Staff use staff accounts; OTP only for others. Safeguards S1–S12.
12. Esri World Imagery live, MBSP orthophoto as fallback; Street View in a new tab.
13. Layer refresh keeps the previous snapshot.
14. Self-hosted Jitsi Meet; recording off by default.
15. No direct OSC3Plus link in the first release.
16. Legacy status codes mapped as in section 14; records without XY migrated without location.
17. Scale and responsiveness targets in section 15.
18. Chair override handled by the officer revising the proposal on the spot before the chair gives it.
19. STENO notes visible to staff only until minutes are drafted.
20. eOSC+ decision authoritative from day one; the module's own committee-decision entry becomes read-only.
21. eOSC+ work builds the core pieces it needs; reduced first versions listed in section 16.
22. Tabling purposes as a configurable list; an application's current decision is its latest tabling's decision.
23. Legacy `PmnKeputusanYDP` migrated as history only.
