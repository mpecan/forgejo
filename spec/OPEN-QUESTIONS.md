# Open questions

Every `open question` declared in the spec set, gathered by module. Each one is a decision
the rebuild has to make: a place where Forgejo's web UI, API and code disagree, where its
behaviour looks accidental, or where the distillation could not tell intent from accident.
The spec itself takes a position where it can (usually the stricter or more consistent one)
and says so in the question.

Generated from the `.allium` files; regenerate rather than edit by hand.

**202 questions across 22 modules.**


## accounts (9)

1. Follow visibility: Forgejo lets anyone follow a user they cannot see (no visibility check on follow). The spec requires visibility to avoid leaking private accounts; confirm.
2. Reserved user/organization placeholder kinds (UserTypeUserReserved/OrganizationReserved) exist in the schema with no creation path; should the rebuild offer admin-reserved names (anti re-registration) as a feature?
3. Admins are not immune to being blocked (TODO in services/user/block.go). Should a block prevent a site admin's actions?
4. Blocking does not remove the blocked user from repositories the blocker owns through an organization, nor org membership; should it?
5. Self-deletion is refused for site admins entirely (not only the last admin). Keep, or allow when other admins remain?
6. Making an address primary via the web does not re-check activation server-side (only the UI hides the button). The spec requires activation; confirm.
7. Badges have a schema and display but no management UI or API; should the rebuild offer admin badge management?
8. Inactive-account cleanup runs only on its cron schedule (default yearly, disabled). Should eligibility be continuous instead?
9. Purge disables the account outside the deletion transaction so a failure can leave a disabled, half-purged account; should purge be resumable?

## authentication (10)

1. Password-reset codes expire after ACTIVE_CODE_LIVE_MINUTES although RESET_PASSWD_CODE_LIVE_MINUTES is shown to the user; should reset codes honour reset_code_lifetime?
2. OAuth2 authorization codes have a valid-until column that is never set, so codes never expire until used; the spec assumes a 10-minute lifetime — is that the intended behaviour?
3. Sign-in does not refuse inactive accounts; they get a session that can only reach the activation page. Should an unconfirmed account rather be refused at sign-in?
4. Password basic auth on the API is marked FIXME for removal; should the rebuild keep it (git clients rely on it) only for git/LFS routes?
5. Signing out does not delete the remember-me record server-side (only the cookie); should sign-out revoke it?
6. The mail throttle key is shared between activation resend and password reset, so one blocks the other for 3 minutes; intended?
7. WebAuthn is a second factor only; should discoverable-credential (passkey) passwordless sign-in be specified for the rebuild?
8. Account-link auto mode trusts an IdP-asserted e-mail or username to link to an existing local account; should this require the IdP's email_verified claim?
9. A grant that exists with a different scope makes consent fail until the user revokes it; should a re-consent update the scope instead?
10. Disabling TOTP is blocked when 2FA is required, but removing the last security key is not; should that be blocked too?

## keys (9)

1. Adding personal SSH keys and deploy keys is allowed while DISABLE_SSH is set (only principals check it). Should a rebuild refuse key registration when SSH is disabled, or keep keys for commit signature verification?
2. Externally supplied SSH keys (LDAP / OAuth2) bypass the algorithm and minimum-size policy. Should the policy apply to them too?
3. Commit trust for SSH-signed commits: Forgejo only consults the collaborator check for GPG signing keys, so an SSH signature by a non-collaborator is 'trusted' under the collaborator models. The spec applies the same rule to both formats; confirm.
4. Signatures by expired or revoked GPG keys still verify (expiry is only displayed). Should verification take key expiry into account?
5. GET /api/v1/user/keys/{id} returns any user's key by id and GET /api/v1/user/keys?fingerprint=... searches keys of all users including deploy keys. Is cross-user lookup intended (it is how 'who owns this key' works) or should it be limited to the caller's keys?
6. The GPG verify endpoints ignore the manage_gpg_keys feature flag, while adding and deleting honour it. Intended?
7. Deploy keys can be added to archived repositories and mirrors. Should that be refused like other settings changes on archived repositories?
8. The signing policy lists are order-sensitive in Forgejo (the first of never/always wins); the spec says never wins over always. Confirm.
9. When several principals of one certificate are registered, Forgejo stops at the first one whose certificate check fails instead of trying the rest. Keep or fall through?

## organizations (8)

1. Forgejo's API lets an owner rename the Owners team (PATCH /teams/{id} with a name), which breaks owner detection by name; the web UI does not. This spec forbids it — confirm.
2. Forgejo does not refuse deleting the Owners team in the backend (only the UI hides the button). This spec refuses it — confirm.
3. Membership visibility: the web lets owners publicize/conceal other members, the API only lets members change their own. Which should a rebuild keep? (Spec: self or owner.)
4. A read/write team's summary access_mode is the lowest non-none unit grant (team_mode_for). Is a summary needed at all, or should read/write teams be defined purely by their unit grants?
5. AdminCreatesOrganizationForUser checks the target owner's permission to create organizations rather than the admin's. Intended?
6. Team-repository add via API requires repository admin (reqAdmin) plus team membership, while the web requires org ownership. Spec treats the API path as the repo-side RepoAdminAddsTeamToRepository (honouring repo_admin_change_team_access). Confirm.
7. Should a user block at org level prevent the blocked user from being added to or invited into the org's teams? Forgejo does not check.
8. Expired invitations are never cleaned up automatically; they stay listed until revoked or replaced. Should a rebuild purge them after some time?

## access (7)

1. Forgejo does not recalculate the access index when a user's restricted flag or an owner's visibility changes; the index is only an optimisation there. Should a rebuild recalculate (as specified here) or never depend on the index for those decisions?
2. On an organization repository a user who is both a collaborator and a team member gets the access-table maximum (including the team's access mode) on every unit, bypassing per-unit team limits. Accidental (spec assumes per-unit strongest of collaboration and team unit access) or intended?
3. Anonymous visitors cannot read individual public units of a private repository (TODO in GetUserRepoPermission). Should per-unit anonymous/public access on private repositories be supported?
4. Everyone-writable is only offered for the wiki unit. Should the per-unit default permission for signed-in users (and an anonymous access mode) be generalised to every unit?
5. A site administrator using a fine-grained token keeps owner mode on repositories the token reaches but loses the admin override on admin-only endpoints. Is that asymmetry intended?
6. Actions task tokens are treated as signed-in for reqToken but have no user; should organization-level endpoints accept them?
7. Visibility of repositories after a repository is made private: forks inherit privacy and stars are cleared (repositories). Should watches and issue subscriptions of users who lost access be cleared immediately via UserLostRepositoryAccess as specified?

## repositories (10)

1. Archiving requires owner access on the web but only repository-admin access through API PATCH archived; which level should a rebuild require? (spec: owner)
2. Should a fork's visibility be refused when changed (spec) or silently forced back to its base's visibility (Forgejo web and API)?
3. Making a repository private deletes all of its stars permanently; should stars instead be hidden and restored when it becomes public again?
4. Collaborators are added without consent; should a rebuild add an invitation/acceptance step like organisation team invites?
5. A failed pull-mirror fetch is retried every update run (10 minutes) indefinitely; should there be back-off or a failure state shown to admins beyond a SystemNotice?
6. Web settings sync-now for a pull mirror needs repository admin, the API only code write access; which is intended? (spec: code write)
7. Push-mirror sync-now via web ignores archived state while the API refuses it; spec refuses on archived repositories.
8. Transferring requires the recipient's quota, but the per-owner repository count limit is not checked on transfer; intended?
9. Forks of private repositories by users who later lose access are kept; should they be removed or detached?
10. Org 'can create repositories' members get transfers completed instantly without consent of the org owners; intended?

## code (7)

1. Quota is not checked for pushes to protected branches in Forgejo (only unprotected branches, tags, other refs). This spec applies it uniformly — confirm.
2. Branch rename needs repository admin in the web UI but only code write in the API. This spec requires code write — which is intended?
3. Upload FILE_MAX_SIZE and MAX_FILES are only enforced by the browser upload widget; should the server enforce them?
4. A deploy key push is attributed to the repository owner (protected tag allowlists, feed). Should deploy keys be first-class pushers?
5. Force-push to a protected branch is refused for everyone, with no force-push allowlist. Should a rebuild add one?
6. Protected tags do not exempt repository or site admins, while branch protection exempts site admins on merge. Intended asymmetry?
7. Deleting a tag through the API is refused when a release uses it, but a git push may delete it freely. Should pushes also be refused?

## issues (9)

1. Web lets an issue's poster change the title of a locked issue; the API refuses poster edits (title, body, ref, state) while locked. Which should the rebuild adopt? The spec follows the web for title/body and requires write for nothing else.
2. Web reactions ignore the lock; API reactions on a locked issue require write access. The spec follows the API (locked means only triagers react).
3. Web lock form validates the reason against LOCK_REASONS; the API accepts any reason. The spec validates.
4. When `blank_issues_enabled` is false the web form only redirects to the chooser; posting a blank issue (or via API) is still accepted. Should blank issues be refused?
5. Uploaded attachments never linked to an issue or comment are kept indefinitely (only a doctor command removes orphans). Should unlinked uploads expire?
6. Non-writers creating an issue through the web form can still set a milestone and assignees (only labels are dropped); the API drops all metadata. The spec drops all metadata for non-managers.
7. Deleting a label erases its label timeline events, while deleting a milestone keeps its events with a 'deleted milestone' placeholder. Should both behave alike?
8. Repository admins may log time for another user and back-date it via the API but not in the web UI; keep it?
9. Issue `priority` exists in storage but nothing sets or shows it; dropped from the spec.

## pulls (11)

1. Scheduling auto-merge when one already exists: the web form silently replaces it, the API refuses with a conflict. Which should the rebuild do?
2. Closing a pull request leaves its scheduled auto-merge in place (it can fire again after reopening). Should closing cancel the schedule?
3. A failing auto-merge is only logged; should the scheduler be told and the schedule removed?
4. Is the 'ancestor' status (head already contained in base, autodetect off) worth keeping distinct from 'empty'?
5. Submitting reviews and code comments is allowed on closed and merged pull requests; intended?
6. Review requests may be added by any collaborator or org team member with pull-request read access, even without write access; intended?
7. Code-comment outdating is skipped for AGit pushes in Forgejo (TODO in the code); the spec applies it to every head move.
8. Update-branch is unsupported for AGit pull requests; should a rebuild support it?
9. Squash-merge author: the PR poster's commit identity is preferred over their primary email; keep this heuristic?
10. Mergeability is checked synchronously at creation and asynchronously afterwards; the spec makes every check asynchronous.
11. Should merge-message templates live only in the repository (.forgejo/default_merge_message) or also at instance level?

## projects (11)

1. Editing a user/org board in Forgejo also forces it open (the edit form sends status=open), so editing a closed owner board silently reopens it; repository boards do not. Treat as a bug (spec says edit never changes status)?
2. Forgejo ignores an empty description on edit, so a description can never be cleared once set. The spec allows clearing; confirm.
3. User/organization boards ignore DISABLED_REPO_UNITS=repo.projects (routes pass ignoreGlobal=true) while repository boards honour it. Should a global switch disable all boards?
4. Assigning an issue to an organization board needs write on the organization's issues unit (team-level), not on the issue's repository; a repo collaborator without team membership cannot use org boards. Intended?
5. Projects of a deleted user or organization are not deleted in Forgejo (orphaned). The rebuild should delete them with their owner; confirm and decide which module owns the reaction.
6. When a repository is transferred to a new owner, cards on the old owner's boards are kept even though the board no longer accepts the issue (CardIssueInScope breaks). Remove such cards on transfer?
7. Closed boards remain assignable from the issue picker. Should a closed board refuse new cards?
8. Forgejo stores closed_at on reopen too and never clears it. The spec keeps closed_at only while closed.
9. There is no REST API for projects in Forgejo. Which operations should the rebuild's API expose (likely all surface provides)?
10. Template seeding with a configured column list longer than max_columns_per_project - 1 bypasses the column cap in Forgejo. Cap the template, or reject the config?
11. Moving cards via drag-and-drop on a repository board requires each issue to be from that repository; on owner boards from any repository of the owner. Should the reorder also be filtered to issues the actor can read?

## releases (10)

1. Publishing a draft whose tag already exists emits only 'release updated' in Forgejo (not 'published'); this spec emits ReleasePublished on every draft -> published. Confirm.
2. Un-publishing (published -> draft) and tag-deletion-by-push (published -> draft) are silent in Forgejo; should webhooks/feeds get a 'release unpublished' event?
3. A push that brings back the tag of a draft release publishes it; Forgejo does so silently. This spec emits ReleasePublished. Intended?
4. Forgejo lets the API change tag_name/target of a published release, which creates a second tag and leaves the old one; this spec restricts retargeting to drafts without a tag. Confirm.
5. Deleting a draft that never had a tag leaves a tag-less 'tag' record in Forgejo; this spec removes the record entirely. Confirm.
6. Forgejo's tags-page delete removes a tag together with its published release (and emits release-deleted); code.allium refuses deleting a tag that carries a published release. Which is intended?
7. [attachment] MAX_SIZE is enforced only by the web upload widget in Forgejo; this spec enforces it on every upload path. Confirm.
8. Archived repositories: Forgejo's API still allows deleting releases/assets of an archived repository; this spec freezes all release mutations. Confirm.
9. Creating a bare tag needs releases-write on the web release form ('tag only') but code-write on the API; should one permission govern both?
10. Should pull-mirrors' releases be read-only (mirror owners cannot meaningfully write releases for upstream tags)?

## wiki (7)

1. Renaming a page onto the name of another existing page silently overwrites it in Forgejo; this spec refuses the rename. Confirm.
2. Saving an edit has no conflict detection (last write wins). Should a rebuild require the base revision and refuse stale edits?
3. The web form requires non-empty content but the API accepts an empty page. Which is intended?
4. Pushing to the wiki with git produces no activity-feed entry, notification or webhook (TODO in Forgejo). Should it emit WikiPageEdited per changed page?
5. Deleting wiki data is restricted to repository owners on the web, while changing the wiki branch is allowed to repo admins via the API but only owners on the web. Unify on repo admin?
6. Forgejo's API PATCH on a page that does not exist creates it; this spec requires the page to exist (create is a separate operation). Confirm.
7. Wiki pages are flat (a '/' in a title is escaped into the name). Should a rebuild support nested page directories?

## actions (8)

1. Dispatching a disabled workflow is allowed by Forgejo (manual and API dispatch do not check the disabled list) while re-runs and scheduled runs of it are refused. Should dispatch also be refused?
2. Repository-scoped secrets/variables/runners need repository admin on the web UI but repository owner on the REST API. Which permission does the rebuild want?
3. Artifact retention requested by a job is not capped by ARTIFACT_RETENTION_DAYS. Should the instance setting be a maximum as well as the default?
4. The registration token is reusable until rotated and never expires. Should registration tokens expire or be single-use?
5. Should the legacy OIDC subject format (legacy_forgejo_15) be offered at all in the rebuild, or only the standard one?
6. Should the v1-v3 artifact protocol (still used by actions/upload-artifact@v3) be supported, or only v4?
7. Deleting a run discards its artifacts and logs; there is no retention for run records themselves. Should old runs be pruned automatically?
8. Scheduled workflows are never disabled for inactivity (GitHub disables after 60 days without repository activity). Keep Forgejo's behaviour?

## packages (8)

1. The web settings page links a package to any repository of the same owner without checking the user's write access to that repository and without requiring the package to be unlinked first, while the REST API checks both. The spec adopts the API behaviour for both; confirm.
2. Unlinking requires the actor to be a site admin, the owning user, or an organisation owner, on top of package write access; linking only needs package write and repository write. Is this asymmetry intended?
3. The cleanup-rule form renders keep/remove last-download-days fields but never binds them, so they cannot be set from the UI today. The spec lets managers set them; confirm they should be user-editable.
4. A token restricted to public resources is refused only for private owners; limited owners are allowed. Should 'public only' also exclude limited owners?
5. Actions workflow tokens can publish to the repository owner's registry with the owner's package permissions (marked TODO in Forgejo). Should they be narrowed to the workflow's own repository / linked packages?
6. Owner count/size limits (LIMIT_TOTAL_OWNER_*) and the quota module overlap. Should a rebuild keep both mechanisms or express the LIMIT_* settings as default quota rules?
7. Dangling container digest versions are purged only by the expired-data cleanup; cleanup rules deliberately skip them and the 'latest' tag. Keep this split?
8. Owner deletion: users/orgs are refused deletion while they own packages unless purged; org purge does not exist. Should organisation deletion also be able to purge packages?

## webhooks (11)

1. Inactive or globally-disabled deliveries: Forgejo marks them delivered, not succeeded, and sets the webhook's last status to failed. Should a skipped delivery leave last_status untouched instead?
2. Kinds that cannot render an event (e.g. sourcehut_builds for issues): Forgejo still creates the delivery and it stays undelivered forever (re-queued at every start). The spec drops such events at queue time (kind_supports) — confirm, or record them as failed?
3. Review comments: Forgejo's event checker gates pull_request_review_comment on the pull_request_comment choice while the matching helper uses pull_request_review. The spec uses pull_request_review for all three review outcomes — confirm.
4. API edit (PATCH .../hooks/{id}) resets event mode to choose_events and leaves package, workflow_run_* and workflow_job_* selections untouched; the API create path also ignores http_method. Should API create/edit accept the full WebhookConfig like the web form?
5. Copying default webhooks into a new repository and copying template webhooks drop the Authorization header in Forgejo. The spec copies the whole config, including it — confirm.
6. Owner deletion: Forgejo leaves a deleted user's or organization's webhooks behind. The spec removes them — confirm.
7. Secret: the web edit form re-renders the stored secret into a password field, while the API never returns it. The spec treats it as write-only; the Authorization header, however, is returned in clear to managers by the API and form — should it also become write-only?
8. Delivery history is visible on repo/org/user/admin pages, but the admin page only shows history for system hooks it can edit; default hooks never have history. Is there a need for an instance-wide delivery log?
9. Should failed deliveries be retried automatically with backoff? Forgejo never retries (one attempt; manual redelivery only).
10. gitea and gogs kinds exist only to speak older receivers' dialects (short ref names on create/delete for gitea, POST-only for gogs). Keep them as distinct kinds in a rebuild?
11. Test deliveries are only offered for repository webhooks; should org, user and system webhooks have a test action too?

## notifications (9)

1. Inbox and mail audiences differ in Forgejo: mail adds the poster and assignees and uses the *issues* watch selection even for pull requests, while the inbox uses the kind-specific selection and omits assignees. Should a rebuild unify both into one audience definition?
2. Forgejo emits the notification for a new pull request with every receiver named explicitly, which skips the unsubscribe and blocked-author filters; is that intended?
3. Reply/unsubscribe tokens are stateless and become invalid when the holder's secret is rotated (e.g. on password change). Should a rebuild revoke outstanding tokens on password change explicitly?
4. Forgejo sends the Actions job-failure mail from three hooks (attempt, status change, completion) and can mail the same failure more than once; should exactly one mail per failed job attempt be guaranteed?
5. Collaborator-added mail is sent only when a collaborator is added through the web UI, not through the API. Should the API path mail too?
6. Feed entries are copied per watcher at the time of the event and never re-evaluated when watching stops or permissions change (visibility is re-checked on read). Should a rebuild compute feeds on read instead?
7. The `commit` notification source is declared but never produced. Keep it for future commit-comment notifications or drop it?
8. The release mail is sent only for non-prerelease releases while the activity feed records prereleases too. Intended?
9. Failed mail is dropped without retry. Should a rebuild retry transient transport failures?

## administration (10)

1. Running a cron task via the API records the scheduler, not the administrator, as the doer (web records the admin). Should a rebuild always record the admin?
2. app.ini documents [cron] ENABLED / RUN_AT_START as global switches but the code ignores them; should a rebuild honour a global cron on/off?
3. There is no maintenance / read-only mode. Should a rebuild provide one (block writes, show banner) for upgrades and backups?
4. `forgejo dump` has no counterpart restore command; should a rebuild offer an instance-level restore, or keep restore manual?
5. Only one announcement kind (modal) exists. Is a non-blocking banner kind, or scheduling (start/end), wanted?
6. Admins may edit their own account in ways that remove their admin status only if another admin exists; may an admin restrict themselves (is_restricted) without that guard?
7. Unadopted-repository listing is instance-wide for admins; users with ALLOW_ADOPTION_OF_UNADOPTED_REPOSITORIES can adopt their own — is that capability repositories' or ours?
8. Should the update checker default to on? It contacts a third-party endpoint (DNS release.forgejo.org) weekly, which offline-minded installs may not expect.
9. Should doctor checks also be runnable from the web admin (Forgejo exposes them only via CLI)?
10. migrations asks for a repository notice on ImportedContentFlagged(migration, detail) as well as on MigrationFailed; only the latter is recorded here — should flagged content also raise a notice?

## quota (9)

1. Deny when uncovered: a holder whose (explicit or default) groups contain no rule covering the checked subject is denied, while a holder with no groups at all falls back to an unlimited size:all rule. Intentional deny-by-default, or should an uncovered subject be allowed?
2. Forgejo exempts pushes to PROTECTED branches from the quota check (only unprotected branches, tags and other refs are gated). This looks accidental; the spec gates every non-deleting push. Confirm.
3. The check uses current usage only (strictly below the limit), so a single upload can overshoot a limit arbitrarily. Should the incoming size be counted (usage + size <= limit)?
4. Wiki storage is never measured (always 0); size:wiki rules can never deny by themselves. Should wiki git size be measured, and should it then also count under size:git:all rather than size:assets:all?
5. Overlapping subjects in one rule (e.g. size:all and size:git:all) are summed twice. Should a rule's usage be the union instead?
6. Web 'accept transfer' charges the accepting user (doer) while the API charges the recipient (which may be an organization). The spec charges the recipient.
7. Manual mirror sync is gated on size:repos:all in the web UI but on size:git:all in the API. The spec uses size:repos:all for both.
8. Several gated API file operations include DELETE of a file, which can only shrink a repository. Should deletions be exempt as they are for git push?
9. Quota group sync from login sources exists only for OAuth2 sources; should LDAP group membership also map to quota groups?

## moderation (9)

1. Forgejo does not check that the admin's content action matches the reported kind (e.g. 'delete repository' on a user report deletes the repository with the same numeric id). The spec requires the kind to match; confirm.
2. Resolving marks ALL reports on the item (including already-resolved ones, re-stamping resolved_at). The spec resolves only open reports; confirm.
3. The purge deletes a shadow copy together with any resolved report linked to it, even when open reports still share that copy. The spec keeps a copy while any report references it; confirm.
4. Reporting is allowed server-side for one's own repository, issue or comment (only the UI hides the entry). Should the server refuse these too?
5. Duplicate reports by the same reporter on the same open item are silently dropped with a 'thank you' message. Should the reporter be told instead?
6. Organizations get no shadow copy before deletion (only user deletion snapshots). Should org deletion preserve evidence too?
7. There is no notification to admins when a report arrives and no API. Should a rebuild add either?
8. Should reporters be told when their report is resolved?
9. Forgejo ships the purge task disabled ([cron.remove_resolved_reports] ENABLED=false), so resolved reports and their evidence are kept forever unless an admin turns it on. The spec purges after keep_resolved_reports_for; should a rebuild keep the separate task switch?

## federation (10)

1. Signer vs actor: inbound activities are accepted when the HTTP signature verifies with ANY known key; Forgejo never checks that the signer is the activity's actor (or that actor's host). Should the rebuild bind them?
2. Visibility: Person and Repository actors (and the repository Like inbox) are served by numeric id with no visibility check, so private users, organisations and private repositories are exposed and can be starred remotely. Should these require public visibility?
3. Person outbox exposes the notes a user RECEIVED (their federated feed) to any signed peer. Should the outbox instead list the user's own public activities (which are already fetchable individually)?
4. Create(Note) is accepted from any known federated user, whether or not the recipient follows them, and outbound follows are never recorded. Should notes be limited to actors the recipient follows (and follows be recorded, Accept moving them to accepted)?
5. Federated unstar: a remote Undo(Like) is not accepted and a local unstar sends nothing; remote stars can therefore never be withdrawn. Implement Undo(Like) both ways?
6. Outbound Create(Note) is pushed once per watcher copy of an activity in Forgejo (duplicates). The spec delivers once per action per follower; confirm.
7. Only the first max_following_repos_liked (10) following repos receive Likes, but the settings form stores any number up to the 2048-character limit. Limit the list on write instead?
8. Outbound Likes are sent synchronously once (max_attempts 1) while other activities retry up to 10 times through the queue. Should Likes use the retrying queue too, and should retries back off?
9. Per-host replay protection (latest_activity) is shared by all actors on a host, so out-of-order Likes from two different users of one host are dropped. Track per actor instead?
10. Remote people are materialised as full accounts/User rows (kind activitypub). Should they be excluded from user counts such as NodeInfo users_total and the user explore page?

## discovery (8)

1. go-get answers reveal a private repository's default branch name to anyone; should the answer for unreadable repositories use the instance default branch instead?
2. User search hides private accounts from everyone but admins/self/org co-members, while profile visibility also admits followers of a private account; should the two rules be unified?
3. The API user search is disabled whenever the explore users page is disabled, which also breaks pickers that use the API; should the API stay available to signed-in users?
4. An indexer that does not become ready within STARTUP_TIMEOUT terminates the instance; should a rebuild degrade to 'search unavailable' instead?
5. Instance-wide code search for site admins searches every repository, including private ones they have no role in; intended?
6. Explore orders by recent update by default but profile tabs fall back to the explore default too; should profiles have their own default sort?
7. Only conversation comments are indexed for issue search; should review (code) comments be searchable too?
8. The relevant-only filter treats 'has avatar' as relevance; keep that heuristic?

## migrations (12)

1. Web migrations run asynchronously as a task; POST /repos/migrate runs synchronously and deletes the repository on failure. The rebuild queues both; should the API instead wait (with a timeout) and return the finished repository?
2. Org targets: the web form accepts any member allowed to create org repositories, the API requires an org owner. This spec uses 'may create repositories in the org' for both.
3. Forgejo only cancels a running migration; cancelling a queued one is a no-op. This spec allows cancelling queued ones too.
4. Forgejo silently clears issue/PR/release selections on mirror imports; this spec refuses such a request instead. Confirm.
5. Re-attribution matches on source KIND plus external id only, so two different GitLab or Gitea hosts can collide on the same numeric id, and an OAuth source must be NAMED exactly like the forge kind. Should matching use the source host as well?
6. Only numeric external ids are re-attributed. Keep that restriction?
7. Restore ignores the archive's is_private flag and always creates a public repository (unless FORCE_PRIVATE). Should restore keep the archive's visibility?
8. dump-repo does not apply the allowed/blocked-hosts policy (it runs as an operator). Should it?
9. Imported merged pull requests record the migration doer as merger; should the merger be left empty / ghost instead?
10. Milestones without a deadline are stored with a far-future deadline (9999-01-01) in Forgejo; this spec keeps the deadline absent.
11. Assignees are carried by the source data but never imported. Import them when they resolve to local accounts?
12. Should repository-wide retry also be offered to the requester when they are not a repository admin (e.g. a site admin migrated into someone else's namespace)?
