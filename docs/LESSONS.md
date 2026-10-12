# Platform lessons (Apps Script + Sheets + HtmlService)

Things that cost real time. Each one is now a rule in the project notes.

## Sheets types what you write

`setValue('TRUE')` stores a boolean. `setValue('2026-01-01')` stores a date.
`setValue('=1+1')` stores a formula. `getValues()` returns `true`, a `Date`, and `2`.
Comparisons like `=== 'TRUE'` silently fail after the first round-trip, and a date written
as text comes back shifted by the sheet's time zone.

Rule: normalise on read (`cellToStr_`) and neutralise on write (`toCell_`). Never compare a
cell against a string literal without normalising.

## `_`-suffixed functions are hidden from the editor too

Suffixing a function with `_` keeps it out of `google.script.run` — but also out of the
editor's "run" dropdown. A migration that was suffixed for safety became unrunnable. The
project notes had asserted the opposite for a while.

Rule: private body `xxx_()` plus a public `runXxx()` that checks the caller is the script
owner acting from the editor (`Session.getActiveUser() === Session.getEffectiveUser()`).

## Triggers must authenticate themselves

A time-driven function is public by necessity. Verify `e.triggerUid` against
`ScriptApp.getProjectTriggers()` and return if it does not match. Mail-sending jobs are the
critical ones: the daily mail quota is shared with the login codes.

## Mail quota is the availability budget

Because login depends on mail, anything that can send mail on request is a denial-of-
service vector. Global rate limits on code issuance are evaluated **before** checking
whether the address is registered. Test-mail buttons have cooldowns. Digest jobs skip users
with nothing to report.

## New Google services need a new consent

Adding a call to a service the script has not used before (Drive export, Gmail, Lock) means
the next deployment silently fails for triggers until the owner re-authorises. The release
checklist runs `setupSheet()` from the editor after every deployment specifically to surface
the consent dialog.

## The sandbox iframe and navigation

`history.pushState` is not available. Use `location.hash` and a `hashchange` listener;
use `location.replace` for the first entry; ignore hash changes you caused yourself
(counter); reset the history on login and logout. Restore by re-opening entities, not by
toggling classes.

## Time zones are three different things

The script project's time zone, the spreadsheet's time zone and the user's browser all
differ in principle. Stamp every timestamp on the server with `Utilities.formatDate(…,
TIMEZONE, …)`; parse stored strings with `Utilities.parseDate` in the same zone; derive
weekday-only logic from the formatted date, not from `Date.getDay()`. Do not let the client
fill timestamps.

## One token, many tabs

The session token lives in `localStorage`, so every tab shares it. An idle-logout in one
tab that invalidates the server token signs out the tab the user is working in. Share a
"last activity" timestamp and only invalidate when no tab has been active.

## A `getValues()` row is a reference

In a loop that both reads and writes rows, keep the in-memory array updated after each
`appendRow`/`setValues`, or a duplicate id later in the same import is appended instead of
merged. Likewise, when a mock server mutates the same object the client holds, "before"
and "after" compare equal — snapshot primitives before the call.

## Allow-lists need coverage tests

A key allow-list that is missing one legitimate key fails *silently* for exactly the
feature that key controls. Every allow-list gets a test that walks the UI's actual keys.

## Idempotent setup, self-healing schema

`setupSheet()` creates what is missing and never touches what exists, so it can be run
after every deployment. Column additions also repair themselves on first read, because
"run the setup function" is the step most often forgotten.

## Build the preview without a server

The four files cannot be opened locally — `google.script.run` does not exist outside Apps
Script. A small script concatenates them into one HTML file and injects a mock
`google.script.run` backed by in-memory sample data. That file is what headless-browser
tests and manual screenshots run against. Keeping the mock honest (same validation rules
as the server) is part of every change; several real bugs were found because the mock
disagreed with the server.
