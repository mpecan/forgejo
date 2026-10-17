# Shared vocabulary

Every module in this spec set uses these names. A concept has exactly one name
and exactly one owning module; every other module refers to it by its qualified
name (`accounts/User`) after `use "./accounts.allium" as accounts`.

If you need a field on an entity you do not own, **do not redeclare the
entity**. Reference the field you need; if the owning module does not have it
yet, record it in the "Requests to other modules" section at the bottom of
your module file as a comment (`-- REQUEST accounts/User.field: why`). The
integration pass reconciles those.

## Modules and what each owns

| Module file | Alias | Owns (entities, enums, value types) |
|---|---|---|
| `accounts.allium` | `accounts` | `User` (people, bots, remote/federated users), `EmailAddress`, `UserRedirect`, `Follow`, `UserBlock`, `UserSetting`, profile, account status, `Visibility` enum |
| `organizations.allium` | `organizations` | `Organization`, `OrgMembership`, `Team`, `TeamMembership`, `TeamUnitAccess`, `TeamInvite`, org-level labels are NOT here (issues owns labels) |
| `authentication.allium` | `authentication` | `Session`, `AccessToken`, `TokenScope`, `TwoFactor`(TOTP), `WebAuthnCredential`, `AuthSource` (LDAP/SMTP/PAM/OAuth2/SSPI/remote), `ExternalLogin`, `OAuth2Application`, `OAuth2Grant`, `OAuth2AuthorizationCode`, registration, password reset, email activation |
| `keys.allium` | `keys` | `SshKey` (user/deploy/principal), `GpgKey`, `GpgKeyImport`, `CommitVerification`, signing policies |
| `repositories.allium` | `repositories` | `Repository`, `RepoUnit`, `UnitKind` enum, `Collaboration`, `Star`, `Watch`, `Topic`, `RepoTransfer`, `RepoRedirect`, `Mirror` (pull), `PushMirror`, `RepoArchive` (download archives), `RepoTemplate` generation, language stats, repo settings/flags |
| `access.allium` | `access` | `AccessMode` enum (`none | read | write | admin | owner`), the derived permission of a user on a repository per unit, restricted users, public/limited/private visibility consequences, the anonymous actor |
| `code.allium` | `code` | `Branch`, `Tag`, `Commit` (external-ish), `ProtectedBranch` (branch protection rule), `ProtectedTag`, `CommitStatus`, `Push`, `LfsObject`, `LfsLock`, web file edits, git transport (HTTP/SSH), server-side hooks, AGit flow, `DeletedBranch` |
| `issues.allium` | `issues` | `Issue` (an issue *or* the conversation half of a pull request — `is_pull`), `Comment` (all timeline event kinds), `Label`, `Milestone`, `Reaction`, `IssueDependency`, `TrackedTime`, `Stopwatch`, `IssuePin`, `IssueWatch` (subscription), `Attachment`, `ContentHistory`, `IssueTemplate`, cross references |
| `pulls.allium` | `pulls` | `PullRequest` (head/base/merge state; its conversation is an `issues/Issue`), `Review`, `ReviewRequest`, `ReviewComment` (code comment), `MergeStyle` enum, `AutoMerge`, conflict/mergeability checking, `CodeOwnerRule` |
| `projects.allium` | `projects` | `Project`, `ProjectColumn`, `ProjectCard` |
| `releases.allium` | `releases` | `Release` (incl. draft/prerelease/tag-only), release attachments (refers to `issues/Attachment`) |
| `wiki.allium` | `wiki` | `WikiPage`, wiki revisions |
| `actions.allium` | `actions` | `Workflow`, `WorkflowRun`, `Job`, `Task`, `Runner`, `RunnerRegistrationToken`, `Secret`, `Variable`, `Artifact`, `Schedule`, `ActionsCache` |
| `packages.allium` | `packages` | `Package`, `PackageVersion`, `PackageFile`, `PackageBlob`, `PackageKind` enum, `CleanupRule`, package-to-repo link |
| `webhooks.allium` | `webhooks` | `Webhook` (repo/org/system/default), `HookEventKind` enum, `HookDelivery` |
| `notifications.allium` | `notifications` | `Notification` (in-app inbox), `MailMessage` (outbound mail), `Activity` (feed entry), dashboards, RSS/Atom feeds, user heatmap |
| `administration.allium` | `administration` | instance configuration-at-runtime, `SystemNotice`, `CronTask`, admin user management, `Announcement`(service message/banner), installation, instance health |
| `quota.allium` | `quota` | `QuotaRule`, `QuotaGroup`, `QuotaSubject` enum, quota enforcement |
| `moderation.allium` | `moderation` | `AbuseReport`, `ModerationAction`, shadow copies, spam handling |
| `federation.allium` | `federation` | `FederationHost`, `FederatedUser`, ActivityPub actors/inbox/outbox, federated stars/follows, WebFinger, NodeInfo |
| `discovery.allium` | `discovery` | explore pages, search (repos, users, orgs, code, issues), indexers, repository activity/insights, sitemap |
| `migrations.allium` | `migrations` | **addon**: `Migration` task, import sources (forge kinds), what is imported |

## Fixed names other modules may rely on

These names and fields are guaranteed by the owning module. Use them freely.

```
accounts/User
    name: String                 -- unique handle, renamable
    kind: individual | bot | remote | activitypub   -- organizations are organizations/Organization
    is_admin: Boolean
    is_active: Boolean           -- email confirmed / activated
    is_restricted: Boolean       -- restricted users see only what they are explicitly given
    prohibit_login: Boolean
    visibility: accounts/Visibility
    primary_email: String
accounts/Visibility = public | limited | private

organizations/Organization
    name: String
    visibility: accounts/Visibility
    owners_team: organizations/Team
organizations/Team
    organization: organizations/Organization
    access_mode: access/AccessMode
    includes_all_repositories: Boolean
organizations/TeamMembership { team, user: accounts/User }
organizations/OrgMembership  { organization, user, is_public: Boolean }

repositories/Repository
    name: String
    owner_user: accounts/User?              -- exactly one of owner_user / owner_org is set
    owner_org: organizations/Organization?
    is_private: Boolean
    is_archived: Boolean
    is_mirror: Boolean
    is_fork: Boolean
    fork_of: repositories/Repository?
    is_template: Boolean
    is_empty: Boolean
    default_branch: String
    status: ready | being_migrated | pending_transfer | broken
repositories/UnitKind = code | issues | pull_requests | releases | wiki | external_wiki | external_tracker | projects | packages | actions
repositories/RepoUnit { repository, kind: repositories/UnitKind, enabled: Boolean }
repositories/Collaboration { repository, user: accounts/User, access_mode: access/AccessMode }

access/AccessMode = none | read | write | admin | owner
access/can_read(user, repository, unit)    -- black-box functions defined in access.allium
access/can_write(user, repository, unit)
access/is_repo_admin(user, repository)

code/Branch { repository, name, head_commit_sha: String, is_deleted: Boolean }
code/Tag    { repository, name, commit_sha: String }

issues/Issue
    repository: repositories/Repository
    index: Integer               -- per-repository number shared by issues and PRs
    poster: accounts/User
    title: String
    is_pull: Boolean
    status: open | closed
    is_locked: Boolean
issues/Comment { issue, poster, kind, content }
issues/Label, issues/Milestone, issues/Attachment

pulls/PullRequest
    issue: issues/Issue
    head_repository: repositories/Repository
    head_branch: String
    base_repository: repositories/Repository
    base_branch: String
    status: checking | mergeable | conflict | empty | error | merged | ...   -- pulls owns the full set

actions/WorkflowRun, packages/Package, webhooks/Webhook, notifications/Notification
```

## Cross-cutting conventions

- **Triggers named for what the actor does**: `UserCreatesRepository(user, ...)`,
  `AdminDeletesUser(admin, user)`. A system/background trigger is named for the
  event: `PushReceived(...)`, `MirrorSyncDue(...)`.
- **Notifications and webhooks are reacted to, not called.** A module emits a
  domain trigger (`IssueOpened(issue)`, `PullRequestMerged(pr, merger)`,
  `PushReceived(push)`), and `notifications.allium` / `webhooks.allium` /
  `actions.allium` write rules `when: issues/IssueOpened(issue)` that react to
  it. Do not create `Notification`/`HookDelivery` entities from inside other
  modules. Every module must list the domain events it emits in its header.
- **Evidence**: every entity and rule carries a one-line `-- from: path:line`
  comment pointing at the Forgejo code it was distilled from.
- **Not a compatibility spec**: describe the behaviour a rebuild needs. Things
  that exist only for legacy compatibility, Gitea API compatibility, or are
  workarounds go in the module header under `Excludes:` with a one-line reason,
  or become an `open question` when it is a genuine judgement call.
- **Config**: instance-level settings (`app.ini`) that change behaviour become
  `config` parameters with Forgejo's default as the default value.
