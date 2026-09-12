# Production Tenant API Key Rotation: Grace Periods, Secret Stores, Auditable Deploys

Short answer: rotate a tenant's production key by issuing the replacement first, distributing it through a secret store, deploying consumers, and revoking the old key only after telemetry proves the overlap is quiet. The grace period is an explicit control with an owner and an expiry, not a hopeful sleep in a deploy script.

At 03:00, the page that matters is not “key rotation started.” It is “tenant 1842's media ingest requests are being rejected.” That page should carry the key identifier, tenant, region, and deploy revision. Without those fields, an on-call engineer is left guessing whether a revoked credential, a stale secret, or a provider-side policy caused the first 401.

The useful trace runs backwards from that page: request rejection, key state, secret version, rollout event, then the rotation command that created the replacement. Each link needs an audit record. Dashboards are evidence, not a plan.

## How should you rotate a production API key without downtime?

For a media platform, the unit of change is a scoped key per tenant, not one global token. A rotation record should include a non-secret key ID, tenant ID, actor, reason, creation time, expiry, and the event that revoked its predecessor. Never put the key value in logs, traces, tickets, or deployment metadata. OWASP's Secrets Management Cheat Sheet also calls for limiting secret exposure, controlling access, and keeping lifecycle events observable.

The overlap has two states. During `issued`, both old and new IDs can authenticate. During `retiring`, the old ID is still accepted for a bounded window while every consumer is expected to reload the new value. After the deadline, `revoked` means the old ID is rejected. A state machine makes an emergency review possible; a shell variable named `GRACE_SECONDS` does not.

No silent expiry.

A practical rotation ledger looks like this:

| Event | Required evidence | Stop condition |
| --- | --- | --- |
| Issue | new key ID, tenant, scope, actor | scope differs from requested scope |
| Distribute | secret version and target workload | any workload cannot read the version |
| Deploy | revision, start time, reload result | old ID still used after rollout |
| Observe | accepted requests by key ID and tenant | unexplained 401/403 or missing telemetry |
| Revoke | old key ID, actor, timestamp | rollback owner is not named |

The stop conditions are deliberately conservative. A rotation that cannot be explained six weeks later is an access-control incident waiting for a calendar invite.

## How do grace periods, secret stores, and deploys fit together?

Treat the secret store as the source of truth and the application as a cache with a refresh contract. The deploy should reference a version or alias, never copy a raw value into an image. A process can load the current version at startup and refresh on a signal or short, bounded interval; the exact mechanism depends on the runtime, but the audit event must say which version was observed.

Here is a small Go sketch of the decision boundary. The store and issuer are interfaces so the policy can be tested without a live secret service.

```go
package rotation

import "context"

type Key struct {
	ID     string
	Version string
}

type Issuer interface {
	Issue(ctx context.Context, tenant, scope string) (Key, error)
	Revoke(ctx context.Context, keyID string) error
}

type SecretStore interface {
	Put(ctx context.Context, name, value string) (version string, err error)
}

type Audit interface {
	Record(ctx context.Context, event string, fields map[string]string) error
}

// Rotate records intent before distribution, making a partial rollout visible.
func Rotate(ctx context.Context, issuer Issuer, store SecretStore, audit Audit, tenant, scope, secretName, value string) error {
	key, err := issuer.Issue(ctx, tenant, scope)
	if err != nil {
		return err
	}
	if err := audit.Record(ctx, "key.issued", map[string]string{"tenant": tenant, "key_id": key.ID, "scope": scope}); err != nil {
		return err
	}
	version, err := store.Put(ctx, secretName, value)
	if err != nil {
		return err
	}
	return audit.Record(ctx, "secret.distributed", map[string]string{"tenant": tenant, "key_id": key.ID, "secret_version": version})
}
```

The example stops before revocation on purpose. Revocation belongs in a separate, authorized step after rollout evidence exists. It also leaves secret retrieval and workload reload policy outside the issuer; coupling those concerns makes it harder to test a failed distribution without accidentally disabling access.

For Node.js consumers, the same contract usually means a startup read plus an atomic in-memory swap when the secret version changes. Do not mutate a shared string while requests are using it. Build a new client or immutable credential object, switch the reference, then let in-flight requests finish. Your mileage may vary with connection pooling, so test the provider client's behavior under a credential swap instead of assuming every keep-alive socket re-authenticates.

## Which failure modes make an overlap unsafe?

The obvious failure is revoking too early. Less obvious: one batch worker never reloads, a canary reads a different secret-store alias, or a retry queue keeps the old credential alive for hours. A grace period that is shorter than the longest retry and job lease is not a grace period; it is delayed downtime.

Another trap is broad scope. If a tenant ingest key can also administer billing, rotation may be operationally smooth while the access boundary is wrong. Issue the narrowest scope the media workflow needs, and make a failed scope check a visible authorization event rather than silently retrying with a more privileged key.

A red 401 graph can look decisive, yet an aggregate that collapses ingest, playback, and billing calls into one line leaves the on-call unable to tell whether a tenant is failing or a noisy neighbor is merely busy. Trace the request path, compare the secret-store version recorded at startup with the version in the deployment manifest, and make a missing reload signal visible. The correction is simple but non-negotiable: emit counters partitioned by tenant and non-secret key ID, attach the deployment revision, and alert on a rate over a short window plus an absolute request count. A single 401 on a low-volume tenant can matter more than a percentage on a high-volume one.

Keep rollback explicit. If the new key is rejected by a downstream policy, pause revocation, restore the previous secret-store version, and record who approved that extension. That is a controlled overlap, not an undocumented workaround. The old key still needs a hard expiry so an emergency pause cannot become permanent access.

## How can teams prove the rotation worked after deploy?

The test plan should exercise the ledger, not just the happy-path HTTP call. In a pre-production tenant, issue two scoped keys, verify both during overlap, deploy a consumer that reloads the secret, revoke the old ID, and assert that old requests fail while new requests continue. Then force a reload timeout and confirm the automation pauses instead of revoking.

At production scale, sample the accepted key ID on every request and retain aggregated counts long enough to cover the maximum job and retry lifetime. Alert on three things separately: old-key use after the expected rollout, new-key rejection, and missing rotation events. Separating them answers the pager question quickly: did the page fire because access was denied, because distribution stopped, or because instrumentation disappeared?

The catch is operational weight. Per-tenant keys, audit retention, and a secret-store integration cost engineering time and can produce noisy telemetry for a small service. This design is not suitable when a single internal process has no tenant boundary and no independent revocation requirement; a simpler identity mechanism may be easier to operate. Stick with scoped rotation when tenant isolation, incident review, or contractual audit evidence matters, and budget for the reload tests and on-call runbook that make the overlap trustworthy.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
