# Integrity: when one thing is stored as two

The most valuable lesson from two years of running this app is not about Apps Script. It
is that **the concepts users have in their heads and the records the system keeps drift
apart**, quietly, whenever the same real-world thing is represented twice. This document
lists every instance found in a deliberate audit (builds 11.19 and 12.0) and what was done.
The pattern is general; the fixes are examples.

## The pattern

Two records that "obviously" describe the same thing, linked only by a string that both
happen to contain. Editing one does not touch the other. Nothing breaks on the day of the
edit. Weeks later a report is wrong, a notification is not sent, or a dropdown is empty,
and the cause is three steps removed.

Three questions catch it:

1. For every entity, where else is its name or id written?
2. When this entity is renamed, deactivated or deleted, which of those places change?
3. If none of them do, is the copy a *snapshot* (history, fine) or a *reference* (must
   follow)?

## Cases found

### IT members vs. login users — HIGH, fixed in 11.19

Two lists: a "member" list in the config sheet (used for the assignee dropdown) and the
login-user sheet (used for authentication). Linked by display-name equality. Add a login
user and forget the member list → the person can log in but cannot be assigned anything
and "my tickets" is always empty. Rename on one side → that person's notifications stop,
because the mail address is looked up by name.

**Fix:** one list. A login user *is* a member when a member flag is on. Renaming cascades
to every sheet that stores the name. Names are unique. The old list is migrated once; names
that do not match any user are shown as "unlinked legacy members" for the administrator to
link or delete.

### Login users vs. the employee directory — HIGH, fixed in 12.0

The same four people exist as login users ("Smith") and as employees ("Jane Smith,
E0001"). No link. Reports show the member and the employee as different people when an IT
member is also a requester; the directory marks someone as left while their login stays
active.

**Fix:** an optional employee-id column on the login user, validated to exist and to be
unique. It enables a "me" button on the ticket form and a warning badge when the linked
employee has left.

### Asset user vs. the employee directory — HIGH, fixed in 12.0

Assets stored the user's name, id, department and site as free text copied at save time.
Nothing refreshed them. With directory sync from Active Directory the directory moved
forward and every asset stayed where it was; assets stayed "in use" by people who had left.

**Fix:** the employee id is the truth. Name, department and site are derived on save and
on import (an unknown id is an error); saving an employee rewrites the derived columns on
that employee's assets. Display prefers the directory's current name.

### Ticket requester — MEDIUM, partly fixed

Requester is stored as free text (snapshot) and, when picked from the directory, as an
id. Lists resolved the id; search, mail bodies and CSV used the text, so a renamed employee
could not be found by their new name.

**Fix:** search and mail resolve the id too. The text remains as the snapshot it was
always meant to be. Quick-capture tickets (no id) remain a known gap.

### Two site columns on one asset — MEDIUM, open

"Belongs to site" and "site name" exist side by side; only one is used by every screen.
Whether they mean different things (user's site vs. physical location) is a business
question, so the fix waits for the answer.

### Rename cascades with gaps — LOW, fixed

Site rename reached employees and assets but not racks or staging rows; member rename
reached tickets but not the asset "updated by" column. Each was a one-line addition once
the inventory above existed.

### Vocabulary defined in several places — MEDIUM, fixed

Seed lists for departments and sites existed in three variants (setup defaults, demo data,
an external sync tool), so a standard "set up, then load demo data" produced a mixed
master. Dropdowns on the asset form were built only from values already present in assets,
so a newly added site could not be chosen. The sample asset set used a vocabulary that
matched nothing else.

**Fix:** one constant for each list, referenced everywhere; dropdowns merge the master
sheets; sample data regenerated from one consistent source.

### Numbers that should agree but did not — LOW, fixed

- "After-hours this month" counted a flag; its "see all" link filtered by ticket type;
  the manual said both. Now one predicate, used by the card, the link and the form.
- The dashboard "overdue" tile excluded on-hold tickets; the list tab and the digest
  included them.
- Closed-at was stamped with browser time, created-at with server time.
- The login-user count on the system page was 0 for non-admins because the list was only
  fetched on the admin screen.
- A "recommended trigger time" appeared with two different values on one screen.

### A regression hidden by a safety feature — HIGH, fixed

A key allow-list added to stop arbitrary writes to the config sheet omitted the global
notification switch. For two builds the switch could not be turned on from the UI, so no
notification mail went out. Caught only by a line-by-line consistency audit. Lesson:
allow-lists need a test that exercises every legitimate key.

## How the audit was run

Three passes over the whole code base, each with a single question:

1. *Vocabulary*: every enumerated value (statuses, types, priorities, device kinds,
   departments, sites) — where is it defined, where is it compared, do the sets match?
2. *References*: every id or name that appears in more than one sheet — which saves and
   deletes touch each copy?
3. *Semantics*: every number shown in two places, every rule described in help text or
   the manual — is it computed from the same inputs by the same predicate?

Each finding was listed with the consequence a user would see, then approved or deferred
one by one before anything was changed.
