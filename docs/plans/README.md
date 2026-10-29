# eOSC+ build plans

The approved design (`../README.md`, v1.1, 2026-10-29) covers several subsystems. It is built as a series of
plans. Each plan ends in working, tested software, and each is written in detail only when the previous one
has landed, so it can use the real code.

| # | Plan | Delivers | Design sections | Status |
|---|---|---|---|---|
| 1 | [Foundation and agenda builder](2026-10-29-01-foundation-and-agenda.md) | Meetings, members, agenda items (linked and standalone, repeated tabling), optional head-of-department endorsement, papers, agenda publishing with versions, staff-only access, `/osc` screens | 3, 4, 5, 6, 8.1, F1 | Built (branch feat/eosc-foundation, pending review/merge) |
| 2 | Map | PostGIS geometry on applications and items (C13), boundary drawing at registration, replicated reference layers with snapshot refresh, scanned layout imagery, Esri imagery and orthophoto fallback, MJKP points and images migrated, map screen | 8.2, 9.3, 14 | Next |
| 3 | Decision panel | Comments, STENO notes, attendance, quorum, conflict of interest, proposed decision, officer-then-chair sequence, four decisions, immutability and corrections, Tangguh carry-over, minimal F12 (decision written to the application; module entry read-only) | 8.3, F10, part of F12 | |
| 4 | Attendee access | Personal and department invitations, seats and extra-seat approval, phone OTP through the C8 SMS adapter, device binding, expiry, revocation, watermark, download rules, safeguards S1–S12 | 10 | |
| 5 | Signed plan and release | C9 signature specimen, staff MFA re-prompt, hash and QR, verify page, stamping on Lulus / Lulus dengan Syarat, permissioned immediate release | 8.3 (5, 6) | |
| 6 | Meeting experience | F3 pre-read, F6 follow the presenter, F8 plan viewer, F10 timer, F11 Jitsi video call | 9.1, 9.2, 11 | |
| 7 | After the meeting | F12 full workflow events (C11) and letters, F13 minutes, F14 dashboard, F4 ulasan tracker, F5 offline pack, F2 AI draft paper | 9.1, 9.4 | |
| 8 | Legacy history | Full MJKP migration: 7,694 records, attachments with checksums, rehearsal and reconciliation | 14 | Rehearsals run alongside plans 2–3 |

Plans 1–3 are the owner's first priorities (F1, F7, F9). Plan 4 is needed before any external attendee joins;
until then meetings run with staff accounts only (design section 16).
