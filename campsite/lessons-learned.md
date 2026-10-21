# Lessons Learned: Laravel Hackathon and AI-DLC Work on CampSite

This page collects reusable guidance from the Laravel migration hackathon and from recent AI-DLC units. Each lesson cites its source so it can be re-checked. Companion to [CLAUDE.md](../CLAUDE.md).

---

## 1. Status: what is and isn't Laravel

- **Production CampSite is the legacy PHP application.** Nothing Laravel is merged into `develop` or `master` (checked 2026-10-21 with `git merge-base --is-ancestor`), and no deployment path exists for it.
- **The Laravel proof of concept lives only on `origin/feature/laravel-campsite`.**
  - **Layout:** a separate Laravel 12 app in `laravel-campsite/`, with its own Docker stack (8 services).
  - **Databases:** it shares the legacy databases (`cf4_master` + camp DBs) without schema changes. Its own sessions, cache and jobs live in a separate `campsite_app` DB.
  - **Login:** it has its own login, not shared with legacy sessions; an SSO bridge is backlog.
  - **Tenancy:** ADR-1 defines it (`aidlc-docs/inception/application-design/architecture-decision-records.md`). Subdomain → `camps.initials` → node lookup → a `tenant` connection, with models split into `Master` and `Tenant`.
  - **Docs:** they disagree on ports and routing. Treat that branch's docs as a proof of concept, not as a description of CampSite.
- **The other hackathon branches are legacy PHP, not Laravel:** `hackathon/ask-campsite` (AI report builder) and `hackathon/bunk-buddy-AI`.
- **The NOW-28 / NOW-29 "modern profile" work is also legacy PHP** (`_x` pages), on feature branches.

When asked for CampSite code, generate legacy PHP that follows [CLAUDE.md](../CLAUDE.md). Laravel code belongs only in work explicitly scoped to `laravel-campsite/`.

---

## 2. Reusable lessons

Each lesson starts with what to do, then its evidence. Sources are paths on `origin/feature/laravel-campsite` unless noted.

### Context before code

- **Read the legacy source before porting or changing a behavior; the schema doesn't capture the business rules.** Write paths must have their rules re-verified rather than "assumed to be covered by the DB schema alone" (`aidlc-docs/inception/application-design/legacy-laravel-coexistence.md`).
- **Sample-based discovery misses things.** A validation pass over the first discovery added 8 new risks (#22–29). A headline finding (87% of files lacking a permission check) turned out to be "mostly a false negative" for `admin/`, which is gated by auto-prepend, but "a real, standing gap" for `ajax/` (`aidlc-docs/inception/reverse-engineering/matrix-validation-notes.md`). Confirm the mechanism before drawing per-file conclusions.

### Preserve behavior and compatibility

- **Keep legacy quirks unless the change is explicitly scoped.** The Shop port has separate sections for "Legacy quirks deliberately preserved" and "Deliberate differences" (`aidlc-docs/construction/shop-module/code/shop-module-summary.md`). Do the same: list what you keep and what you change, and why.
- **Don't flip risky security settings as a side effect.** SAML `strict` was "deliberately NOT changed" because flipping it could cause "a production incident" (`aidlc-docs/legacy-vs-laravel-audit.md`).

### Tenancy and isolation

- **Every new model must target the right database** (master vs tenant), or "it will silently query the wrong database" (`aidlc-docs/inception/application-design/laravel-target-architecture.md`). In legacy code the equivalent is: make sure the global `$db` is the right DB before creating objects.
- **Fail closed on unknown tenants.** Legacy `isValidCamp()` returns 404 for an unknown subdomain. The POC's local configuration has a default dev camp (`CAMPSITE_DEV_CAMP_INITIALS`); a default-tenant fallback like that must never reach a deployed environment.

### Authorization and ownership

- **Check permissions at the mutation endpoint, not only in the UI.** Legacy checks are "mostly UI-gating rather than enforced at the mutation endpoint", and admin has "no CSRF protection anywhere" (`aidlc-docs/legacy-vs-laravel-audit.md`).
- **A flow that is enrollable is not necessarily enforced.** Lockout and 2FA defects in the port: 2FA "was enrollable but never enforced", and the lockout counter was "checked but never persisted" (same audit). Test the enforcement path, not just the setup path.

### Configuration assumptions

- **Production config "lives only on running instances".** Local `config/` says nothing about production values (same audit; `AfterInstall.sh` deletes the repo copy on deploy).
- **`campmanagement.com` is the live production domain.** Without a hosts entry, a local test signs into production (same audit, "The hosts entry is not optional").
- **htaccess differs per environment.** A missing `QSA` flag meant "production would have kept discarding the query string" (NOW-28, commit `ddd35b039c`). Check `.htaccess-production` whenever you change `.htaccess-development`.

### Test the whole workflow

- **A green suite can be meaningless.** In NOW-29, a test checked a copy of the function, so "the test could not have failed" (`aidlc-docs/NOW-29-process-retrospective.md` on `origin/feature/NOW-29-view-as-parent`). Test the real code path; mirror only what cannot be loaded.
- **Browser passes find what other tools miss.** In NOW-29, "curl and unit tests both passed" while the guard failed silently on parent pages (same retrospective). In NOW-28, axe reported zero violations while Lighthouse caught a broken skip link, and styling was "correct by luck" because a Tailwind `@source` pointed at deleted files (commit `339abfe499`).
- **A failing check is often a wrong test.** In this project "a failing check has more often meant a wrong test than wrong code", which can only be told apart by going back to the legacy source (`aidlc-docs/construction/build-and-test/build-and-test-summary.md`).
- **Test across subdomains.** For cross-subdomain flows such as OAuth, test on at least two subdomains (`test/CAMP-6710-takeaways.md`, on `develop`).

### Findings: separate verified defects from recommendations

- **Assess every finding.** "A finding raised is not a finding assessed": attach a reachability verdict with the evidence (NOW-29 retrospective).
- **Give every "cannot" a subject:** who cannot, and why (same).
- **Track assumptions.** A standing assumption needs an owner and a stage by which it is tested or downgraded (same).
- **Label speed claims honestly.** The "278x faster" forms result was not a framework benchmark, and legacy was "Not measured" (`aidlc-docs/construction/forms-module/code/forms-module-summary.md`). State what was and was not measured.

### Incremental change with explicit criteria

- **Turn repeated defects into a count.** When the same defect repeats N times, make the test "a count that must reach zero" (NOW-29 retrospective).
- **Never run two implementations of a cron at once,** and roll schema changes out per camp DB before shipping dependent code (coexistence doc; `sql/migrations/` header on `develop`).

---

## 3. How to apply this in AI-DLC stages

| Stage                    | Apply                                                                                                                                                         |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Inception / requirements | State the tenant scope (camp, family, admin), the permission strings involved, and which UI layer is targeted. Read the legacy files end-to-end, not by grep. |
| Design                   | List preserved quirks vs intended differences. Say how writes stay consistent without transactions. Name the DB (master or camp) for every table touched.     |
| Code generation          | Follow the [CLAUDE.md](../CLAUDE.md) rules; add endpoint-level permission, ownership, and CSRF checks where state changes.                                    |
| Build & test             | Run inside the `web` container and exercise the real workflow in a browser on your subdomain. Report skipped tests, not just passes.                          |
| Review                   | Classify each finding as a **verified defect** (with reproduction or evidence) or a **recommendation**. Never mix them in one list without labels.            |
