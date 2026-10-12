# Roadmap and deliberate omissions

## Done, in order of appearance

| Version | Theme |
|---|---|
| v2–v5 | tickets → assets → employees → reports |
| v6 | visual redesign |
| v7 | CSV in/out, in-app help, member management |
| v8 | e-mail one-time-code login, login-user management, auth log |
| v9 | department and site masters, admin tab, admin-only enforcement, session/admin guard split |
| v10 | idle logout, admin guide, sample-data buttons |
| v11.0–11.8 | team tab (handover threads, schedule), daily backup + weekly mail, assignment and SLA notifications, cross-entity links, rack layout, site inventory via SNMP, SLA config sheet |
| v11.9–11.18 | codename versioning, systematic security hardening (session checks, trigger guards, owner-gated migrations, rate limits, formula injection), in-app back navigation, pre-production audit |
| v11.19 | members unified with login users |
| v12.0 | integrity audit: asset users and login users linked to the employee directory, one vocabulary, consistent counts |

## Open

- **Asset site columns**: decide whether "belongs to" and "located at" are one concept or
  two, then remove or rename.
- **Delete actions**: either expose asset/employee deletion in the UI with reference
  counts, or close the server functions. Today deletion happens in the sheet directly,
  without the orphan check the code already implements.
- **Directory sync and "on leave"**: the export tool currently marks every enabled account
  as active, overwriting a manually set "on leave" status on each import.
- **Response-time SLA**: targets are configured and displayed but only resolution time is
  monitored. Either monitor both or stop showing the first.
- **Quick capture requester**: free text only; add the directory suggester used by the
  full form.

## Deliberately not planned

- **Remote monitoring of sites** (an earlier "v12" idea). Management protocols do not
  traverse the site firewalls, and opening them is not within the team's remit. Replaced by
  the on-demand inventory scanner run on site.
- **Per-record permissions, approval workflows, SLA pause/clock rules.** The team is three
  or four people; the cost of modelling these exceeds their value at this size.
- **A framework or a build step for the app itself.** The maintainer must be able to read
  and paste the source. The tooling around the app may use Node; the app may not.
