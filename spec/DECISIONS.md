# Decisions

Answers to the questions in [`OPEN-QUESTIONS.md`](OPEN-QUESTIONS.md), made while
reviewing the spec for the rebuild. Each decision is applied to the module it
belongs to: the `open question` is removed and the behaviour written as rules,
invariants or config. Numbers refer to the question's position in its module in
`OPEN-QUESTIONS.md` at the time it was decided.

## accounts

| # | Question | Decision |
|---|---|---|
| 1 | Following a user you can't see | **Require visibility.** Following is refused for an account the follower cannot see, as if it did not exist. |
| 2 | Reserved user/organization names | **Admins can reserve names.** A site admin can reserve a name (optionally when deleting a user); a reserved name can't be registered, used for an organization or taken by a rename until an admin releases it. |
| 3 | Can a block bind a site admin? | **Yes, like Forgejo.** A block applies to admins exactly as to anyone; an admin uses admin tools to override. |
| 4 | Reach of a personal block | **Personal only, like Forgejo.** A personal block covers the blocker's own repositories; organizations keep their own block list. |
| 5 | Admin self-deletion | **Allowed if another active admin remains.** The last active site admin cannot delete their own account. |
| 6 | Making an unactivated address primary | **Refused.** Only an activated address can become primary, checked by the rule, not only the UI. |
| 7 | Badges | **Dropped.** `Badge` and `UserBadge` leave the spec; nothing in Forgejo creates or awards them. |
| 8 | When inactive accounts are cleaned up | **Scheduled sweep only.** The admin-controlled cleanup task deletes eligible accounts when it runs. |
| 9 | A purge that fails halfway | **Resumable.** A purge is a recorded job; the account is marked as being purged and can't sign in; each step is idempotent; an admin can resume or retry a failed purge until it completes. |

## authentication

| # | Question | Decision |
|---|---|---|
| 1 | Password-reset link lifetime | **Its own setting.** Reset links expire after `reset_code_lifetime`, the value the reset email states. |
| 2 | OAuth2 authorization code lifetime | **10 minutes, configurable, single-use.** |
| 3 | Signing in before confirming the email | **Limited session, like Forgejo.** Sign-in succeeds into a session that can only resend activation or sign out. |
| 4 | Password Basic auth | **Tokens everywhere.** No username+password Basic auth on the API or on git/LFS transport; git clients use a token (or OAuth2) as the password. |
| 5 | Sign-out and remember-me | **Revoke on sign-out.** Signing out deletes that device's remember-me record. |
| 6 | Throttle between activation and reset mails | **Admin's choice.** A config setting chooses one throttle shared across account mails (Forgejo's behaviour, the default) or a separate throttle per kind of mail. |
| 7 | Passkeys | **Passwordless passkey sign-in.** A user can register a passkey (discoverable credential) and sign in with it alone; security keys still work as a second factor. |
| 8 | Auto-linking external accounts | **Require `email_verified`.** Auto-link matches by email only, and only when the IdP asserts the email is verified; otherwise the user signs in to the local account to link it. |
| 9 | Re-consent with a different scope | **Re-consent replaces the scope.** The consent screen shows the requested scopes; approving sets the grant to exactly those. Existing tokens keep their scope until refreshed. |
| 10 | Removing the last second factor when 2FA is required | **Blocked.** A user can't remove their last remaining second factor (TOTP, security key or passkey) while 2FA is required. |

## keys

| # | Question | Decision |
|---|---|---|
| 1 | SSH keys while SSH is disabled | **Keep user keys for signing.** User SSH keys can still be added (commit-signature verification); deploy keys and principals, which only serve SSH access, are refused while SSH is disabled. |
| 2 | Policy for externally supplied keys | **Apply the policy.** Directory/OAuth2 keys must meet the algorithm and size policy; a non-compliant key is skipped and recorded as a system notice. |
| 3 | Trust for SSH-signed commits | **Same rule for both formats.** SSH and GPG signatures are judged by the same trust model, collaborator check included. |
| 4 | Expired or revoked GPG keys | **Expiry-aware.** A signature made while the key was valid stays verified; one dated after expiry, or by a revoked key, is unverified with that reason. |
| 5 | Looking up other users' keys | **Own keys, plus a visibility-respecting lookup.** Listing/getting by id returns only the caller's keys; a lookup by fingerprint names the owning user only if the caller can see their profile, and never returns deploy keys. Site admins see everything. |
| 6 | GPG verification and the manage-keys flag | **Verification stays available** when users can't manage their GPG keys; it doesn't change the key set. |
| 7 | Deploy keys on archived repositories and mirrors | **Refused on archived repositories; read-only only on mirrors.** |
| 8 | `never` and `always` in a signing policy | **`never` wins**, whatever the order. |
| 9 | Certificates with several principals | **Try every principal.** Authentication succeeds if any listed principal is registered to an active user and passes the certificate checks. |

## organizations

| # | Question | Decision |
|---|---|---|
| 1 | Renaming the Owners team | **Forbidden** by every path; the Owners team is identified by its role, not its name. |
| 2 | Deleting the Owners team | **Forbidden** by every path. |
| 3 | Who changes a member's membership visibility | **The member only.** Owners can't publicize or conceal someone else's membership. |
| 4 | A team's summary access mode | **Dropped.** A team is owner, admin, or a set of per-unit grants; any summary is computed for display only. |
| 5 | Admin creating an organization for a user | **The admin's right governs.** A site admin can create an organization owned by any user, whatever that user's own creation right. |
| 6 | Adding a team to a repository | **Org owners, or repository admins when the org allows it,** on every path. |
| 7 | Org blocks and team membership | **Not checked, like Forgejo.** An org block limits the blocked user's own interactions; it doesn't stop owners adding them to teams. |
| 8 | Expired team invitations | **Kept, like Forgejo.** They stay listed as expired until an owner removes them or a new invitation replaces them. |

## access

| # | Question | Decision |
|---|---|---|
| 1 | The access index | **Permission is computed live.** The spec defines it purely from current state; any index or cache is an implementation detail that must give the same answers. The recalculation rules leave the spec. |
| 2 | Collaborator who is also on a team | **Per-unit strongest.** On each unit the mode is the maximum of the collaboration mode and that unit's team grant. |
| 3 | Public units of a private repository | **Supported.** A repository admin can make individual units public on a private repository; visitors get read on those units only. |
| 4 | Default permission for signed-in users | **Generalised to every unit.** Each unit can set a default for signed-in users (none/read/write) and for anonymous visitors (none/read). |
| 5 | Site admins with a fine-grained token | **The token limits everything.** Scope and repository restrictions always apply; admin powers need a token with the admin scope. |
| 6 | Actions job tokens on organization endpoints | **Repository-scoped only.** A job token reaches its own repository and packages linked to it; organization endpoints refuse it. |
| 7 | Watches and subscriptions after losing access | **Cleared immediately** whenever a user loses read access to a repository, for any reason. |

## repositories

| # | Question | Decision |
|---|---|---|
| 1 | Who can archive a repository | **Any repository admin** may archive or unarchive, on every path. |
| 2 | Changing a fork's visibility | **Refused with a reason** when it would make a fork of a private repository public; a fork of a public repository may be made private. |
| 3 | Stars when a repository goes private | **Stars of people who keep read access stay;** the others are removed (consistent with clearing on lost access). |
| 4 | Adding collaborators | **Invitation and acceptance.** Adding a collaborator sends an invitation; access starts when they accept. Site admins can add directly. |
| 5 | Failing pull mirrors | **Back-off and a failing state.** Retries back off exponentially up to the mirror interval; after a configurable number of consecutive failures the mirror is marked failing, shown to repository admins, and paused until someone syncs it. |
| 6 | Who can trigger a pull-mirror sync | **Code write**, on every path. |
| 7 | Push-mirror sync on an archived repository | **Allowed** on every path; pushing unchanged content keeps backups current. |
| 8 | Transfers and the repository count limit | **Checked.** Accepting a transfer is refused if it would exceed the recipient's repository limit; site admins are exempt. |
| 9 | Forks after the forker loses access | **Kept, like Forgejo.** The fork stays the forker's, private and linked. |
| 10 | Transferring into an organization | **Immediate when the doer may create repositories there,** like Forgejo; the org owners are notified. |

## code

| # | Question | Decision |
|---|---|---|
| 1 | Quota on pushes to protected branches | **Admin's choice.** A config setting decides whether pushes to protected branches are exempt from the quota check (Forgejo's behaviour) or checked like every other push. |
| 2 | Renaming a branch | **Code write, except protected branches and the default branch,** which need repository admin. |
| 3 | Web upload limits | **Enforced on the server.** Upload rules refuse too many or too-large files whatever the client. |
| 4 | Deploy-key pushes | **First-class pusher.** The deploy key is the pusher; allowlists can name deploy keys, and the feed shows the key, never the owner. |
| 5 | Force-push to protected branches | **Optional force-push allowlist** on branch protection (users, teams, deploy keys); empty means nobody. |
| 6 | Admins and protected tags | **No exemptions.** Only the allowlist may create or delete a protected tag; an admin who needs to changes the rule. |
| 7 | Deleting a tag that has a release | **Refused on push too** when the release is published; delete the release first. Draft releases don't block. |

## issues

| # | Question | Decision |
|---|---|---|
| 1 | Editing a locked issue | **The poster may still edit the title and description;** triagers may edit everything. Locking stops the conversation, not corrections. |
| 2 | Reactions on a locked issue | **Anyone who can read may react;** the lock doesn't affect reactions. |
| 3 | Lock reasons | **Validated on every path:** empty or one of the configured reasons. |
| 4 | Blank issues when disabled | **Refused on every path;** repository writers excepted. |
| 5 | Orphan uploads | **Expire** when not linked to an issue, comment or release within a configurable time (default 24 hours), via a scheduled task. |
| 6 | Metadata from non-triagers | **All dropped:** labels, milestone, assignees and project are ignored for someone who can't triage. |
| 7 | Timeline events of deleted labels and milestones | **Kept for both,** showing the name at the time, marked deleted. |
| 8 | Logging time for someone else | **Repository admins may,** on every path, with a chosen date; the entry records who logged it. |
| 9 | Issue priority | **Dropped.** Priority is expressed with scoped labels. |

## pulls

| # | Question | Decision |
|---|---|---|
| 1 | Scheduling auto-merge when one exists | **Replaces it** (style, message, who scheduled it); the timeline shows the change. |
| 2 | Closing a PR with auto-merge scheduled | **Closing cancels the schedule;** reopening doesn't restore it. |
| 3 | A failing auto-merge | **Notify and remove:** the schedule is removed, a timeline comment says why, and whoever scheduled it is notified. |
| 4 | `ancestor` vs `empty` | **Kept distinct.** |
| 5 | Reviews on closed or merged PRs | **Allowed, like Forgejo,** comments and verdicts alike. |
| 6 | Who can request reviews | **Any reader of pull requests, like Forgejo.** |
| 7 | Outdated code comments on AGit pushes | **Every head move outdates,** AGit included. |
| 8 | Update branch for AGit PRs | **Supported:** the server can merge or rebase the base into the AGit head ref. |
| 9 | Squash-merge author | **The poster's account identity** (commit email, respecting keep-email-private); co-authors become `Co-authored-by` trailers. |
| 10 | Mergeability checks | **Always asynchronous;** a new or updated PR is `checking` until the check finishes. |
| 11 | Merge-message templates | **Repository template, else an instance-wide template, else the built-in default.** |

## projects

| # | Question | Decision |
|---|---|---|
| 1 | Editing a closed owner board reopens it | **Bug.** Editing never changes whether a board is open or closed. |
| 2 | An empty description can't clear it | **Bug.** An empty description clears it. |
| 3 | Global switch and owner boards | **One switch disables all boards:** repository, user and organization. |
| 4 | Adding an issue to an organization board | **Board write plus triage on the issue** in its repository. |
| 5 | Boards of a deleted user or organization | **Deleted with their owner;** projects reacts to the owner-deletion events. |
| 6 | Cards after a repository transfer | **Removed** from the old owner's boards when the repository moves. |
| 7 | Closed boards and new cards | **A closed board accepts no new cards;** existing cards can still move. |
| 8 | `closed_at` after reopening | **Cleared on reopen;** kept only while closed. |
| 9 | Projects API | **Everything the web offers** is in the API too. |
| 10 | Templates longer than the column cap | **A configuration error** at startup. |
| 11 | Reordering cards the actor can't read | **Filtered:** a reorder only moves cards whose issues the actor can read. |

## releases

| # | Question | Decision |
|---|---|---|
| 1 | Publishing a draft whose tag exists | **Emits `ReleasePublished`,** like every draft-to-published change. |
| 2 | Un-publishing | **A new `ReleaseUnpublished` event** on every published-to-draft change (by edit or by the tag being deleted), for webhooks and feeds. |
| 3 | A pushed tag reviving a draft | **Emits `ReleasePublished`.** |
| 4 | Retargeting a published release | **Drafts only.** A published release's tag is fixed; un-publish first to change it. |
| 5 | Deleting a never-tagged draft | **Removes the record entirely.** |
| 6 | Deleting a tag that has a published release | **Refused** (settled by code decision 7); delete the release first. |
| 7 | Attachment size limit | **Enforced on every upload path.** |
| 8 | Release changes on archived repositories | **All frozen,** deletes included. |
| 9 | Permission to create a bare tag | **Code write,** wherever it happens; protected-tag rules apply. |
| 10 | Releases on pull mirrors | **Read-only.** A pull mirror's releases follow upstream tags only; creating, editing or deleting them is refused. |

## wiki

| # | Question | Decision |
|---|---|---|
| 1 | Renaming onto an existing page | **Refused.** |
| 2 | Edit conflicts | **Stale edits refused.** An edit carries the revision it started from; if the page has changed since, the save is refused. |
| 3 | Empty page content | **Refused on every path.** |
| 4 | Git pushes to the wiki | **Emit an event per changed page** (created, edited, deleted), attributed to the pusher. |
| 5 | Who deletes wiki data and changes the wiki branch | **Repository admins,** on every path. |
| 6 | Editing a page that doesn't exist | **Refused;** creating is a separate operation. |
| 7 | Nested pages | **Supported.** A `/` in a page name is a directory path. |

## actions

| # | Question | Decision |
|---|---|---|
| 1 | Dispatching a disabled workflow | **Allowed.** Disabling stops automatic triggers (push, schedule, events) and re-runs; a manual dispatch still starts it. |
| 2 | Who manages repo secrets, variables and runners | **Repository admins,** on every path. |
| 3 | Artifact retention vs `ARTIFACT_RETENTION_DAYS` | **The config is only the default.** A workflow may request any retention. |
| 4 | Runner registration tokens | **Expire after a configurable lifetime** (default 1 hour), reusable for several runners until then. |
| 5 | Legacy OIDC subject format | **Dropped.** One subject format only; no `legacy_forgejo_15` option. |
| 6 | Artifact protocols | **v1–v4 all supported.** |
| 7 | Pruning old run records | **Admin config, off by default.** When set, finished runs older than the retention age are deleted with their logs and artifacts. |
| 8 | Disabling schedules for inactivity | **Admin config, off by default.** When set, a repository with no activity for that period has its scheduled workflows disabled and its admins are notified. |

## packages

| # | Question | Decision |
|---|---|---|
| 1 | Linking a package to a repository | **The API's checks everywhere:** package write plus write on the repository, and a linked package must be unlinked before relinking. |
| 2 | Unlinking permission | **Symmetric with linking:** package write plus write on the linked repository. |
| 3 | Download-age conditions on cleanup rules | **User-editable** by package managers. |
| 4 | "Public only" tokens and limited owners | **Limited owners excluded too.** Public only means visible to anonymous visitors. |
| 5 | Actions job tokens and packages | **Own repository's packages only.** A job token may publish new packages (linked to its repository) and write packages linked to its repository; everything else is read-only to it. |
| 6 | `LIMIT_TOTAL_OWNER_*` vs quota | **Folded into quota.** The per-type limits are dropped; package limits are expressed as default quota rules. |
| 7 | Dangling container digests | **Keep the split.** Instance cleanup purges dangling digests; user cleanup rules never touch them or `latest`. |
| 8 | Organisation deletion with packages | **Purge option,** as for users. |

## webhooks

| # | Question | Decision |
|---|---|---|
| 1 | Inactive hooks and globally disabled webhooks | **No delivery is created;** `last_status` is untouched. |
| 2 | Kinds that can't render an event | **Dropped at queue time,** and the hook form offers only the events the kind supports. |
| 3 | Gate for review comments | **`pull_request_review`** gates approve, reject and comment reviews. |
| 4 | API create/edit | **Full config via the API.** API and web set the same fields; an edit changes only the fields given. |
| 5 | Copying default/template hooks | **The Authorization header is dropped;** the copy must be re-authorised. The rest of the config is copied. |
| 6 | Hooks of a deleted owner | **Removed,** with their delivery history. |
| 7 | Authorization header visibility | **Write-only,** like the secret: managers can replace or clear it, never read it. |
| 8 | Instance-wide delivery log | **Yes.** Site admins can browse all deliveries, filterable by hook and status. |
| 9 | Automatic retries | **Retry with backoff.** A configurable number of attempts (default 3) with exponential backoff on network errors and 5xx responses; manual redelivery stays. |
| 10 | `gitea` and `gogs` kinds | **Dropped.** One native kind plus the chat/CI integrations; the import addon maps old hooks. |
| 11 | Test deliveries | **Every level:** repository, organisation, user and system hooks. |

## notifications

| # | Question | Decision |
|---|---|---|
| 1 | Inbox vs mail audience | **Unified.** One audience: subscribers, kind-specific watchers, participants, assignees and mentions, minus the actor, unsubscribers and users who blocked the actor. Mail goes to that audience filtered by each person's mail preference. |
| 2 | Filters for new pull requests | **Applied.** New pull requests go through the same unsubscribe and block filters as every other notification. |
| 3 | Mail reply/unsubscribe tokens on password change | **Revoked explicitly** on password change and account reset. |
| 4 | Job-failure mail | **Exactly one mail per failed run attempt.** |
| 5 | Collaborator mail | **Every path.** The invitation mail goes out however the invitation is made. |
| 6 | Activity feeds | **Computed on read:** events from what the reader follows or watches now, filtered by current visibility. |
| 7 | The `commit` notification source | **Dropped.** |
| 8 | Release mail for prereleases | **Prereleases are mailed too.** |
| 9 | Retrying failed mail | **No retry.** A failed mail is dropped and logged. |

## administration

| # | Question | Decision |
|---|---|---|
| 1 | Doer of a manually run cron task | **The admin who triggered it,** on every path. |
| 2 | Global cron on/off | **No global switch.** Each task has its own enabled flag; the misleading global settings are dropped. |
| 3 | Maintenance / read-only mode | **Yes.** Admins switch it on; every write (web, API, git push) is refused with a banner; reads and admin pages keep working. |
| 4 | Restoring a dump | **CLI restore** that loads a dump into an empty instance. |
| 5 | Announcement kinds | **Modal or banner,** each with optional start and end times. |
| 6 | Admins weakening themselves | **One guard for all of it.** A self-edit that would leave no other active admin (demote, restrict, deactivate, prohibit login) is refused. |
| 7 | Adopting unadopted repositories | **Repositories owns adopt and delete,** for users and admins; administration owns the instance-wide listing. |
| 8 | Update checker default | **Off.** Admins opt in. |
| 9 | Doctor checks from the web | **Read-only checks** can be run and viewed from the web admin; fixes that change data stay CLI-only. |
