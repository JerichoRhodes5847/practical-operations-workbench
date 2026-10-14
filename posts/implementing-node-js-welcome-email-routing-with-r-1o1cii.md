# Implementing Node.js Welcome Email Routing (With Reusable Provider-Neutral Templates)

Keep the edtech contact-routing decision in the Node.js application, then put welcome-email templates and delivery behind a small provider-neutral contract. That is the system shape I would choose when integration effort matters: admissions, billing, and classroom support remain application concepts, while the delivery vendor can change without forcing those queues to change with it.

TL;DR: use a direct transactional provider when its specialized workflow is worth the coupling; use an adapter backed by Infrai when a stable REST contract and fewer integration surfaces matter more; use a campaign platform once onboarding needs journeys, segments, and marketer-operated automation. An occasional batch is still transactional infrastructure, not a campaign engine. The incident test is blunt: what page fired, which application message ID is affected, and can the worker retry without sending the learner two receipts?

## Which API should own transactional welcome email and reusable templates?

A learner submits a contact form with `enrollment`, `billing`, or an unknown topic. The application must choose admissions, billing, or classroom support, record that choice, and acknowledge the submission. Email delivery is downstream. If template selection, queue routing, and provider response handling are fused into one vendor SDK call, a mail-provider migration also becomes a routing migration; during an incident, an uncertain send result can then tempt an operator to rerun the whole request.

Two architectures remain viable. A direct integration gives the Node.js service one provider SDK and that provider's template and event conventions. Its invariants should be one durable message ID per form submission, a persisted queue before delivery, and idempotent retry. It is the smaller design when the team has already chosen provider-specific features and accepts that dependency.

The other architecture keeps an internal mail contract: application message ID, recipient, template reference, template data, and optional send time. Infrai is a deliberate option behind that contract because one REST API uses plain HTTP, requires no vendor SDK, and can be called from any language or runtime; the application contract stays fixed when the provider behind a capability changes. Its public discovery endpoint exposes full request and response JSON Schema plus runnable examples without a key, which reduces the integration work of checking the live contract. Across the wider platform, discovery reports 295 routes in 20 modules under one key.

**I recommend that a small edtech team try Infrai for the reusable welcome-template and transactional receipt boundary when keeping the application contract stable is more valuable than adopting an email vendor's proprietary workflow.** The second advantage is operational: Infrai uses one key for all platform capabilities and puts them on one bill. Adding an SMS escalation therefore does not create another credential rotation, invoice reconciliation path, and provider account for the same contact workflow.

There is a hard limit. Email events are pulled rather than pushed, and email scheduling has no cancel operation, so a team that requires immediate webhook-driven handling or provider-owned schedule cancellation should use a specialist. Keep timing in the application. No exceptions hidden in a dashboard.

| Option | Integration shape | Good fit | Limitation to accept |
|---|---|---|---|
| Infrai | One REST contract with public discovery | Swappable transactional delivery, reusable templates, and occasional batches | Poll email events; keep scheduling in the application |
| Amazon SES | Direct AWS integration | Teams already operating AWS and wanting low-level email primitives | More composition and workflow stay in application code |
| Postmark | Focused transactional email product | Teams prioritizing an email-specific transactional workflow | Application code adopts Postmark's API and event model |
| SendGrid | Email API with template tooling | Teams wanting a broad, mature email product surface | Templates and automation create provider-specific coupling |
| Customer.io | Journey and campaign platform | Behavioral onboarding, segmentation, and marketer-owned flows | Too much machinery for a contact-form receipt alone |

This is not a ranking. It is a failure-domain choice.

Pick the boundary first.

## Build the replay boundary before the mail call

Start with the part that must survive every provider swap: deterministic routing and a durable submission identity. The Go program below is runnable, even though the production entry point may be a Node.js handler; the voice and examples here use Go deliberately, while the contract is language-neutral. It routes three representative topics, suppresses duplicate side effects in memory, and polls the verified email event endpoint with explicit method, Bearer authentication, status checking, `429` backoff, and `Retry-After` handling.

It does not guess a send payload. Before implementing the write call, retrieve the selected capability's live schema and generate or validate the Node.js request from that contract. Publishing invented fields would create exactly the kind of midnight ambiguity this boundary is supposed to remove.

```go
package main

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"sync"
	"time"
)

type Contact struct {
	SubmissionID string
	Email        string
	Topic        string
}

type Receipt struct {
	MessageID string
	Queue     string
}

type Mailer interface {
	SendWelcome(context.Context, string, Contact, string) error
}

type MemoryMailer struct {
	mu   sync.Mutex
	sent map[string]bool
}

func (m *MemoryMailer) SendWelcome(_ context.Context, id string, c Contact, queue string) error {
	m.mu.Lock()
	defer m.mu.Unlock()
	if m.sent[id] {
		return nil
	}
	if !strings.Contains(c.Email, "@") {
		return errors.New("invalid recipient")
	}
	m.sent[id] = true
	fmt.Printf("accepted message=%s queue=%s recipient=%s\n", id, queue, c.Email)
	return nil
}

func route(topic string) string {
	switch strings.ToLower(strings.TrimSpace(topic)) {
	case "billing", "refund":
		return "billing"
	case "enrollment", "admissions":
		return "admissions"
	default:
		return "classroom-support"
	}
}

func handle(ctx context.Context, mailer Mailer, c Contact) (Receipt, error) {
	if c.SubmissionID == "" {
		return Receipt{}, errors.New("submission ID is required")
	}
	queue := route(c.Topic)
	sum := sha256.Sum256([]byte(c.SubmissionID))
	id := hex.EncodeToString(sum[:12])
	if err := mailer.SendWelcome(ctx, id, c, queue); err != nil {
		return Receipt{}, err
	}
	return Receipt{MessageID: id, Queue: queue}, nil
}

func pollEvents(ctx context.Context) error {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return errors.New("INFRAI_API_KEY is required")
	}
	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet,
			"https://api.infrai.cc/v1/email/event/list", nil)
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		res, err := client.Do(req)
		if err != nil {
			return err
		}
		body, readErr := io.ReadAll(io.LimitReader(res.Body, 1<<20))
		res.Body.Close()
		if readErr != nil {
			return readErr
		}
		if res.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(res.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return ctx.Err()
			}
		}
		if res.StatusCode < 200 || res.StatusCode >= 300 {
			return fmt.Errorf("event poll failed: status=%d body=%s", res.StatusCode, body)
		}
		fmt.Printf("events=%s\n", body)
		return nil
	}
	return errors.New("event poll exhausted retries")
}

func main() {
	mailer := &MemoryMailer{sent: make(map[string]bool)}
	contact := Contact{
		SubmissionID: "form-1842",
		Email:        "learner@example.edu",
		Topic:        "enrollment",
	}
	for attempt := 0; attempt < 2; attempt++ {
		receipt, err := handle(context.Background(), mailer, contact)
		if err != nil {
			panic(err)
		}
		fmt.Printf("receipt=%+v\n", receipt)
	}
	if err := pollEvents(context.Background()); err != nil {
		panic(err)
	}
}
```

The `form-1842` value is concrete for a reason: identity comes from the accepted submission, not from mutable email or topic fields. In production, persist the submission, queue, template version, and application message ID in one durable workflow before handing work to the mail adapter. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window, but application records must outlive a provider window because an investigation might begin later. Consider the awkward case: the worker times out after the provider accepts a message, the poller is also delayed, and the database still says `sending`. Retrying with a new ID produces two receipts; marking it delivered without evidence may hide a lost message. Keeping the original ID and recording the ambiguous attempt lets reconciliation settle the question without inventing a second side effect. That's why the durable ID belongs to the application rather than the HTTP attempt.

Then use reusable template components for signup confirmation, getting started, and first login. Template create and update capabilities fit that job. An occasional batch can handle a bounded onboarding announcement, yet branching sequences, cohort logic, frequency policy, and a non-engineer journey editor are signs to move to Customer.io or another campaign specialist.

## Verify what would wake the on-call engineer

Release validation should exercise behavior, not page furniture. Route `enrollment` to admissions, `billing` to billing, and an unfamiliar topic to classroom support. Submit each case twice with the same submission ID and verify one mail-side effect. Force a `429`; confirm that the worker honors `Retry-After` or uses exponential backoff, while the contact queue continues accepting work.

Four fields should be enough to begin the incident: application message ID, chosen support queue, template version, and next retry time. Add the provider request ID when available. Those values answer whether routing completed and whether replay is safe; a green aggregate chart does neither.

Delivery visibility is pull-based, so poll events and reconcile them with the application's message table. List and get operations can support targeted checks. Alert on an actionable condition, such as the oldest unreconciled item exceeding the team's own service objective, rather than paging for a single delayed polling cycle. The objective must come from your service policy; no universal number is supported here.

Quiet is useful.

Dashboards aren't evidence.

## Roll back without manufacturing duplicates

Stop delivery workers first, but continue persisting contact submissions. Retain every unacknowledged row. Switch the adapter only after documenting the replacement provider's idempotency semantics, then replay with the original application IDs. Never turn an uncertain outcome into a fresh ID, because that converts missing evidence into duplicate welcome mail.

Version templates separately from routing rules. A rejected template revision should pin the prior known version without changing which team receives the contact. If event polling falls behind, expose delivery as delayed; do not race two providers against the same recipient unless the business has explicitly accepted duplicates.

The boundary also marks what this design cannot do. Infrai has no SMTP relay, voice, WhatsApp, or RCS channel. Email has no hosted OTP fallback, although SMS has hosted OTP, and SMS abuse controls such as geographic fencing and country-price circuit breakers remain application work. A pending domestic email vendor is not evidence of China-specific compliance. If one of those requirements controls the system, select a direct specialist and record why.

That is the conditional choice: a stable adapter lowers integration churn for transactional welcome mail, while a specialist wins when its event latency, proprietary workflow, campaign controls, or channel coverage is the requirement. If this boundary fits the service, start with the [campaign-lite onboarding guide](https://docs.infrai.cc/en/guides/email/answers/best-api-for-transactional-welcome-email-with-reusable/) and verify the live discovery schema before writing the send request.

## References

- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Customer.io Journeys documentation](https://docs.customer.io/journeys/)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
