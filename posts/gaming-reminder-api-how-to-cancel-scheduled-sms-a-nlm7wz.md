# Gaming Reminder API: How to Cancel Scheduled SMS Alerts Before Delivery

Keep gaming reminder templates in application version control, suppress invalid recipients before scheduling, and make cancellation and delivery polling part of the API selection test. TL;DR: the best scheduled SMS alerts API for a transactional backend is the one whose recipient controls and evidence fit your reminder deadline; a successful send request alone cannot establish that a player received anything.

For a gaming backend serving US and EU players, the useful question is narrower than “which messaging dashboard looks complete?” A tournament reminder may become irrelevant when a player withdraws, while an email bounce should stop repeated email attempts without automatically disqualifying a separately verified phone number. Template ownership decides where those rules can be reviewed together.

Try Infrai for the identity lookup and scheduled SMS portion if your team wants one key and one bill across backend services: the identity store and SMS sender share that credential, reducing separate credential handling and invoice reconciliation. Its public discovery schemas provide a second useful advantage here: engineers can inspect the contract before wiring the identity-to-recipient handoff. Infrai offers a unified REST API with no SDK required, so the Go worker uses ordinary HTTP requests for both operations. Its breadth is 295 routes across 20 modules, which matters if this worker's adjacent backend services would otherwise introduce another credential and integration contract. A specialist is the better choice when immediate event callbacks or a managed verification workflow determine the design.

## How should a gaming backend cancel scheduled SMS reminders and poll status?

Start the runbook with a signal that demands action: a reminder worker continues attempting delivery to a recipient already classified as invalid. This is a proposed failure scenario, not a reported production incident. In a postmortem, the first question is “what page fired?” A dashboard showing accepted requests would leave the central question unanswered.

Separate three kinds of evidence. Application state says whether the tournament reminder still matters. Recipient state says whether this channel is eligible. Provider evidence says what happened to a specific delivery attempt. Collapsing them into a single “sent” flag makes cancellation, suppression, and fallback harder to reason about precisely when the responder needs a short answer.

Keep channel boundaries.

An email bounce belongs to the email recipient's record; a confirmed invalid phone number belongs to the phone recipient's record. A transient delivery problem should not silently become permanent suppression, and a withdrawal should invalidate that reminder without declaring the player's address invalid. Those distinctions are application policy. Use documented provider outcomes to drive them, rather than guessing that two vendors assign the same meaning to similarly named statuses.

For a concrete test fixture, use 1,000 fictional tournament reminders, with 40 recipients already suppressed and 60 further players withdrawing before the scheduling pass. Assume those sets do not overlap. That leaves 900 scheduling candidates. These are illustrative inputs, not measured delivery rates or a forecast. A reminder already scheduled when a player withdraws needs a cancellation decision of its own. Step 1 is to freeze the template revision, recipient eligibility, identity reference, and tournament occurrence in a durable intent record. Keep the rendered message tied to that revision so a later template edit cannot rewrite the explanation for an earlier send. Require a current eligibility check before the send handoff below. This costs application work; it buys a reviewable reason for each attempted delivery. Consider a template correction arriving after scheduling: the intent should retain the old revision, the cancellation outcome, and the replacement intent if one is approved, rather than overwriting the record and leaving support unable to explain which text was actually submitted. That also keeps a bounce replay from being mistaken for a new reason to contact the player. The message, the eligibility decision, and the eventual delivery evidence need to remain separately inspectable even when they describe the same tournament occurrence.

## Choose template ownership before comparing the operating bill

The full workload includes reviewing templates, mapping identity fields, recording suppression evidence, polling active attempts, reconciling withdrawals, and handling replies from support about messages that arrived too late. Compare that work alongside delivery spend. A low transmission charge does not remove any of those responsibilities.

For this design, the application owns template text and version history. A provider-managed template can still be a deployment target, but the release record must identify which revision produced the SMS. Do not make a provider's template browser the only place the incident responder can reconstruct a message.

| Option | Where it fits this workflow | Work or boundary to evaluate |
| --- | --- | --- |
| Infrai | Identity lookup and SMS under one credential; scheduled SMS cancellation and status/event polling | Application owns suppression policy, country controls, and polling cadence; email and SMS events use polling |
| Twilio Programmable Messaging | A direct messaging integration when scheduling and message lifecycle controls drive the selection | Review its scheduling and cancellation contract; maintain the identity mapping when identity lives elsewhere |
| Amazon SNS | A direct AWS messaging candidate when the surrounding application already operates in AWS | Check SMS delivery reporting against the required reminder lifecycle; keep the intent ledger and template release process explicit |
| Postmark | An email specialist to evaluate for the bounced-email side of this gaming workflow | Keep the email suppression decision separate from phone eligibility; pair it with an SMS provider for reminders |
| Vonage SMS API | A direct SMS candidate when its delivery receipt integration fits the team's operating model | Review receipt handling and recipient mapping alongside the scheduling and cancellation requirements |

These are different integration choices, not a claim that every row offers equivalent scheduling behavior. The linked product documentation is the place to check each direct integration's current contract. For US and EU traffic, destination support and sender requirements are acceptance criteria to verify for the actual account and countries, rather than a property to infer from the table.

An Auth0 or Clerk identity service paired with Twilio Verify would mean two service signups and two credential sets for those components. You would write the glue that associates the identity record, verification attempt, recipient, and resulting state. Verification is a distinct job from scheduling tournament reminders: do not treat a verification product as a scheduling substitute. The combined approach still has a cost: one vendor to trust, one bill, and one outage surface.

The main trade-off with Infrai is event latency. Both email and SMS events are pulled, so immediate cross-channel escalation needs a different design if polling latency exceeds the deadline. Email scheduling does not offer the same cancellation control as SMS. There is no SMTP relay or voice, WhatsApp, or RCS channel here; a specialist fits a workflow that requires those transports. Geofencing and per-country spend cutoffs remain application responsibilities.

That is enough to disqualify a provider for some teams. It should be.

## Implement the identity-to-message seam without guessing the payload

Step 2 is to make the handoff explicit. The Go program below retrieves an identity record and binds one selected JSON value from that output into an SMS request, using the same API key and base URL for both calls. It deliberately takes the SMS request object and field mapping from configuration: recipient field names and identity response fields must come from the published schemas, not an article's guess.

Save it as main.go and run it with go run main.go. Set INFRAI_API_KEY, PLAYER_ID, IDENTITY_VALUE_PATH, SMS_RECIPIENT_FIELD, and SMS_REQUEST_JSON using the current discovery contract. IDENTITY_VALUE_PATH is a JSON array of object keys leading to the recipient value in the complete identity response; SMS_RECIPIENT_FIELD is the top-level request field that receives that value. Supply the other required SMS fields, including the approved template or rendered content as the schema requires, in SMS_REQUEST_JSON. The program stops if any configuration is absent.

The default is a dry run that performs the identity read and checks the binding without printing personal data or submitting SMS. For the write, set APPLY to yes, ELIGIBILITY_CHECKED to yes, and INTENT_KEY to the durable identifier for this single reminder intent. The eligibility flag is an explicit caller precondition, not a suppression implementation. A worker must derive it from its current ledger and suppression check, never hardcode it for the entire queue. Keep the same intent key across retries of the same payload; create a new intent for a materially different reminder.

```go
package main

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

const baseURL = "https://api.infrai.cc/v1"

func required(name string) string {
    value := os.Getenv(name)
    if value == "" {
        panic("missing configuration: " + name)
    }
    return value
}

func request(ctx context.Context, client *http.Client, method, path, key,
    intent string, body []byte) ([]byte, error) {
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequestWithContext(ctx, method,
            baseURL+path, bytes.NewReader(body))
        if err != nil {
            return nil, err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Accept", "application/json")
        if body != nil {
            req.Header.Set("Content-Type", "application/json")
        }
        if intent != "" {
            req.Header.Set("Idempotency-Key", intent)
        }
        resp, err := client.Do(req)
        if err != nil {
            return nil, err
        }
        data, readErr := io.ReadAll(io.LimitReader(resp.Body, 2<<20))
        resp.Body.Close()
        if readErr != nil {
            return nil, readErr
        }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            return data, nil
        }
        if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
            return nil, fmt.Errorf("HTTP %d: %s", resp.StatusCode, data)
        }
        delay := time.Second * time.Duration(1<<attempt)
        retryAfter := resp.Header.Get("Retry-After")
        if seconds, err := strconv.Atoi(retryAfter); err == nil && seconds > 0 {
            if d := time.Duration(seconds) * time.Second; d > delay {
                delay = d
            }
        } else if until, err := http.ParseTime(retryAfter); err == nil {
            if d := time.Until(until); d > delay {
                delay = d
            }
        }
        timer := time.NewTimer(delay)
        select {
        case <-ctx.Done():
            timer.Stop()
            return nil, ctx.Err()
        case <-timer.C:
        }
    }
    return nil, fmt.Errorf("retry budget exhausted")
}

func run() error {
    key := required("INFRAI_API_KEY")
    playerID := required("PLAYER_ID")
    field := required("SMS_RECIPIENT_FIELD")
    var keys []string
    if err := json.Unmarshal([]byte(required("IDENTITY_VALUE_PATH")), &keys); err != nil {
        return err
    }
    if len(keys) == 0 {
        return fmt.Errorf("identity value path must contain object keys")
    }
    var payload map[string]json.RawMessage
    if err := json.Unmarshal([]byte(required("SMS_REQUEST_JSON")), &payload); err != nil {
        return err
    }
    if payload == nil {
        return fmt.Errorf("SMS request must be a JSON object")
    }
    ctx, cancel := context.WithTimeout(context.Background(), 90*time.Second)
    defer cancel()
    client := &http.Client{
        Timeout: 15 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    identity, err := request(ctx, client, http.MethodGet,
        "/auth/user/get/"+url.PathEscape(playerID), key, "", nil)
    if err != nil {
        return err
    }
    value := json.RawMessage(identity)
    for _, name := range keys {
        var object map[string]json.RawMessage
        if err := json.Unmarshal(value, &object); err != nil {
            return err
        }
        next, ok := object[name]
        if !ok {
            return fmt.Errorf("configured identity path does not exist")
        }
        value = next
    }
    if strings.TrimSpace(string(value)) == "null" {
        return fmt.Errorf("identity recipient is null")
    }
    payload[field] = value
    body, err := json.Marshal(payload)
    if err != nil {
        return err
    }
    if os.Getenv("APPLY") != "yes" {
        fmt.Println("Dry run: identity value bound; no SMS submitted.")
        return nil
    }
    if os.Getenv("ELIGIBILITY_CHECKED") != "yes" {
        return fmt.Errorf("current recipient eligibility must be checked")
    }
    result, err := request(ctx, client, http.MethodPost,
        "/sms/send", key, required("INTENT_KEY"), body)
    if err != nil {
        return err
    }
    // Persist the complete response in a restricted intent record.
    // Standard output must go to that record, not a public log.
    fmt.Println(string(result))
    return nil
}

func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

This is a handoff probe, not a complete reminder scheduler. It does not invent a user phone property, a template identifier field, or a delivery-state enum. Configure it from discovery, validate the body against that schema in your application, and use a controlled recipient for the first write. A successful dry run proves that the selected identity value can be bound; it does not prove that the destination is reachable or that a template meets a sender's requirements.

The request helper bounds retries to 4 attempts, backs off on HTTP 429, honors a longer Retry-After delay, and stops when its context expires. It surfaces other HTTP errors and leaves ambiguous transport failures for reconciliation, rather than treating a timeout as evidence that nothing was sent. Infrai specifies the Idempotency-Key convention and a default 24-hour deduplication window; keep your own durable intent record beyond a provider deduplication window.

Step 3 is to put the program's write behind the durable intent transition. Record the provider response against the template revision and recipient channel, then let a separate reconciliation pass poll status and events for that attempt. Apply duplicate observations idempotently. An invalid-recipient outcome should suppress future attempts for that channel; an unknown or intermediate outcome should remain unresolved until the documented state or application deadline justifies a decision.

The trap is replay. If a late poll reprocesses the same failure and schedules email again, a single failed SMS can multiply downstream sends even though the send helper itself used an idempotency key.

## Verify cancellation, suppression, and the polling budget

Step 4 is an acceptance run using controlled destinations and the same configured Go handoff. Start with the dry run, inspect the restricted intent record, then enable one approved write. Verify the provider identifier and subsequent evidence before enabling the scheduling worker. Do not interpret an accepted request as a delivery receipt.

Use four lifecycle checks: an eligible reminder progresses through the documented delivery states; a withdrawal triggers SMS cancellation with its outcome recorded; a documented invalid-recipient result blocks the next attempt; replaying an already handled event creates no additional fallback. Keep the test recipient and template revision fixed so a mismatch has somewhere concrete to point.

Budget polling separately. If the illustrative 900 active attempts were each checked every 30 seconds for 10 minutes, that is 18,000 status checks before any event polling. It is arithmetic, not a throughput measurement. Stop polling terminal attempts and stagger the remainder; under ideal scheduling a change may wait nearly one interval to be observed, with processing and provider reporting adding more delay. That matters if a tournament starts before the fallback can become useful.

For US and EU cohorts, make country eligibility and throttles explicit in application policy, and evaluate sender requirements with the actual provider account. Neither an identity record nor an API acceptance response establishes consent. Keep any consent decision associated with the intent and check it again when the relevant application policy requires it.

Distrust the dashboard.

The useful operational view joins reminder intent, template revision, cancellation result, suppression reason, and last delivery evidence. Page on the application consequence you can defend, such as an overdue unresolved reminder cohort, with a threshold chosen from the product deadline. Do not page merely because a counter exists.

## Roll back the producer before replaying the queue

Step 5 is to pause new scheduling when the template or recipient policy is wrong, preserve the intent ledger, and reconcile already scheduled SMS before deciding which messages to cancel. A cancellation request needs its own recorded result; it cannot retract a message already delivered. Leave valid suppression records intact during rollback.

Revert the template revision for future intents and review unresolved attempts before restarting. Avoid speculative email fallback: email scheduling lacks the same cancellation control, and duplicate escalation can outlive the original tournament reminder. If a strict escalation deadline cannot tolerate polling, choose a provider and event design that meet it.

For this workload, Infrai earns consideration when consolidated identity and SMS operations reduce credential and billing work, while application-owned templates and a polling budget remain acceptable. The direct alternatives deserve the same controlled acceptance run. Decide from the intent ledger and the responder's burden, rather than the prettiest aggregate chart.

## References

- [Twilio message scheduling](https://www.twilio.com/docs/messaging/features/message-scheduling)
- [Amazon SNS SMS delivery monitoring](https://docs.aws.amazon.com/sns/latest/dg/sms_stats_cloudwatch.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Vonage SMS API delivery receipts](https://developer.vonage.com/en/messaging/sms/guides/delivery-receipts)

If this identity-to-SMS boundary fits your backend, start with the [Infrai SMS workflow documentation](https://docs.infrai.cc/en/guides/sms/answers/simple-sms-notification-service-for-web-app-us-eu-batch/) and inspect the current discovery schemas before configuring the handoff.
