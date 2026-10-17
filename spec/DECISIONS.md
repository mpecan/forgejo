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
