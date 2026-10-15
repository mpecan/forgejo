# Forgejo, specified in Allium

A behavioural specification of Forgejo's full surface, distilled from this
repository's code. It is written so an **alternative implementation** can be
built from it. Compatibility with Forgejo or Gitea is deliberately not a goal:
the spec describes the behaviour a rebuild needs, not Forgejo's accidents.
Importing data from Forgejo and other forges is specified separately as an addon
(`migrations.allium`).

The spec is written in [Allium](https://github.com/juxt/allium). It comes with
**journeys**: executable claims about what particular people can do, walked
against the spec by
[`allium-journey`](https://github.com/mpecan/allium-inspect).

```sh
allium check .                              # structural validation of the whole set
allium analyse <module>.allium              # process-level findings for one module
allium-journey walk . journeys/ --text      # walk every journey
allium-inspect                              # browse it (reads .allium-inspect.toml)
```

## At a glance

| | |
|---|---|
| Modules | 22 `.allium` files, about 39,900 lines |
| Entities | 196, plus 18 external entities and 54 value types |
| Rules | 1,170 |
| Surfaces | 205, for 66 actors |
| Invariants | 214 |
| Config parameters | every `app.ini` setting that changes behaviour, with Forgejo's default |
| Open questions | 202 ([`OPEN-QUESTIONS.md`](OPEN-QUESTIONS.md)) |
| Journeys | 306 in 22 files: **674 of 682 steps hold**; the other 8 are undecided because of walker limitations (see below) |
| Cross-module reactions | 299; every one matches its emitter's trigger name and argument names |
| `allium check .` | 0 errors |


## Modules

| Module | What it owns |
|---|---|
| [`accounts`](accounts.allium) | User accounts as identities: kinds, profile, visibility, status, email addresses, renames and redirects, follows, blocking, deletion and purge |
| [`authentication`](authentication.allium) | Sign-up, sign-in, sessions, password reset, 2FA (TOTP, WebAuthn), access tokens and scopes, Forgejo as an OAuth2/OIDC provider, external auth sources (LDAP, SMTP, PAM, OAuth2/OIDC, reverse proxy) |
| [`keys`](keys.allium) | SSH keys, deploy keys, principals, GPG keys, commit signature verification and trust models, instance signing policy |
| [`organizations`](organizations.allium) | Organizations, the Owners team, teams and per-unit team access, membership and its visibility, team invites |
| [`access`](access.allium) | The authorisation model: access modes, how a user's permission on a repository and unit is derived, restricted users, visibility, the authorisation table for every class of operation, the shared actors |
| [`repositories`](repositories.allium) | Repository lifecycle (create, template, fork, rename, transfer, archive, delete), units and their settings, collaborators, stars, watches, topics, pull and push mirrors, archive downloads |
| [`code`](code.allium) | Git content: transport (HTTP, SSH), pre- and post-receive, branches, tags, branch and tag protection, web file edits, commit statuses, notes, LFS, AGit |
| [`issues`](issues.allium) | Issues and the comment timeline, labels, milestones, assignees, reactions, dependencies, time tracking, pins, locks, subscriptions, attachments, templates, commit-keyword references |
| [`pulls`](pulls.allium) | Pull requests, mergeability, merge styles and merge gates, auto-merge, reviews, review requests, code comments, CODEOWNERS |
| [`projects`](projects.allium) | Project boards for repositories, users and organizations: columns and cards |
| [`releases`](releases.allium) | Releases, drafts, pre-releases, tag reconciliation, release assets |
| [`wiki`](wiki.allium) | The wiki unit: pages, revisions, the wiki git repository |
| [`actions`](actions.allium) | CI: workflow discovery and triggering, runs, jobs and tasks, runners, secrets, variables, artifacts, logs, schedules, automatic tokens |
| [`packages`](packages.allium) | The package registry: 23 package kinds, versions, files and blobs, per-kind policy, repository indexes, cleanup rules |
| [`webhooks`](webhooks.allium) | Webhooks at repository, organization, user and system level, the 13 target kinds and 37 event kinds, delivery, signing, history, cleanup |
| [`notifications`](notifications.allium) | In-app inbox, outbound and inbound mail, activity feeds, dashboards, RSS/Atom, heatmap |
| [`administration`](administration.allium) | Installation, site administration, the 31 scheduled tasks, system notices, announcements, runtime settings, the admin CLI |
| [`quota`](quota.allium) | Quota rules, groups and subjects, and every operation quota gates |
| [`moderation`](moderation.allium) | Abuse reports, preserved copies of reported content, moderator actions |
| [`federation`](federation.allium) | ActivityPub actors, inbox and outbox, HTTP signatures, federated stars and follows, WebFinger, NodeInfo |
| [`discovery`](discovery.allium) | Explore and search (repositories, users, organizations, code, issues), indexers, repository insights, go-get, sitemap, the misc API |
| [`migrations`](migrations.allium) | **Addon.** Importing from other forges, dump and restore, original-author attribution, and the core extension points the addon needs |

[`VOCABULARY.md`](VOCABULARY.md) records which module owns each concept.

## How the modules talk to each other

A module never reaches into another to cause a side effect. It **emits a domain
trigger** (`issues/IssueOpened`, `code/PushReceived`, `pulls/PullRequestMerged`)
and other modules **react** with rules of their own, for example
`when: issues/IssueOpened(issue, mentions)`. Each module's header lists what it
`Emits:` and what it `Reacts to:`. The big consumers are `notifications`,
`webhooks`, `actions`, `discovery` and `access`. `notifications.allium` carries
the full fan-out table.

Scheduled behaviour is owned by `administration`: each of its 31 scheduled tasks
emits a `…Due` trigger, and the module that owns the work reacts to it. That is
also what makes "run now" from the admin panel work.

## Reading a module

Every module follows the same layout:

- **Header**: `Scope`, `Includes`, `Excludes` (each exclusion with its reason),
  `Emits`, `Reacts to`.
- **Entities**, with their lifecycle `transitions`, and `config`, whose defaults
  are Forgejo's own `app.ini` defaults.
- **Rules**: one for every state-changing operation reachable from the web UI,
  the REST API, git or SSH transport, background jobs or internal hooks. Each
  rule's `requires` are the permission checks and validations extracted from the
  code.
- **Invariants.**
- **Surfaces**: one per kind of party and context, showing what that party sees
  (`exposes`) and can do (`provides`). The concrete web routes and API endpoints
  each surface covers are listed in its `@guidance`. That is the endpoint
  inventory for the rebuild, kept out of the normative spec.
- **`open question`s**: places where Forgejo's web UI, API and code disagree, or
  where the code's behaviour looks accidental. Each one needs a decision before
  the rebuild implements that part. [`OPEN-QUESTIONS.md`](OPEN-QUESTIONS.md)
  collects all of them.
- **`-- REQUEST`** lines at the end: things a module needed from another one that
  are not yet fully specified there.

Every entity and rule carries a `-- from: path/to/file.go:LINE` comment pointing at
the Forgejo code it was distilled from.

## Conventions forced by the tooling

The `allium-journey` limitations behind these are filed as [#6–#13](https://github.com/mpecan/allium-inspect/issues?q=is%3Aissue+6..13) on allium-inspect.

These shape how the spec is written. They are not domain decisions.

- **Permission checks are local predicates.** A rule says
  `requires: can_write_code(user, repository)`, not `access/can_write(...)`,
  because the parser reads a qualified call as a division and the journey
  walker cannot stipulate one. Each module's header maps every local predicate to
  the `access` contract function it stands for (`AuthorizationPolicy` in
  `access.allium`). An implementation should route all of them through one
  authorisation service.
- **Triggers that cross modules carry named arguments**
  (`PullRequestMerged(pull_request: pr, merger: user)`), and consumers bind the
  same names. The journey walker binds chained-trigger arguments by name.
- **Visibility is written as `context … where` or `for x in C where …`**, not as
  `when`-guarded `exposes` lines. The walker treats a `when`-guarded exposure as
  never shown, which would make `cannot see` claims vacuous.
- **Actors are documentation; `requires` are enforcement.** The walker does not
  check a surface's `facing` actor, so every permission is restated as a rule
  precondition.
- **Journeys run on default config.** A behaviour that is off by default
  (federation, quota, moderation, LFS) is gated on its surfaces with
  `provides … when config.enabled`. Its rules are walked as if it were on.

## Journeys

[`journeys/`](journeys/) holds one file per module. Each journey names a cast,
sets the starting state, and walks numbered steps:

- `does`: an act through a surface.
- `cannot do`: a refusal that is the point of the step.
- `after`: time passes.
- `then`: an assertion.
- `sees` / `cannot see`: visibility.

Every module has refusals and visibility claims, plus a time step where it has
scheduled behaviour. The verdicts are the checks: **holds** (the spec does
this), **refused**, **unspecified** (the spec has no such thing yet),
**unexposed**, and **undecided** (the tool could not tell). The undecided steps
that remain are listed below with their reasons.

### Steps that do not hold, and why

None of these is a gap in the spec. Each is something `allium-journey` (0.1.1)
cannot decide.

| Journey | Why |
|---|---|
| actions: *A Dispatched Workflow Runs To Success* (step 5), *A Running Run Cannot Be Rerun Until It Finishes* (3), *A Silent Task Becomes A Zombie* (3), *A Job Nobody Picks Up Is Abandoned* (2); pulls: *Auto Merge Waits For Required Checks* (3) | During an `after` step the walker treats an entity collection as open-world. `webhooks/FanOutHookEvent` loops `for w in Webhooks where …`, which yields an unknown member even when the journey casts no webhook, so the downstream `QueueDeliveryForMatchedEvent` cannot be decided "while the clock moved". This reproduces in a 20-line spec. The same fan-out holds in non-timed steps. ([#6](https://github.com/mpecan/allium-inspect/issues/6)) |
| webhooks: *A Week Later The History Is Cleaned Up* | The same open-world behaviour, applied to the timeout timer of a delivery that the cleanup just removed ([#6](https://github.com/mpecan/allium-inspect/issues/6)) |
| federation: *A Repository Admin Follows A Remote Repository* (2) | The walker cannot evaluate `.all(…)` over a collection ([#7](https://github.com/mpecan/allium-inspect/issues/7)) |
| releases: *A Maintainer Links An External Asset* (1) | The walker does not bind an `issues/Attachment` created by a releases rule ([#9](https://github.com/mpecan/allium-inspect/issues/9)) |


## How this was produced

The Allium `distill` process, run as a fan-out:

1. The codebase was mapped into 22 modules, distilled by 20 agents (releases and wiki shared one, as did quota and moderation), and
   [`VOCABULARY.md`](VOCABULARY.md) fixed module ownership and shared names.
2. One agent per context read that context's models, services and web/API
   routers. It wrote the module and its journeys, and validated both with
   `allium check`, `allium analyse` and `allium-journey walk`.
3. An integration pass audited every cross-module trigger reaction against its
   emitter (name, arity, argument names). It unified duplicated concepts and
   actors, and wired every scheduled task to its owner.
