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
| Modules | 22 `.allium` files, about 42,800 lines |
| Entities | 200, plus 18 external entities and 54 value types |
| Rules | 1,224 |
| Surfaces | 216, for 68 actors |
| Invariants | 225 |
| Config parameters | every `app.ini` setting that changes behaviour, with Forgejo's default |
| Open questions | none left: all 201 were decided ([`DECISIONS.md`](DECISIONS.md)) |
| Journeys | 452 in 22 files: **all 1,025 steps hold** with allium-journey 0.1.4, with no walker diagnostics |
| Cross-module reactions | 320; every one matches its emitter's trigger name and argument names |
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
- **`-- decided:`** comments: where Forgejo's web UI, API and code disagreed, or
  its behaviour looked accidental, the distillation raised an `open question`.
  All of them have since been decided; [`DECISIONS.md`](DECISIONS.md) records
  each decision and the spec carries it as a `-- decided:` comment beside the
  behaviour it shaped.
- **`-- REQUEST`** lines at the end: things a module needed from another one that
  are not yet fully specified there.

Every entity and rule carries a `-- from: path/to/file.go:LINE` comment pointing at
the Forgejo code it was distilled from.

## Conventions forced by the tooling

These shape how the spec is written. They are not domain decisions. They are
written for allium-journey 0.1.4. Releases 0.1.2–0.1.4 closed allium-inspect
[#6](https://github.com/mpecan/allium-inspect/issues/6)–[#13](https://github.com/mpecan/allium-inspect/issues/13), [#16](https://github.com/mpecan/allium-inspect/issues/16)–[#27](https://github.com/mpecan/allium-inspect/issues/27) and [#31](https://github.com/mpecan/allium-inspect/issues/31)–[#33](https://github.com/mpecan/allium-inspect/issues/33).

- **Permission checks are local predicates.** A rule says
  `requires: can_write_code(user, repository)`, not `access/can_write(...)`,
  because the parser reads a qualified call as a division. Each module's header
  maps every local predicate to the `access` contract function it stands for
  (`AuthorizationPolicy` in `access.allium`). An implementation should route all
  of them through one authorisation service.
- **Every emission uses named arguments, and listeners use the same names.**
  The walker binds a trigger's arguments per listener: a named argument binds
  only the parameter of that name, and a bare name is positional. Mixing the two
  is a collision. Cross-module listeners write the emitter's parameter names
  even for parameters they ignore, so the contract is visible at both ends.
- **A module never emits into another module's namespace.** It emits its own
  trigger, and the owning module reacts to it, for example
  `actions/JobCommitStatusReported` and code's reaction to it. The walker
  supports both styles; this one keeps each module's effects inside the module
  that owns them.
- **A deletion event carries what its listeners need, read with `let`.** An
  emission's arguments see the state *after* the rule, so a rule that deletes a
  record binds the fields it reports with `let` first.
  The walker enforces this, and names the `let` to add. A listener reads nothing
  from a deleted record but its identity.
  For example, `UserAccountDeleted` carries the user's `kind`, and
  `PackageVersionDeleted` carries its `package`.
- **Surfaces face real actors, and journeys cast people who match them.** The
  walker checks `facing`. The shared actors live in `access.allium`: `Visitor`
  (anyone), `SignedInUser`, `RestrictedUser`, `SiteAdmin` and `RepoAdmin`. Cast
  users carry the fields those actors need (`is_active`, `prohibit_login`, …).
  A surface facing a `within` actor passes that context (repository, issue,
  organization, team) as an argument of every trigger it provides. Permissions
  are *also* rule `requires`: a rule never trusts its caller.
- **A rule-level `for` filters with `where`.** A member that doesn't qualify is
  skipped by the filter. A `requires` inside the loop would be ambiguous
  between "refuse the rule" and "skip the member".
- **Black boxes are answered where the story is told.** A call about something
  that already exists is stipulated in `given` or the file's `world`. A call
  about something the act creates is answered on the act's `creating` line
  (`creating issue: issues/Issue where audience(issue) = {}`). An act that
  creates several things has one `creating` line for each. Every answer is
  listed in the journey's stipulation ledger.
- **Off-by-default features are gated in their rules and walked switched on.**
  Federation, quota, moderation and LFS each check `config.enabled` in `requires`.
  Their journeys set `config.… = true` (it shows in the stipulation ledger), and
  each has one journey showing the refusal when it is off.

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

### Verdicts

Every step holds. The walk reports no `undecided`, `refused`, `unspecified` or
`unexposed` step, and no diagnostic. Each journey's stipulation ledger lists
everything it took as given: cast fields, config switched on, and black-box
answers. That ledger is the place to look when a verdict seems too easy.

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
4. A second pass adapted the set to allium-journey 0.1.2. That release enforces
   `facing`, binds arguments per listener, follows chains to the end, and reads
   `when` guards, quantifiers, creations and config. Five agents worked on
   disjoint files from one brief. Along the way they turned up and fixed real spec
   bugs: an admin user deletion that skipped the accounts cascade, an auto-merge
   that read its schedule after deleting it, a rename redirect that compared a
   name with itself, and listeners that read records their own event had just
   deleted.
5. A third pass adapted the set to allium-journey 0.1.3, which answers black
   boxes on an act's creations, takes calls in `given` and `world`, and seeds
   collection config defaults. Two agents used it to make the remaining steps
   hold. They also removed the workarounds the earlier walker had needed:
   restated defaults, unrolled loops, `if` in place of `??`, and stipulations
   repeated in every journey.
6. A last pass adapted the set to allium-journey 0.1.4, which reads a removed
   record the same way whichever `ensures` line comes first (strictly: not at
   all), catches several creations per act, and answers module-qualified
   calls. It turned up eight more rules that reported fields of a record they
   had just deleted. After fixing those, all 703 steps hold.
7. The 201 open questions were then decided with the project owner, module by
   module, and recorded in [`DECISIONS.md`](DECISIONS.md). One agent per module
   applied its decisions: removing the question, changing rules where the
   decision changed behaviour, and adding journeys for the new behaviour
   (often a refusal). Two follow-up questions that came up while applying them
   are recorded too. The set grew to 452 journeys, and all 1,025 steps hold.

