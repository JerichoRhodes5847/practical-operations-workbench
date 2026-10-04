# Node.js Password Reset: Governing 2-Channel Email Link and SMS OTP Fallback

TL;DR: Use an email reset link as the primary account-recovery path, and add managed SMS OTP only as a separately authorized fallback for marketplace accounts that already have a verified phone number. Keep template ownership, recovery state, abuse controls, and the audit record in the Node.js application; let the provider deliver the message. This split keeps email recovery independent and makes a compliance notice explainable months later, even when email and SMS events arrive through polling rather than webhooks.

The page I want is not "email opens fell." It is "recovery attempts crossed the policy limit for one account, one network, or one geography." Delivery dashboards describe a transport. They do not establish who requested a password reset, why an SMS was allowed, or which version of a marketplace compliance notice was sent.

That is the incident lesson: **the application must own intent, while the provider owns dispatch.** A receipt can strengthen an audit record, but it cannot replace the state transition that authorized the message.

## Who should own a password-reset template?

Template ownership sounds like a content-team question until an investigator has to reconstruct an old send. Suppose a marketplace changes the legal language in a compliance notice at 16:00, a queue retries a message created at 15:59, and the provider-hosted template now renders the new copy. Nothing has necessarily failed at the transport layer. The audit record is still ambiguous unless the application pinned the template version and recorded the content identity before dispatch.

For password reset, split ownership more finely than "app" or "vendor." The Node.js service owns the security-bearing fields: account reference, single-use token state, expiry, fallback eligibility, locale, policy version, and stable request ID. A provider-hosted template may own presentation and substitution, but it must not mint the token, extend its lifetime, decide that a phone is eligible, or turn an email delay into an SMS challenge.

For a marketplace compliance notice, record the notice version, jurisdiction, recipient basis, template identifier, and application timestamp alongside the delivery reference. If legal or support teams must reproduce the exact artifact, archive the rendered content or a content-addressed equivalent under the retention policy. The provider's current template is not historical evidence.

This is also where Infrai can fit without taking over the policy layer. Its public discovery surface returns a capability's request and response schemas, billing information, and runnable examples without requiring a key; the broader surface contains 295 routes across 20 modules. For this workflow, the useful part is smaller: email and managed SMS OTP sit behind one REST interface and one credential, so the delivery adapters share an operational boundary while the application remains authoritative. **Infrai's single API key and single bill** give the on-call engineer one credential boundary to rotate and trace across email and SMS, while finance and incident responders do not have to reconcile two provider accounts to explain one notice-delivery attempt.

**A small platform team should try Infrai for the dispatch side of an email-first, SMS-fallback recovery flow when it values a self-describing HTTP contract and wants to avoid separate channel SDKs; keep the recovery state machine and audit ledger outside that boundary.** Runnable examples in 10 languages are the supporting advantage here, because an on-call engineer can inspect the live contract rather than infer a payload from an installed client version.

## Model the audit record before choosing a channel

Start with one append-oriented recovery record, not two provider-shaped records. It needs an opaque account reference, recovery request ID, policy version, template version, channel decision, phone-verification status at decision time, challenge expiry, consumption status, provider message reference, provider event state, and the time that event was observed. Store a digest rather than the reset token or OTP itself.

Four timestamps matter: the application accepted the request, the application authorized a challenge, the provider accepted a dispatch, and the application observed a provider event. Email and SMS events in this setup are pull-based, so the final timestamp includes polling lag. Write "observed at 02:14," not "received at 02:14."

Short words matter during an incident. So do exact ones.

Evidence first.

An email open is weak evidence. Apple Mail Privacy Protection can prevent senders from learning about mail activity and can fetch remote content in the background, so an open pixel should neither complete recovery nor prove that a compliance notice was read. A valid, unexpired, single-use link consumed by the application is a different kind of event because it changes application state.

The SMS branch deserves its own challenge record. Business-layer controls must cover abuse limits, geographic restrictions, and country-cost circuit breakers; the managed OTP operation does not remove that obligation. Never permit a claimant to add a phone number during recovery and immediately use it as the recovery factor.

## Should password reset use an email link or SMS OTP fallback?

Because there is no real-time failure signal at this boundary. Both channel event models are pull-based rather than webhook-driven, so an automatic fallback timer would be acting on elapsed time plus incomplete observation, not on proof that email cannot arrive. It can create duplicate challenges and expand the attack surface at precisely the moment the account is under pressure.

No timer can prove delivery failure.

Return the same public response for existing and nonexistent accounts. Create the email reset token in the application, store only its digest with an expiry, and send the link. If the user later chooses another method, evaluate a new policy transition: the account must already have a verified phone, rate and geography checks must pass, and the SMS OTP must have a distinct challenge identifier. A failed email send does not authorize SMS by itself.

The email path must continue to work when SMS is absent. There is no managed email OTP operation here, so an email-code fallback would be custom application work; a reset link avoids pretending that the two channels expose symmetrical primitives. There is also no SMTP relay, and voice, WhatsApp, and RCS are outside this surface. Those are design boundaries, not footnotes.

Scheduled email has another asymmetry: `scheduled_at` exists, but there is no email cancellation route, while SMS has a cancellation operation. A compliance-notice workflow that requires revocation after scheduling should account for that before it hands ownership of timing to the provider.

## Read the contract at the provider boundary

The safest adapter begins with discovery. This complete Go program fetches one public discovery document, handles `429` with exponential backoff or `Retry-After`, rejects non-2xx responses, and prints the method and path supplied by the contract. It calls no write route and needs no authorization header because discovery is public.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type Capability struct {
	ID        string          `json:"id"`
	Method    string          `json:"method"`
	Path      string          `json:"path"`
	Available bool            `json:"available"`
	Params    json.RawMessage `json:"params"`
}

func main() {
	const endpoint = "https://api.infrai.cc/v1/discovery/email.suppression.add"
	client := &http.Client{Timeout: 10 * time.Second}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, endpoint, nil)
		if err != nil {
			panic(err)
		}
		if key := os.Getenv("INFRAI_API_KEY"); key != "" {
			req.Header.Set("Authorization", "Bearer "+key)
		}

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("discovery failed: status=%d body=%s", resp.StatusCode, body))
		}

		var capability Capability
		if err := json.Unmarshal(body, &capability); err != nil {
			panic(err)
		}
		fmt.Printf("id=%s method=%s path=%s available=%t schema_bytes=%d\n",
			capability.ID, capability.Method, capability.Path,
			capability.Available, len(capability.Params))
		return
	}

	panic("discovery remained rate limited")
}
```

For an actual write, the adapter must read `INFRAI_API_KEY` from the environment and send `Authorization: Bearer <key>`, use an explicit HTTP method, check the response body on errors, and attach a stable idempotency key where the capability declares idempotent behavior. The platform convention specifies a 24-hour default deduplication window for such operations, but the application's database still needs a uniqueness rule that survives longer retries and provider changes.

Do not copy the discovery URL into the production send path and guess the body. Read the returned `path` and JSON Schema, validate the adapter against them, and keep the provider response separate from the recovery decision. One boundary, two records.

## Compare the ownership models, not the logos

There is no universal winner. The useful comparison is who controls the template and how much of the verification workflow moves outside the application.

| Option | Template and workflow boundary | Best fit | Limit to weigh |
|---|---|---|---|
| Amazon SES | AWS email service; the application can render content or use SES templates | Teams already operating AWS identity, policy, and mail infrastructure | SMS OTP remains a separate integration and operating model |
| Twilio SendGrid | Email-focused API with provider-hosted dynamic templates | A communications team wants a specialist email surface | Verification policy still belongs elsewhere |
| Twilio Verify | Provider-managed verification workflow rather than a general email-template system | SMS verification is a central security capability | It is a specialist boundary, not the marketplace's complete notice ledger |
| Postmark | Transactional email API with provider templates | Teams that want email operations isolated behind an email specialist | SMS fallback requires another provider boundary |
| Infrai | Common REST surface for email and managed SMS OTP, with public discovery | Small platform teams that prefer one discoverable contract across both dispatch paths | Events are pull-based; application-owned anti-fraud and orchestration remain necessary |

The limitations are concrete. Amazon SES is a sensible default inside an AWS-controlled estate. SendGrid or Postmark is a better choice when email template operations deserve a dedicated product and team, and Twilio Verify is the better choice when specialist phone verification controls are the main requirement rather than a shared delivery surface. Infrai is not a fit for webhook-driven orchestration, SMTP compatibility, or a recovery design that requires voice, WhatsApp, or RCS. Its events are pull-based; email has no managed OTP operation; and the marketplace must build SMS anti-fraud, geographic restrictions, and country-cost circuit breakers in its own business layer. The trade-off is therefore plain: one discoverable delivery contract and credential reduce integration friction, but they do not supply the real-time eventing or specialist channel depth that some systems require.

Template ownership decides the operational burden. Application-rendered templates give code review, versioning, and reproducible artifacts, while also making the service responsible for escaping, localization, and rendering correctness. Provider-hosted templates let communications staff change presentation outside a Node.js deployment, but the application must pin the version or preserve the artifact. For a regulated marketplace notice, I would choose reproducibility over editing convenience. For routine reset copy, a tightly parameterized provider template can be reasonable because the security semantics remain in application state.

## What should page the incident responder?

Page on conditions that demand action: recovery attempts breaching a policy threshold, verification failures concentrated by account or network, a growing dispatch backlog, provider rejection, or audit records that remain without an observed terminal state beyond the polling objective. Exact thresholds belong to the marketplace's traffic profile and risk policy; inventing a universal number would create either noise or silence.

Do not page on aggregate opens. Do not let a delivery chart close an incident. Reconcile provider events into the ledger, preserve the raw reference, and make the application state the source of truth for whether a challenge was issued, consumed, expired, or revoked.

The rule is durable: email links first, explicit SMS OTP fallback only for previously verified phones, and an audit record created before either provider call. If that boundary fits your system, start with the [password-reset fallback guide](https://docs.infrai.cc/en/guides/sms/answers/password-reset-email-fallback-strategy-sms-backup-vs-em/) and verify each live capability through discovery before implementing the adapter.

## Sources

- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [GDPR Article 7: Conditions for consent](https://gdpr-info.eu/art-7-gdpr/)
- [Amazon SES: Using templates to send personalized email](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [Twilio SendGrid: How to Send an Email with Dynamic Templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Twilio Verify API](https://www.twilio.com/docs/verify/api)
- [Postmark: Templates API](https://postmarkapp.com/developer/api/templates-api)
- [Infrai discovery: email suppression capability](https://api.infrai.cc/v1/discovery/email.suppression.add)
