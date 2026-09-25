# Personal calendar

A played person's **personal calendar** — dated reminders they keep and
read on demand, on the aether implant. Seeded thin by the
clinical-medicine build as the honest home for *"come back in two weeks"*
(a clinician's PROGNOSIS, communicated — not something a wound reveals);
the full design lives in `docs/slates/builds/personal-calendar-slate.md`.

⚠ **Distinct from the world calendar** (`lib/time/DefaultCalendar`), which
is the shared game date. This is your own diary.

## The mixin

**`CalendarMixin`** (`lib/calendar/Calendar.ts`, `Mixins.Calendar`,
`MixinApi.isCalendarKeeping`) is composed on **`Avatar`** (a played person
keeps a calendar; an NPC does not), inside `PersistableMixin` so the
entries ride the Avatar snapshot. An entry is `{ id, whenGameS, label,
source, addedAtS, firedAtS? }`.

- **`addCalendarEntry({label, whenGameS, source})`** — the author seam,
  `@Final @Unshadowable` and **UNGATED** (the `creditDeed` precedent). The
  writer set is every acting controller (medicine writes the aftercare
  date; contract/employment/banking later); write-authorisation is the
  calendar slate's open question, deferred.
- `getCalendarEntries()`, `removeCalendarEntry(id)`,
  `dueCalendarEntries(nowS)`, `rescheduleCalendarPing()`.

## The ping

`rescheduleCalendarPing()` cancels the prior handle and books a one-shot
`WorldClockApi.at(nextDue, cb, { host: this })`; re-armed in
`Avatar.postRegister` after `materialize` and after every add. The
callback stamps `firedAtS` and pushes *"Your calendar: <label>."* to self
(topic `session.notice`). ⭐ The ping is **timeliness, never validity** —
`calendar` lists overdue entries regardless of whether the one-shot fired
(an entry due while offline pings once at the next re-arm,
deferred-not-skipped); correctness never depends on it.

## The surface

**`CalendarUpdate`** (`platform/idea/CalendarUpdate.ts` =
`CalendarAppMixin(AetherHostedMixin(Idea))`) is the hosted app on the
implant, cloned in `Avatar.installDefaultLoadout` beside comms / forums /
wallet; it reads its host's entries through the hosted-update
`getOperator` seam. Its `commandContributions.self` flows the **`calendar`**
verb onto the host. `CalendarController` lists overdue-first, rendered
with `DefaultCalendar.formatDate` — never a raw second count.

⚠ **No `CalendarApi` / `CalendarLogic`** — `CalendarApi.add(player, …)`
would be exactly the `XApi.verb(host, …)` shape `lint:object-verbs` holds
at zero; an entry write belongs to one object and there is nothing to
orchestrate. The mixin method is the seam every future writer calls.

## The medicine consumer

`suture` and `operate` (`trade-medicine`) write the return date when a
`competent+` treater acts: `patient.addCalendarEntry({ label, whenGameS,
source: 'medicine' })`, narrowed by `isCalendarKeeping` (a no-op on an NPC
patient). ⚠ **`assess` stays present-state only** — never a future date;
the calendar owns the *scheduled* reminder, the notify alarm owns *sensed*
changes.

## History

Shipped by the clinical-medicine build (W4). See
`docs/plans/clinical-medicine-plan.md` D12 and
`docs/slates/builds/personal-calendar-slate.md`.
