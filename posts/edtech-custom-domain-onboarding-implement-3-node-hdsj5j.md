# Edtech Custom Domain Onboarding: Implement 3 Node.js Zone Records Safely

The least complex reliable option is to treat SPF, DKIM, and DMARC onboarding as asynchronous reconciliation: store the records you expect, let the customer or platform publish them according to zone ownership, then verify authoritative DNS before enabling authenticated mail. Do not hold an HTTP request open waiting for DNS.

**TL;DR:** A Node.js API should record desired state and enqueue a check. A worker observes DNS on a bounded schedule, and the domain advances only after it sees the exact SPF authorization, DKIM public key, and DMARC policy requested. Customer-owned zones get instructions; platform-owned zones may get idempotent writes. Both paths require independent verification.

Imagine the page: `mail_authentication_deadline_breached` for `learn.example.edu`, with `ownership=customer`, `missing=dkim`, `attempts=18`, and the last observation attached. That gives the on-call an affected domain, a failed control, and a next action. A dashboard saying "DNS verification success rate is 94%" gives none of those things. My first question is what page fired, because a calm aggregate can conceal one school whose password resets are failing authentication.

## How should you implement custom domain onboarding for a customer-owned zone?

A page should represent impact or an imminent breach of a promise. During onboarding, DNS absence is expected: the customer may need an administrator to approve a change, recursive caches may retain an earlier answer until its TTL expires, and negative answers can also be cached. Poll failures belong in logs and metrics. They should not all become alerts.

The earlier signal is a stuck state measured against an application deadline, not inferred from one resolver error. Record `requested_at`, `next_check_at`, `last_checked_at`, `last_observation`, and the controls still missing. Suppose the school has published SPF and DMARC but the selector name still returns `NXDOMAIN`. The event should say that DKIM alone is absent, show which name was queried, identify the zone as customer-owned, and schedule another observation; it should not flatten the result into `verification_failed`. If the activation deadline then passes, the ticket or page carries the accumulated evidence instead of forcing the responder to repeat every lookup. If the domain has already been permitted to send, the severity changes because unauthenticated production mail is now plausible. If it has never been enabled and no commitment has expired, the same DNS response remains ordinary pending work. This state-aware split is the instrumentation change that should happen before anyone adjusts an alert threshold. Warn the owning team before a scheduled class launch; page when sending is enabled without verified controls, or when a committed activation deadline has passed. If every `NXDOMAIN` wakes somebody, responders learn to ignore the pager.

DMARC adds another reason to reject a single green check. RFC 7489 evaluates identifier alignment: SPF and DKIM can produce authentication results while DMARC still fails when neither authenticated identifier aligns with the visible From domain. DNS verification confirms publication. A separate mail-path test confirms alignment.

Short version: presence is not delivery proof.

Context changes severity.

## Separate ownership before writing anything

The ownership choice determines who may mutate DNS and who can recover from a mistake. Make it domain data, not a hidden branch inside a provider adapter.

| Zone model | Application action | Verification action | Primary operational risk |
| --- | --- | --- | --- |
| Customer-owned | Generate exact records for an authorized administrator | Read DNS until expected values appear | Ambiguous delegation, transcription error, or approval delay |
| Platform-owned | Submit idempotent changes through the zone control plane | Read DNS after the change reports success | Wrong-tenant writes or trusting acceptance too early |
| Delegated subdomain | Manage only the delegated child zone | Check delegation, then child records | Parent delegation changes without notice |

For an edtech platform, a delegated mail subdomain can narrow the blast radius while the school retains its institutional apex. It is a trade-off, not a universal answer: it affects the visible From strategy, the records central IT must approve, and the recovery runbook.

Never request broad DNS credentials merely to automate three records. In a customer-owned zone, return a deterministic record set and observe public DNS. In a platform-owned zone, scope credentials to intended zones, serialize conflicting changes, retain an audit trail, and make retries idempotent. A control-plane success response means a change was accepted. It does not prove what resolvers can observe.

I would choose narrower authority, even when it adds a delegation step, because a slower onboarding is recoverable and a cross-tenant DNS write is not.

## Build a reconciler, not a waiting request

Keep the state machine small: `pending`, `checking`, `verified`, and `attention_required`. Store desired records separately from observations; overwriting intent with the latest answer destroys evidence needed during an incident. A generation number prevents an old job from approving a domain after its DKIM selector or ownership mode changes.

The Node.js edge only needs to persist intent and enqueue the domain identifier plus generation. The worker owns DNS reads and conditional state transitions. The core boundary below is Go because keeping the verifier behind interfaces makes its failure behavior explicit.

```go
package onboarding

import (
	"context"
	"time"
)

type Desired struct {
	Domain, SPFToken, DKIMName, DKIMValue, DMARCName, DMARCValue string
	Generation int64
}

type Observation struct {
	CheckedAt time.Time
	SPF, DKIM, DMARC bool
	Errors map[string]string
}

type Resolver interface {
	TXT(context.Context, string) ([]string, error)
}

type Store interface {
	LoadDesired(context.Context, string) (Desired, error)
	SaveIfCurrent(context.Context, string, int64, Observation) error
}

func Reconcile(ctx context.Context, r Resolver, s Store, id string, now time.Time) error {
	want, err := s.LoadDesired(ctx, id)
	if err != nil { return err }

	root, rootErr := r.TXT(ctx, want.Domain)
	dkim, dkimErr := r.TXT(ctx, want.DKIMName)
	dmarc, dmarcErr := r.TXT(ctx, want.DMARCName)
	obs := Observation{
		CheckedAt: now,
		SPF: validSPF(root, want.SPFToken),
		DKIM: validDKIM(dkim, want.DKIMValue),
		DMARC: validDMARC(dmarc, want.DMARCValue),
		Errors: map[string]string{},
	}
	if rootErr != nil { obs.Errors["spf"] = rootErr.Error() }
	if dkimErr != nil { obs.Errors["dkim"] = dkimErr.Error() }
	if dmarcErr != nil { obs.Errors["dmarc"] = dmarcErr.Error() }
	return s.SaveIfCurrent(ctx, id, want.Generation, obs)
}
```

The three validators are intentionally record-specific. DNS libraries may expose a long TXT value as multiple character strings, while SPF mechanisms and DMARC tag lists need syntax-aware handling. Reconstruct TXT RDATA according to the resolver library's contract, parse the applicable grammar, and compare normalized semantics only where the standard permits it. A substring search can approve the wrong record.

Use exponential backoff with jitter, cap concurrency per authoritative server, and stop aggressive polling after the onboarding deadline. Retrying every few seconds does not make a customer's approval happen faster. Respect the published TTL when estimating when a changed cached answer may be replaced, but do not promise that every recursive resolver updates simultaneously.

## Instrument the transition that can fail

Emit one structured event per attempt with a bounded result such as `verified`, `incomplete`, `temporary_error`, or `permanent_configuration_error`. Keep domain and tenant identifiers in logs or traces rather than metric labels; putting them in labels creates unbounded cardinality when an incident expands.

Measure requested-to-verified time, counts by ownership and missing control, attempts per onboarding, stale jobs rejected by generation, and enabled domains that later lose a required record. Attach the latest answers, observation time, expected names, ownership, deadline, and runbook to an actionable alert. Never attach DKIM private key material; DNS publishes the public key while the signer retains the private key.

Classify failures. `NXDOMAIN` for a selector differs from a timeout, `SERVFAIL`, malformed DMARC, or an SPF record that exists but does not authorize the sending path. Preserve the raw response for diagnosis, then map it to a stable reason code. Without that split, one resolver outage can look like thousands of customers deleting records.

Do not page on that illusion.

After activation, recheck verified domains at a slower cadence. Records disappear during unrelated DNS work. Synthetic messages through each supported sending path should inspect authentication results because this is where alignment errors surface; aggregate DMARC reports add evidence about real traffic, but arrive outside the synchronous onboarding path and should not gate an immediate verification request.

## Thresholds spend attention

Page on the first missing lookup and every customer-side delay becomes an incident. Wait without a deadline and the first signal may be a teacher reporting absent account invitations. Use different timers for customer action, propagation observation, and post-verification regression, then tune them from observed distributions and the service promise. No standard defines one universal number of minutes for every zone.

Start with tickets for incomplete onboarding and pages for unsafe sending or missed commitments. Review false positives as seriously as missed pages: determine whether the resolver view was stale, the ownership state was wrong, or the alert grouped unrelated domains. The useful alert names the broken transition and the party able to act.

**The decision rule is plain:** customer-owned zones receive precise instructions plus independent verification; platform-owned zones receive scoped, idempotent writes plus the same independent verification. Neither path is complete until SPF, DKIM, and DMARC publication is checked and the real mail path demonstrates alignment.

## References

- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
- RFC 6376, DomainKeys Identified Mail Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- RFC 1034, Domain Names Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 2308, Negative Caching of DNS Queries: https://datatracker.ietf.org/doc/html/rfc2308
