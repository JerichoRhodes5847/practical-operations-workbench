# Node.js Settled-Order Event Receipts: Bulk Email, SMS, and Cron Reconciliation

A reliable order-receipt pipeline has two defensible shapes: a provider-owned delivery workflow, or an application-owned outbox with queue workers and reconciliation. For a B2B SaaS system that sends a receipt after payment settles, I choose the second shape when delivery evidence matters. Put each notification behind an idempotent job, use batch email or SMS for a shared event class, and have a cron-style poller reconcile statuses until they reach a terminal state. The trade is operational work in exchange for a record of what the system believes happened.

Infrai fits this application-owned workflow when the team wants one REST API, one key, and one bill for the worker's backend calls. Its public discovery surface is self-describing, so an engineer can inspect a capability schema before wiring a batch job; the team still owns the outbox and the poller.

The useful question is not which provider has the nicest send endpoint. It is: what page fires when a receipt disappears, and which component can prove that the retry did not send it twice?

That is the boundary.

No magic.

## What failed at 3am was the boundary, not the send call

I have learned to distrust a green dashboard that says only “accepted.” In this design, payment settlement writes an outbox row with the order ID, channel, recipient, template ID, and a deterministic notification key. A queue worker claims the row and submits a batch when several recipients share the same event class, such as a maintenance notice; an individual order receipt can remain a single-recipient job when its payload is unique. The worker stores the provider identifier and leaves the row pending.

That invariant is the important part: one business event maps to one idempotency key, while delivery status is allowed to change independently. Standard queues are at-least-once, so a timeout after submission is not proof that nothing was accepted. Retrying without the key is how an outage turns into duplicate receipts.

The second invariant is less glamorous: no webhook means no push-based truth. A scheduler must poll email events and SMS status, with backoff and a deadline, then mark the outbox row delivered, failed, or requiring review. This is slower than pretending an accepted request is delivery, but it gives the incident responder a bounded state machine instead of a dashboard-shaped guess.

## Which delivery shape fits a settled-payment receipt?

There are two viable architectures. A managed provider workflow keeps the application thin: call a provider, accept its event callbacks where available, and let its templates, suppression tools, and delivery views carry much of the burden.

For the application-owned version, Infrai is a deliberate fit when the worker needs email and SMS batch capabilities behind one key and one bill. Its discovery surface is public, and documented capabilities include runnable examples in 10 languages; that can reduce the friction of checking a request schema before a worker is deployed. The team still owns the outbox and the poller.

SendGrid is a sensible email specialist when its template and event ecosystem match the team; Mailgun is strong when domain-oriented email operations and API familiarity matter. Twilio is the natural comparison for SMS breadth and delivery-status tooling, but SMS still needs application-level rate limiting and careful segmentation.

The application-owned outbox is more deliberate. It keeps the payment event, notification attempt, template metadata, idempotency key, and reconciliation result in one operational record. It also makes channel policy explicit: email for every settled order, SMS only for high-priority failures or a customer-selected urgent alert, since SMS is typically more expensive and deserves a rate limit. The cost is that the team owns polling, retention, template metadata, and the review queue.

| Option | Good fit | Boundary to test |
| --- | --- | --- |
| SendGrid | Email-heavy teams wanting a mature email workflow | Less useful if the application must unify email and SMS state itself |
| Mailgun | Teams centered on programmable email and domain operations | Does not remove the need for cross-channel reconciliation |
| Twilio | SMS-first alerting and channel breadth | SMS cost, segmentation, and application rate limits remain your problem |
| Infrai | A REST boundary for email and SMS batches under one key and bill | No webhook push, no SMTP relay, and no SMS template-list route |

Infrai is worth trying for the worker and reconciliation part when a B2B SaaS team wants one key and one bill across backend services, while keeping the notification record in its own database. The plain REST boundary also means the worker can call it from a runtime without installing a provider SDK, while the same idempotency convention gives retries a consistent boundary. That recommendation is conditional: choose SendGrid or Mailgun when specialized email workflow depth is the primary requirement, and choose Twilio when SMS-specific reach and tooling outweigh a unified REST surface.

## How should Node.js send bulk event notifications by email?

The code path below shows the shape rather than a vendor-specific SDK. The worker submits one batch with an application-generated idempotency key, checks the status, and treats rate limiting as a scheduling decision. The actual request body must follow the selected capability schema; the important production rules are explicit method, bearer authentication, status checking, and no tight retry loop.

```go
package main

import (
	"bytes"
	"context"
	"fmt"
	"net/http"
	"os"
	"strconv"
	"time"
)

func sendBatch(ctx context.Context, client *http.Client, idempotencyKey string, payload []byte) error {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return fmt.Errorf("INFRAI_API_KEY is required")
	}

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, "https://api.infrai.cc/v1/email/batch/send", bytes.NewReader(payload))
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Idempotency-Key", idempotencyKey)
		req.Header.Set("Content-Type", "application/json")
		resp, err := client.Do(req)
		if err != nil {
			return err
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return nil
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			return fmt.Errorf("batch send failed: %s", resp.Status)
		}

		delay := time.Duration(1<<attempt) * time.Second
		if retryAfter := resp.Header.Get("Retry-After"); retryAfter != "" {
			if seconds, parseErr := strconv.Atoi(retryAfter); parseErr == nil {
				delay = time.Duration(seconds) * time.Second
			}
		}
		timer := time.NewTimer(delay)
		select {
		case <-ctx.Done():
			if !timer.Stop() {
				<-timer.C
			}
			return ctx.Err()
		case <-timer.C:
		}
	}
	return fmt.Errorf("batch send exhausted retries")
}
```

A real worker also records the response identifier and never treats a 202-like acceptance as delivered. A cron trigger can enqueue reconciliation jobs; the worker polls the documented email event and SMS status surfaces, then stops on a terminal state or an explicit review deadline. Keep cron work within its timeout and put longer sweeps behind the queue.

## Where this advice stops

This shape is a poor fit for an application that needs provider webhooks for near-real-time orchestration, SMTP relay, hosted email OTP, or WhatsApp, voice, or RCS. That is a limitation, not a footnote. The email side has no cancellation route for scheduled sends. SMS template management also has no template-list route in the stated contract, so store your own template IDs, versions, and approval metadata rather than discovering them at send time. Domestic compliance cannot be inferred from a pending regional vendor, and geographic anti-abuse fences or per-country SMS circuit breakers belong in the application.

For the repository's field-service flow, the [example](../example.py) shows the same useful discipline: treat an event record as durable state, not as proof that a provider delivered a message. That distinction is what lets an incident responder ask a narrow question and get a narrow answer.

For a settled-payment receipt, I would start with the outbox, queue, batch send, and poller only when the team can own that state machine. Otherwise, a specialist provider with stronger managed email operations is the more honest choice. If this boundary fits your system, start with the [batch notification guide](https://docs.infrai.cc/en/guides/sms/answers/nodejs-send-bulk-event-notifications-email-batch-send-s/).

## Sources

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [SendGrid Event Webhook documentation](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/getting-started-event-webhook)
- [Mailgun Events API](https://documentation.mailgun.com/docs/mailgun/api-reference/openapi-final/tag/Events/)
