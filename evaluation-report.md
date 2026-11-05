# rAccess evaluation: status, quality and features

2026-11-05 · Timo Ruohomäki

## Summary

rAccess is not ready to be the security layer for a PowerBI replacement. It is a young (v0.1.1.3), single-maintainer package that has had no upstream commits for a year, and it controls *which UI features a user sees*, not *which data rows a user can read*.

- **Status:** published on CRAN 2025-11-05, upstream dormant since 2025-11-06, 2 open issues unanswered for 11+ months. Your fork is identical to upstream HEAD.
- **Quality:** readable but early-stage R code: large monolithic files, heavy duplication, little input validation or error handling, one confirmed bug in a public function.
- **Security:** authentication is fully delegated to Posit Connect (good), but admin write actions are not re-checked on the server, the documented example identifies every user as the same person, and a user without access can stop the app for everyone.
- **Features:** a simple feature-flag style permission matrix (panels × units × users) stored as a versioned `pins` data frame, with an admin UI and an audit trail. No groups/roles, no row-level filtering, no database backend.

Verdict: usable as inspiration or for a low-risk internal app on Posit Connect. For a PowerBI replacement with row-level security you would need to build the data-filtering layer yourself, and either harden rAccess or write a small purpose-built module backed by PostgreSQL.

## Project status and maturity

The project is early-stage and currently unmaintained in practice: it went from first commit to CRAN in 23 days and has had no activity since.

| Signal | Finding |
| --- | --- |
| Owner | [johnsonandjohnson/rAccess](https://github.com/johnsonandjohnson/rAccess), funded by J&J Innovative Medicine, Apache-2.0 |
| Version | 0.1.1.3 on [CRAN](https://cloud.r-project.org/web/packages/rAccess/index.html), published 2025-11-05 |
| History | 21 commits, 2025-10-13 to 2025-11-05; 18 of 21 by one author (Peyman Eshghi, maintainer) |
| Last upstream push | 2025-11-06, about 12 months ago |
| Community | 18 stars, 2 forks, 2 merged PRs (both by the maintainer) |
| Open issues | #3 "Improve default parameters" (maintainer, 2025-11-07) and #4 "Remove here::here() from constructor" (external, 2025-12-19), both with no reply |
| Your fork | rAccess-custom is at upstream HEAD (276a500), no local changes |
| CI | R-CMD-check on 5 OS/R combos, test-coverage and pkgdown workflows; coverage badge shows 88% |

The access-matrix test fixtures date from 2023, so the code was probably used internally at J&J before it was open-sourced. That suggests real-world use, but the public project shows no sign of ongoing maintenance.

**Dependencies.** 18 hard imports, mostly well-maintained Posit/r-lib packages (shiny, pins, DT, dplyr, R6). Two points to note: `httr` is superseded by `httr2`, and several imports (`config`, `here`, `shinyalert`, `shinyWidgets`) are used for small conveniences, which widens the dependency surface of a security component.

## Code quality and testing

The code works for its happy path but is prototype-grade: about 2.100 lines in three files, with copy-pasted logic and almost no defensive error handling. I reviewed it by reading; I did not run the tests (no R in this environment).

| Area | Observation | Where |
| --- | --- | --- |
| Structure | Three files hold everything; `module_sub_iam.R` is 1.065 lines and `functions.R` 945 | `R/` |
| Duplication | The "list pins, strip prefix, read `<app>_access_board`" block is repeated about 12 times; add/edit handlers are duplicated per access mode | `R/functions.R`, `R/module_sub_iam.R` |
| Confirmed bug | `s3_pinboard()` passes the access key as the secret key, so it cannot authenticate | `R/functions.R:942` |
| Bug | `ifelse(is.null(pin_name), NULL, pin_name)` errors when `pin_name` is NULL | `R/functions.R:298` |
| Typing | Permissions are stored as the strings "TRUE"/"FALSE" in some paths and logicals in others; `is_admin()` compares to "TRUE", `no_admin()` to `TRUE` | `R/functions.R:433`, `:458` |
| Stale state | `check_access()` and `is_admin()` use the pin list cached at construction, other methods re-list | `R/functions.R:358`, `:413` |
| Error handling | No `tryCatch` anywhere; a network error from S3, Connect or the user API surfaces as a crashed session | whole package |
| Side effects | Every session can rewrite the access board on startup when the configured units differ from the stored ones | `R/module_sub_iam.R:76-110` |
| Tests | 47 expectations across 9 files; the 556-line app test mostly drives inputs without asserting on the result, so 88% coverage overstates verification | `tests/testthat/` |
| Docs | Good README, pkgdown site and a tutorial vignette with screenshots | `README.md`, `vignettes/` |

## Security findings

The design choice to delegate login to Posit Connect is sound, but enforcement inside the app is mostly UI hiding, and three findings would block production use as-is. All are from code reading and were not exploited against a running app.

| Severity | Finding | Where |
| --- | --- | --- |
| High | **Admin actions are not re-authorised on the server.** The tutorial calls `module_iam_server()` for every user and only hides the admin tab. The add/edit/delete handlers write to the access board without calling `is_admin()`, so a non-admin who sends Shiny inputs from the browser console (`Shiny.setInputValue`) can very likely grant themselves access, including ADMIN. | `R/module_iam.R:45-50`, `R/module_sub_iam.R:406-520`, `vignettes/Tutorial.Rmd:352-364` |
| High | **The documented user lookup is broken.** `ifelse(!exists("session$user"), "UserID", session$user)` always returns "UserID", because `exists()` looks for a variable literally named `session$user`. Copying the tutorial makes every visitor the same user. | `vignettes/Tutorial.Rmd:352` |
| High | **One denied user stops the app for everyone.** With `secure_mode`, a user with no access triggers `Sys.sleep(10)` and then `stopApp()`, which blocks and then ends the shared R process for all sessions on it. | `R/functions.R:542-556` |
| Medium | **Fail-open defaults.** `secure_mode` defaults to FALSE, and while the board has no admin, every visitor sees the admin panel and can make themselves admin. | `R/functions.R:129`, `R/module_iam.R:80-83` |
| Medium | **Race conditions.** Each change is read-modify-write of one whole data frame; two admins editing at once silently lose one change. | `R/module_sub_iam.R` |
| Medium | **Secrets in config.** S3 keys sit in the YAML file and are copied into process environment variables. The YAML is read with `eval.expr = TRUE`, so the config file can run R code. | `inst/config/rAccess.yml`, `R/functions.R:185`, `:619` |
| Low | Audit trail records "Edited by \<user>" per version but not what changed; history is only as durable as the pin board's versioning. | `R/module_sub_iam.R` |

## Features

rAccess is an in-app permission matrix with an admin UI: it answers "may user X use feature Y?" and leaves enforcement to the app developer.

**Access model.** Two levels: *access panels* (groups of features, shown as admin tabs) contain *access units* (individual features, data views or buttons). A mandatory ADMIN panel controls who may administer each panel. A special user "Everyone" grants a unit to all users. `access_mode = "single unit"` limits a user to one unit per panel. There are no groups or roles, no deny rules and no inheritance; every grant is user × unit.

**Identity.** No login of its own. In production the user ID is `session$user`, provided by Posit Connect (or Shiny Server Pro / ShinyProxy that sets the same field). The user picker searches a static `user_df` and, on Connect, the Connect users API.

**Storage.** One data frame per app, stored with the `pins` package on a local folder, AWS S3 or Posit Connect. Pin versioning gives a history of every change and `get_accesshistory()` / `get_accesslist()` return the matrix at a date or range. There is no PostgreSQL, MongoDB or SQLite backend, though any `pins` board could be passed in.

**Shiny integration.**

1. Create `rAccess$new(user = session$user, config = "rAccess.yml")` per session.
2. Add `module_iam_ui()` / `module_iam_server()` for the admin screen.
3. Call `get_user_accesslist()` and show or hide tabs and outputs yourself with `showTab()`/`hideTab()` or `req()`.

**Admin UI.** DT-based tables per panel, add/edit/delete users, switch or dropdown display, CSV download of the access list.

**Config.** Everything can be declared in a YAML file, including the panel structure and an optional data path per unit, which hints at data-level use but is only passed through to the app.

## Gap analysis vs PowerBI

The largest gap is row-level security: PowerBI filters the data itself, while rAccess only toggles features, so the filtering layer would be new work either way.

| PowerBI capability | rAccess | Gap |
| --- | --- | --- |
| Authentication (Entra ID / SSO) | Delegated to Posit Connect or another proxy | None, if you host on Connect or ShinyProxy with OIDC |
| Row-level security (DAX role filters on tables) | Not provided | Must be built: filter queries per user, ideally enforced in PostgreSQL with RLS policies or views |
| Object-level / column security | Feature toggles per unit approximate it, enforced only by the app's own `req()` / hide calls | Partial; enforcement is the developer's job |
| Roles mapped to AD groups | Individual users only, plus "Everyone" | No groups, no directory sync |
| Dynamic RLS (`USERPRINCIPALNAME()` lookup tables) | `session$user` is available; mapping table would be yours | Must be built |
| Admin UI for permissions | Yes, per app | Comparable for small user counts |
| Audit of permission changes | Versioned pins with editor name | Basic; no diff, no tamper protection |
| Central governance across many reports | One board per app | No cross-app view or reuse of roles |
| Scale / concurrency | Whole-table rewrites, no locking | Fine for tens of users and rare edits, risky beyond |

## Recommendation and next steps

Do not adopt rAccess unmodified as the security layer. Treat it as a reference design and decide between hardening the fork and writing a small, PostgreSQL-backed access module.

- Put identity in front of Shiny: Posit Connect, or ShinyProxy with OIDC against your identity provider, so `session$user` (and ideally groups) comes from a trusted source.
- Enforce row-level security in the database: PostgreSQL RLS policies or per-role views keyed on the user or group, so a Shiny bug cannot leak rows.
- If you keep rAccess, fix at minimum: server-side `is_admin()` checks in every write handler, remove `Sys.sleep()`/`stopApp()`, correct the tutorial's user lookup, fix `s3_pinboard()`, default `secure_mode` to TRUE, and move storage to a database table with transactions.
- Add groups or roles mapped to directory groups before rolling out beyond a handful of users.

Open question: which hosting will you use for Shiny (Posit Connect, ShinyProxy, or self-hosted Shiny Server)? It decides how identity and groups reach the app.

## Sources

- [rAccess on CRAN](https://cloud.r-project.org/web/packages/rAccess/index.html)
- [johnsonandjohnson/rAccess on GitHub](https://github.com/johnsonandjohnson/rAccess) (activity, issues and PRs read via the GitHub search API)
- Source review of [timoruohomaki/rAccess-custom](https://github.com/timoruohomaki/rAccess-custom) at commit 276a500
