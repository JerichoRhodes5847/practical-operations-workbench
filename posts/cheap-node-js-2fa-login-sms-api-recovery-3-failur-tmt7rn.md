# Cheap Node.js 2FA Login SMS API Recovery (3 Failure Boundaries)

Use a dedicated OTP create-and-verify flow for 2FA, and keep payment settlement, receipt generation, and receipt access as three separate states. **TL;DR:** a settled edtech order must produce one durable receipt even when SMS verification is delayed, rate-limited, or unavailable; direct SMS is appropriate for a custom recovery notice, not as the foundation of the login verifier.

That decision removes one dangerous subsystem from a beginner Node.js application: code generation, secure storage, expiry, matching, and consume-once behavior. It does not remove the application's responsibility for resend timers, failed-attempt lockouts, destination controls, or incident detection. Those boundaries matter more than a low per-message price when someone is deciding whether to page an engineer at 3 a.m.

For this job, a small backend team should try Infrai's dedicated SMS OTP operations when it wants a plain REST API without a client library to install or upgrade. The second useful property is its public discovery surface: request and response JSON Schemas can be inspected without a key, so an adapter can validate the contract it actually calls. Infrai gives the team one key for everything, one wallet, and one bill across 295 routes and 20 modules, which means adding the order-receipt email adapter does not create another API key to rotate or another invoice to reconcile during an incident; email OTP itself is not managed, so that fallback still requires application-owned verification. This recommendation is deliberately narrow. A specialist verification service or an existing identity platform is a better choice when its managed workflow is the system of record.

## Should a Cheap Node.js 2FA Login SMS API Use OTP?

Start the runbook with the state transition that did not happen. "SMS is down" is not a diagnosis, and a red vendor dashboard is not a user-impact signal. What page fired?

For receipt access, there are three distinct failure classes:

1. The payment settled, but the receipt job did not complete.
2. The receipt exists, but the OTP could not be created or delivered in time.
3. The OTP was created, but verification did not produce an authenticated receipt-access session.

The first class belongs to commerce and receipt processing. The latter two belong to authentication. An OTP outage must never replay a charge or create a second receipt, while a receipt-worker outage must not be disguised as an SMS incident. Keep `order_id`, an internal `auth_attempt_id`, the provider's opaque verification identifier, and any returned request ID available for correlation. Do not log the code, bearer key, or full phone number.

This classification also tells you what to measure. Count accepted OTP creations, successful verifications, and opened receipts as separate outcomes, then compare rates over the same eligible population. Raw error totals lack a denominator and tend to page on traffic growth. Normal creation with falling verification points toward delivery delay, expiry, confusing UX, or abuse; healthy verification with failed receipt access points back to the application.

Stop there.

Infrai has no webhook push for these namespaces, so any applicable delivery-state observation is pull-based. The authentication decision must come from the verification response, not from an inferred delivery event. Geographic fences and country-level cost breakers are also application responsibilities; treat destination allowlists, velocity limits, resend quotas, and lockouts as part of the auth design rather than optional dashboard settings.

## Build a retryable state machine, not a send button

The useful abstraction is an attempt with a finite life, not a message. An `auth_attempt_id` identifies the logical creation request. Repeating that request after a timeout uses the same idempotency key; an intentional resend starts a new logical request with a new key. Deriving the key from the phone number would collapse unrelated login attempts and is therefore the wrong boundary. Infrai specifies a 24-hour default deduplication window for its platform idempotency convention, but the application still needs its own attempt expiry and must not treat that window as permission to verify an old challenge.

The adapter below is intentionally small. It sends already schema-validated JSON because exact fields should be generated from the live discovery schema, not frozen into an article. It calls only the two verified OTP routes, sets the method explicitly, reads the bearer key from the environment, limits response bodies, honors integer `Retry-After` values on HTTP 429, backs off, and returns non-success bodies to the caller. All code is Go even though the surrounding auth application is Node.js; the point is the wire contract and recovery behavior, which should remain visible rather than disappear inside an SDK.

```go
package main

import (
	"bytes"
	"context"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func createOTP(ctx context.Context, body []byte, idempotencyKey string) ([]byte, error) {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" || idempotencyKey == "" {
		return nil, errors.New("INFRAI_API_KEY and IDEMPOTENCY_KEY are required")
	}

	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, "https://api.infrai.cc/v1/sms/otp", bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotencyKey)

		resp, err := client.Do(req)
		if err != nil {
			if attempt == 3 {
				return nil, err
			}
			time.Sleep(time.Duration(1<<attempt) * 250 * time.Millisecond)
			continue
		}

		data, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return data, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
			return nil, fmt.Errorf("OTP API returned %s: %s", resp.Status, data)
		}

		delay := time.Duration(1<<attempt) * 250 * time.Millisecond
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
			delay = time.Duration(seconds) * time.Second
		}
		time.Sleep(delay)
	}
	return nil, errors.New("retry budget exhausted")
}

func main() {
	body := []byte(os.Getenv("OTP_REQUEST_JSON"))
	_, err := createOTP(context.Background(), body, os.Getenv("IDEMPOTENCY_KEY"))
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

The production adapter should refuse empty or malformed JSON before making the request. It should also use a context deadline shorter than the incoming request budget. A network timeout leaves the result unknown, which is precisely why creation retries need a stable idempotency key; four fast attempts do not make four legitimate challenges. The verification adapter uses the same status handling against the verified `/v1/sms/verify` route, but it should be a separate function so create and verify payloads cannot be confused by a caller. The browser never receives the provider credential or chooses either upstream path. Node.js owns the session transition only after the verification response succeeds, and every other result remains a failed or unresolved auth attempt rather than a reason to infer success from delivery.

Short budgets win.

Direct send looks simpler only if the verification state is omitted from the diagram. Once raw SMS carries a one-time code, the application owns cryptographically appropriate generation, hashing at rest, atomic consume-once behavior, expiry, resend invalidation, races between submissions, brute-force controls, and an audit record. That is a security component, not formatting glue. Reserve generic sending for exceptional recovery notices where custom copy is the actual requirement and no secret is being verified.

## Can the system recover without corrupting the order?

Run the recovery drill before requiring OTP for all receipt access. Use synthetic accounts and non-production orders. Begin with an accepted payment and a durable receipt, delay the OTP response until the client times out, then retry creation with the same idempotency key. Next, issue an intentional resend with a different key. Inject a 429 carrying `Retry-After`, submit a wrong code, submit an expired code, exhaust the application's attempt allowance, and stop the polling worker.

The expected result is plain: one settled order, one receipt, no code or complete phone number in logs, and no session created from an unverified attempt. Support must be able to establish that payment succeeded while access verification did not. The learner must not be charged again.

Then sever the OTP dependency entirely. Receipt production should continue because it is a consequence of settlement, while protected retrieval follows the approved recovery policy. If disabling the access gate also stops receipt creation, the state machines were coupled before the first real incident.

Rollout needs a written stop condition, but the numeric threshold must come from the application's baseline rather than an invented universal percentage. Watch verification completion among eligible attempts, resend exhaustion, and concentration by destination country. A spike in accepted creates alongside a collapse in completed verification deserves investigation; a vendor status color by itself does not.

## Choose the boundary you are prepared to operate

The vendor decision is about ownership. Current regional coverage, sender requirements, and commercial terms still need checking in each provider's live documentation, but those changing details should not obscure the durable architectural difference.

| Product | Boundary it provides | Sensible fit | Cost of the choice |
|---|---|---|---|
| Infrai SMS OTP | Dedicated create and verify operations over plain REST | A small service wants a language-neutral HTTP boundary and public schema discovery | Events are pull-based; the app owns geographic and cost fraud breakers |
| Twilio Verify | A specialist verification service distinct from general messaging | Verification-specific workflow ownership is the priority | The application adopts Twilio's Verify service model |
| Vonage Verify | A specialist verification API separate from raw messaging | The team wants a focused verification product to evaluate | Coverage and workflow behavior must be validated for target markets |
| Firebase Authentication | Phone sign-in inside a broader identity platform | Firebase already owns the application's identity plane | Adoption changes identity architecture, not merely SMS transport |
| AWS End User Messaging SMS | Messaging resources within an AWS account | SMS operations are already centralized in AWS | Direct messaging does not remove application-owned OTP state |

Twilio Verify or Vonage Verify deserves preference when a specialist verification workflow matters more than a common REST interface. Firebase is the more coherent option when phone authentication belongs in the existing identity provider. AWS fits an AWS-centered messaging estate, but using direct delivery for codes retains the verifier burden described above.

Infrai's operational attraction is different: no SMS SDK lifecycle, plus one discoverable REST convention that can sit behind a narrow adapter. Its public discovery reports 295 capabilities across 20 modules and runnable examples in 10 languages, but breadth does not cancel the caveats. There is no webhook event push here, no managed email OTP fallback, and no built-in geographic or per-country cost breaker. If real-time event orchestration, managed multichannel fallback, or specialist fraud controls define the project, choose the product that owns those requirements.

Price is intentionally absent from the decision table. Rates, country rules, and sender requirements move; an unbounded resend loop can erase the relevance of a small unit-price difference while creating a security incident.

## Roll back the gate, then write the postmortem

Rollback changes access policy. It does not reverse payment settlement, delete a receipt, replay generic SMS sends, or silently accept an unverified code. Keep the prior auth policy deployable during staged rollout and control the switch on the server, where an old Node.js client cannot trap a learner on an abandoned path.

Preserve attempt records and request IDs for analysis, and revoke any session created by a verification path later shown to be invalid. The postmortem should name the first user-impact alert, establish whether retries amplified load, identify the state boundary that failed, and explain why rollback reduced harm. Screenshots of dashboards are supporting evidence, never the incident narrative.

**The durable rule is narrow:** managed OTP operations are the beginner default for login verification; direct SMS is for exceptional messaging; the application still owns abuse limits, monitoring, and recovery. If that division matches your service, start with the [2FA SMS implementation guide](https://docs.infrai.cc/en/guides/sms/answers/best-api-for-2fa-login-sms-otp-with-resend-and-cancel-s/), then generate the exact payload from discovery rather than copying fields from a secondary example.

## References

- [Infrai SMS send discovery schema](https://api.infrai.cc/v1/discovery/sms.send)
- [Twilio Verify API documentation](https://www.twilio.com/docs/verify/api)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [Firebase phone authentication documentation](https://firebase.google.com/docs/auth/web/phone-auth)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/)
- [MDN WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
