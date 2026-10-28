# Go Password Reset Emails: 7 Security Fields for B2B SaaS

A secure password reset email needs a narrowly scoped action, a short-lived single-use token, a clear expiry, and enough context for the recipient to recognize an unexpected request. Treat those fields as a security boundary, then deliver the message with the same discipline you would apply to a payment-settled order receipt: create it from durable state, make retries safe, and record what was sent without recording the secret.

The useful test is not whether the template looks complete. It is what page fired at 3 a.m. when a reset was requested for an account that did not ask for one. A polished template can still leak whether an address exists, accept a replayed link, or send a token through a channel whose authentication was never checked.

## Which fields belong in a password reset email?

The message should contain seven functional pieces, although some may be represented by one link rather than separate visible fields:

1. The account identifier, usually the email address or a masked form, so the recipient can tell which account is affected without exposing unnecessary data.
2. A single reset action that points to your application’s HTTPS origin.
3. A random, opaque, single-use token carried by that action.
4. An explicit expiry time or short validity window.
5. A brief instruction to ignore the message if the request was not theirs.
6. A support or security contact that does not ask the recipient to disclose the token.
7. A recognizable sender identity whose domain authentication is configured and monitored.

The visible copy should never contain a password, a security answer, or a token that can be pasted into a support ticket. The token belongs in the link, and the server must decide whether it is valid; the browser is only carrying an untrusted value.

Do not confuse a complete template with a complete recovery system.

This is where DKIM helps and where it does not. DKIM lets a receiving system verify a cryptographic signature associated with the message’s signing domain, but it does not make a reset token short-lived, single-use, or bound to the correct account. Authentication of the message and authorization of the reset are separate controls.

## How should the reset link survive retries and replay?

Generate the token with a cryptographically secure random source, store only a digest or otherwise protected representation, and associate it with the account, purpose, creation time, expiry, and consumed state. On use, compare the presented value against the stored representation, consume it in the same transaction that begins the reset flow, and reject a second use. A password reset email is a command envelope, not proof that the person holding the inbox is trustworthy forever.

Here is the shape I want the application boundary to enforce. It is deliberately a generic interface, because the important decision is the state transition rather than a particular mail provider.

```go
type ResetRequest struct {
	AccountID string
	Token    string
	Expires  time.Time
	Used     bool
}

func consumeReset(ctx context.Context, store Store, accountID, token string, now time.Time) error {
	return store.ConsumeReset(ctx, ResetRequest{
		AccountID: accountID,
		Token:     token,
		Expires:   now.Add(15 * time.Minute),
	})
}
```

That snippet is a contract sketch, not a complete implementation: the store must verify a protected token representation and perform the check-and-consume atomically. The 15-minute value is an explicit example of a short window, not a universal rule; choose a window that matches the account’s risk and recovery process, then test the boundary at expiry.

The trade-off is real: a shorter window limits replay exposure but creates more support work for people who cannot reach their inbox promptly. A longer window is friendlier to delayed delivery and less forgiving of a leaked link. The right decision depends on the account’s recovery alternatives, not on a template convention.

A common operational mistake is to enqueue the same reset email again after a worker timeout without an idempotency key. The user receives two valid links, or the later database write invalidates the earlier message while both appear current. Give the reset request a stable event identifier, make message creation idempotent, and keep delivery retry state separate from token state. The same distinction matters for an order receipt after payment settles: retrying delivery must not settle the payment twice.

Retries are not authorization.

Consider the failure sequence carefully. The reset request is accepted, the token record is committed, and the outbound worker times out after handing the message to its transport. A naive retry creates a second message and may create a second token, so the recipient cannot tell which link is authoritative. The opposite implementation is also bad: the worker marks the first attempt as failed, generates another token, and leaves the first link usable because consumption was never tied to the account’s current reset state. Both paths turn a delivery ambiguity into a security ambiguity. A stable request identifier lets the producer recognize the same logical message; a token record with a single purpose and an atomic consume operation lets the application decide which use is valid. The receipt workflow has a related but different invariant: a payment event can be retried for delivery, while the settlement event itself must remain idempotent. Keeping those state machines separate makes the rollback decision legible during an incident.

There is a boundary here. Email is convenient recovery transport, but it is not a substitute for stronger account authentication; high-risk changes may need an additional factor or a separate support procedure.

## What should the recipient learn from an unexpected message?

The email should say what happened without confirming more than the product is willing to reveal. If the public request endpoint always returns the same response for known and unknown addresses, the email layer should preserve that boundary; otherwise an attacker can turn delivery behavior into an account-enumeration oracle.

The copy needs a plain failure path: ignore the message if the recipient did not request it, use the support route for suspected abuse, and never forward the link. The reset page should show the same account context after the token is accepted, not put sensitive account data into the URL or subject line. Subject lines and preview text are copied into notifications, search indexes, and support screenshots more often than teams expect.

Keep the URL boring. Use HTTPS, avoid putting an email address in query parameters when an opaque identifier is sufficient, and make the reset page refuse tokens for the wrong purpose. A token minted for a password reset must not be accepted by an email-change or invitation endpoint merely because the strings share a format.

## How do you verify the path before it pages someone?

Test the whole path as an operational workflow: request a reset, inspect the generated message in a controlled mailbox, follow the link once, follow it again, wait past expiry, and request two resets close together. Assert that the old token is rejected according to the chosen policy, that the new message is not silently dropped, and that logs contain the request identifier rather than the token or full reset URL.

For the payment-settled receipt path, the useful dashboard is not a count of successful sends. I want a trace from the durable payment event to the outbound message, with separate states for accepted, deferred, rejected, and permanently failed delivery. A reset flow deserves the same evidence: request identifier, account reference with controlled redaction, template version, token outcome, and provider response category. Never make the dashboard a substitute for the question of which alert actually fired.

The limitation of this method is operational, not cosmetic: an authenticated sender domain cannot repair a compromised mailbox, and delivery telemetry cannot prove that a human recognized the message. Keep those assumptions visible in the threat model.

Roll back the template or worker independently from the token rules. If a new template renders incorrectly, disable that template version and replay only messages whose delivery state is retryable. Do not recreate tokens just because a delivery worker was redeployed, and do not bulk-retry a queue until duplicate sends and idempotency behavior have been checked.

The decision rule is straightforward: expose enough information for a legitimate recipient to recognize and complete the action, keep the secret opaque and disposable, authenticate the message domain, and make every retry observable without making it a second authorization path.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://pages.nist.gov/800-63-4/sp800-63b.html
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
