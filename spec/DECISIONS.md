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
