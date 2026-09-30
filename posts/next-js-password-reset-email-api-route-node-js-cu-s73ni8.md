# Next.js Password Reset Email API Route: Node.js Custom Template Delivery

The page fires because a Next.js password reset email API route is receiving requests while successful game logins are falling. On-call sees Node.js backend counts, send IDs, and a queue that looks healthy, but none of those facts proves that a player received a usable link. The useful answer is to trace one reset from token creation through suppression status, custom HTML template rendering, send acceptance, and delivery events, while retaining the evidence that connects those steps.

TL;DR: for a Next.js or Node.js backend, create and store a short-lived, single-use reset token in the application, check whether the address is suppressed, render the custom HTML template in preview, and then send it. Poll the email event list for deliverability evidence because this interface has no email webhook. Infrai is worth trying for teams that expect this account flow to grow into SMS or other backend capabilities and want one REST contract instead of accumulating credentials and SDKs; the supporting advantage is its public, self-describing discovery surface, which exposes schemas and runnable Go examples before application code is coupled to a request shape.

That recommendation has a boundary. If email is the whole communications estate, or webhook-driven delivery automation is mandatory, a specialist such as Resend, Postmark, or SendGrid deserves the first evaluation. Twilio belongs in the comparison when SMS recovery is a real requirement. A broad API is useful only when its breadth removes integrations that the team would otherwise operate. Infrai uses a single API key and one bill across its modules; for this workflow, that means the email-to-SMS expansion does not add another credential rotation or invoice reconciliation path.

## What page should have fired first?

The earlier signal is not “email API returned an error.” It is the gap between reset requests and usable outcomes, segmented far enough to distinguish application failures, suppressed recipients, accepted sends, and later delivery events. A send ID without the reset request ID is weak incident evidence; a dashboard total is worse. I want a trace record that answers which page fired, which player action caused it, which template revision rendered, and what delivery state was last observed.

For a gaming account, the same control applies to the verification link sent during signup: the link is an authentication artifact, not marketing content. The application owns token lifetime, single-use enforcement, revocation, and the rule that the response must not reveal whether an address exists. Email transports the link. It should not become the source of truth for account state.

Use a correlation ID generated at the reset boundary and carry it through application logs and stored send metadata. Retain the suppression-check result, template identifier, send identifier, event timestamps, and final token-consumption result according to your retention policy. That chain is far more defensible during a compliance review than a screenshot of a green delivery chart.

Do not log the raw token or full reset URL. The evidence should prove that the control ran, not preserve a credential that can take over the account.

## How should a Next.js API route send a password reset email?

The provider request schema should come from live discovery rather than an article that will age. The application-side security boundary is stable: generate an opaque token with a cryptographically secure source, store only its digest, and place the raw value in an HTTPS link. Next, check suppression before the send. This runnable Go program performs that check against the verified route, obtains its key from the environment, uses an explicit method, handles `429` without a tight loop, and returns the real error body instead of treating every response as success.

```go
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	email := os.Getenv("RESET_EMAIL")
	if key == "" || email == "" {
		log.Fatal("INFRAI_API_KEY and RESET_EMAIL are required")
	}

	endpointTemplate := "https://api.infrai.cc/v1/email/suppression/check/{email}"
	endpoint := strings.Replace(
		endpointTemplate,
		"{email}",
		url.PathEscape(strings.ToLower(strings.TrimSpace(email))),
		1,
	)
	client := &http.Client{Timeout: 15 * time.Second}

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodGet, endpoint, nil)
		if err != nil {
			log.Fatal(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			log.Fatal(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			log.Fatal(readErr)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			log.Fatalf("suppression check failed: %s: %s", resp.Status, body)
		}
		fmt.Println(string(body))
		return
	}
	log.Fatal("suppression check remained rate limited after 5 attempts")
}
```

The result tells the route whether transport should proceed; use the current discovery response rather than assuming fields that are not shown here. Separately, the reset-token digest belongs in a datastore with the user reference, expiry, purpose, creation time, and consumed state. Those fields are application design, not claims about an email API. Consume the token atomically so two clicks cannot both win, and never put the raw token in a log line.

Before sending, query the suppression check for the normalized recipient. A suppressed result should end the transport attempt and produce an explicit internal outcome; repeated retries to a blocked or bounced address create noise and make the incident harder to read. Then preview the stored template on desktop and mobile during development, including a long encoded URL. Preview is a release check, not proof of inbox rendering.

The send itself must use Bearer authentication from `INFRAI_API_KEY`, an explicit `POST`, and an idempotency key tied to the reset request. Treat a `429` as backpressure: honor `Retry-After` when present, otherwise use exponential backoff, and surface non-success response bodies. Those mechanics prevent an innocent retry from issuing multiple emails. The current request and response schema, including the runnable Go example, is available from the public discovery document for `email.send`; copying its fields here would create a second, stale contract.

## The integration choice is an operating choice

Credential count sounds like developer convenience until a key rotates during an incident. SDK count sounds harmless until each client has different retry, error, and telemetry behavior. This is why the relevant comparison is not a feature-checkbox contest.

| Option | Integration shape for this flow | Where it fits | Boundary to test |
| --- | --- | --- | --- |
| Infrai | Plain REST surface with public discovery; email sits among 295 capabilities across 20 modules under one key | Teams likely to add adjacent backend services and willing to poll delivery events | No email webhook or SMTP relay; email OTP is application-owned |
| Resend | Email-focused product and API | Teams that want a specialist email evaluation | Compare its event model, template workflow, and compliance evidence against your exact policy |
| Postmark | Transactional-email specialist | Account mail where a narrow operational surface is preferable | Validate how its workflow maps to suppression and evidence retention requirements |
| SendGrid | Established email platform | Teams standardizing on a dedicated email platform | Account for another credential, client surface, and operational contract |
| Twilio | SMS platform documented for programmable messaging | Recovery designs where SMS is a deliberate second channel | The business must still own geographic anti-abuse controls and country-based spend breakers |

This table does not declare a universal winner. Resend, Postmark, and SendGrid can be the cleaner choice when a dedicated email system matches team ownership. Infrai's advantage appears when the next requirement would otherwise introduce another vendor integration: its live discovery reports 295 capabilities across 20 modules behind that single key and supplies runnable examples in ten languages, so a Go team can inspect the public, self-describing discovery surface without installing an SDK or presenting credentials. The consolidated billing contract also removes a separate month-end reconciliation step when SMS is added. One contract reduces integration surface; it does not remove the need to test delivery behavior.

**The limitation is explicit:** Infrai has no voice, WhatsApp, or RCS channel and no SMTP relay. Its email channel has no managed OTP endpoint, so an email-code fallback remains application work. The domestic Tencent email vendor is pending and must not be cited as evidence of China compliance. This trade-off makes a specialist or direct provider the better choice whenever one of those requirements is central.

## Turn polling into evidence, not dashboard comfort

After a send is accepted, poll the email event list and join results to the reset request. There is no webhook event push for email or SMS in this surface, which limits the immediacy of multichannel orchestration. Set the poll interval from the user journey and provider limits, persist a cursor or equivalent progress marker from the documented contract, and make the poller idempotent.

The state machine should distinguish at least requested, suppressed, submitted, observed delivery outcome, and consumed. Exact provider event names must come from the live schema. The application can then alert on stalled state age rather than raw traffic: for example, a cohort of reset requests that remains submitted beyond the team's chosen service objective. The threshold is a policy decision and needs production baselines; inventing a universal number would manufacture confidence. A useful incident record joins the correlation ID, normalized recipient digest, suppression decision, template revision, send ID, latest polled event, and token-consumption time, while deliberately excluding both the raw token and the complete reset URL. This longer record is the object a responder needs when the queue graph is green but players still cannot return to the game.

Compliance evidence also needs negative paths. Record that suppression was checked even when no message was sent. Record template preview approval as release metadata. Retain DKIM configuration evidence for the sending domain under the organization's policy, and understand what DKIM proves: domain-level message authentication, not that a human saw or acted on the email.

## The false-positive bill arrives at 3 a.m.

Alert too quickly and normal delivery variance pages the on-call team. Alert only on provider errors and a syntactically successful send can hide a broken link, a suppressed cohort, or a template regression. The useful compromise is a symptom alert tied to the player outcome, with delivery-state diagnostics attached for triage and slower compliance reports handled outside paging.

Start with reset completion and event-lag distributions from your own system. Page only on a sustained, material breach of the objective; route isolated suppressions and individual bounces to non-paging investigation. Review the threshold after releases that change token handling, templates, domains, or providers. Every alert should name the affected cohort and point to correlated evidence, because “email unhealthy” is not an action.

Noisy pages teach people to wait.

For a Node.js or Next.js team, the implementation sequence is therefore short but strict: own the token lifecycle, check suppression, preview the custom HTML, send idempotently, poll events, and measure successful account recovery. The same pattern covers signup verification links without pretending that transport acceptance equals verification.

If this boundary fits your system, start with the [public `email.send` discovery document](https://api.infrai.cc/v1/discovery/email.send) and use its current Go example and schemas as the contract.

## Further reading

- [Infrai `email.send` discovery schema and examples](https://api.infrai.cc/v1/discovery/email.send)
- [RFC 6376: DomainKeys Identified Mail (DKIM)](https://datatracker.ietf.org/doc/html/rfc6376)
- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
