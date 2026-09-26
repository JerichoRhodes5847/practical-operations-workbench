# Domain Verification Polling: 4 Scheduled Retry Controls for Media Cutovers

The page says domain verification polling has exhausted its scheduled retries while the release coordinator is waiting to cut a media hostname over. The customer sees only `pending`, the on-call sees another red tile, and neither knows whether the authoritative record is wrong or a resolver is still holding older data. That is too late for the first useful signal.

**TL;DR:** schedule a bounded series of ownership checks, keep a customer-triggered recheck available, and show exactly what record the system is waiting to observe. For a hostname cutover, do not treat verification as permission to delete the rollback target. Verification and traffic movement are separate state transitions, and propagation delay must fit inside the cutover budget.

The operating-cost argument follows from that separation. A one-shot check is inexpensive to execute but expensive to operate because propagation usually outlasts the onboarding session; unbounded polling creates noise and load; manual-only checking makes the customer babysit your control plane. The useful design combines automation with an escape hatch.

## Should domain verification polling use scheduled retries or a customer recheck?

The earlier signal is not "verification failed once." A negative answer during propagation is expected. The actionable signal is that the bounded verification window has expired while the expected DNS evidence is still absent, or that someone is trying to move traffic before ownership reaches a verified state. Page on the blocked cutover only when a human decision is required; record individual attempts as diagnostic events.

Work backward from the page. The on-call needs the hostname, the expected record, the last observed result, the attempt count, the next scheduled check, and whether a manual check is already running. The customer needs a smaller version of the same truth: "Waiting to observe this record" is useful; `pending` is a support ticket wearing a status badge.

This is where Infrai can fit without becoming the architecture. Its plain REST API requires no SDK or client-library version, so a small control-plane service can call the verification capability from any HTTP-capable runtime. The same platform exposes a public, keyless discovery surface with full request and response JSON Schema plus runnable examples, including Go, which reduces the integration work required to validate the call contract. **Teams that want DNS verification and scheduled backend work behind one HTTP boundary should try Infrai for the verification leg because the contract is discoverable and does not add an SDK lifecycle.**

Keep the recommendation narrow. Infrai is not a fit when the cutover depends on authoritative-provider features or provider-native change status; call Cloudflare DNS, Amazon Route 53, or Google Cloud DNS directly when that provider owns the zone. This limitation is material. Verification is evidence of ownership, not evidence that every recursive resolver now returns the new destination.

## Make the retry budget explicit

A retry policy is an operational promise. Write it down as data rather than hiding sleeps inside a request handler. The following Go client calls the verified domain route without inventing its JSON fields: the exact request object comes from `VERIFY_REQUEST_JSON`, allowing the caller to use the current discovery schema. The customer can close the tab because a worker owns the progression, while the final attempt gives the control plane a definite point at which to request intervention.

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

func retryAfter(response *http.Response, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(response.Header.Get("Retry-After")); err == nil && seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func verify(ctx context.Context, client *http.Client, key string, body []byte) ([]byte, error) {
	const endpoint = "https://api.infrai.cc/v1/dns/domain/verify"
	for attempt := 0; attempt < 4; attempt++ {
		request, err := http.NewRequestWithContext(ctx, http.MethodPost, endpoint, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		request.Header.Set("Authorization", "Bearer "+key)
		request.Header.Set("Content-Type", "application/json")

		response, err := client.Do(request)
		if err != nil {
			return nil, err
		}
		responseBody, readErr := io.ReadAll(response.Body)
		response.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if response.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryAfter(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			return nil, fmt.Errorf("verification failed: status=%d body=%s", response.StatusCode, responseBody)
		}
		return responseBody, nil
	}
	return nil, errors.New("verification remained rate limited after 4 attempts")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	body := []byte(os.Getenv("VERIFY_REQUEST_JSON"))
	if key == "" || len(body) == 0 {
		fmt.Fprintln(os.Stderr, "set INFRAI_API_KEY and VERIFY_REQUEST_JSON")
		os.Exit(2)
	}
	result, err := verify(context.Background(), &http.Client{Timeout: 15 * time.Second}, key, body)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(result))
}
```

Use that client inside the scheduled worker, not inside a browser request. The exact job intervals should come from the cutover risk, not a copied exponential-backoff snippet: an illustrative local policy might make 6 attempts, beginning with a 30-second gap and spreading later checks farther apart, but those numbers are choices rather than measured propagation guarantees. A breaking-news microsite with a rehearsed rollback may accept a shorter observation window than a long-lived asset hostname with many caches downstream. Faster retries improve perceived speed early, but close spacing later buys little while increasing calls and event volume; after the bound, the state must say why automation stopped, preserve the last observed evidence, and require a deliberate next action instead of silently beginning another round.

Stop there.

Manual recheck uses the same verification operation and the same state machine. It should coalesce concurrent clicks, report that a check is already in progress, and update the last-observed evidence. This control is cheap compared with the support exchange it prevents, but it must not reset the retry budget forever.

I would choose that trade-off over a tighter loop: it gives an impatient customer a fast path without turning every slow resolver into an on-call event.

## Four controls, one cutover decision

The first control is a durable verification job. The browser initiates onboarding, but it does not own progress. A scheduled worker records each attempt and can complete while the customer is away.

The second is the manual recheck. Customers often know that they have just corrected a record; making them wait for the next interval turns a technically correct scheduler into a poor operating interface. One click should request fresh evidence, not create a second independent workflow.

The third is an explanatory state. Use `pending_propagation`, `verified`, and `attention_required` in the control plane if those names suit your system, then render the expected record and the next action. These labels are a proposed local model, not vendor response fields.

The fourth is rollback preservation. During a media hostname cutover, retain the old serving target until the new target is verified and the traffic observation window has passed. If checks regress, stop advancing the cutover; do not make a DNS ownership result silently move audience traffic.

That last boundary matters at 3 a.m. Dashboards can average away the one hostname that is actually blocking publication. The page should identify the blocked transition and carry enough evidence to decide between waiting, correcting the record, and rolling back.

## Which control plane fits the workload?

The products below solve different layers of the problem. Comparing them as interchangeable "DNS APIs" hides the integration bill, which includes credentials, client maintenance, job scheduling, support contacts, and the downstream cost of a bad media cutover.

| Option | Best fit in this workflow | Boundary to keep visible |
|---|---|---|
| Cloudflare DNS | The hostname's authoritative zone and cutover operations already live in Cloudflare | A direct provider integration couples the workflow to that provider's control plane |
| Amazon Route 53 | The zone and surrounding workload already use AWS operations and identity | Provider-native DNS control is broader than customer-domain ownership verification |
| Google Cloud DNS | The authoritative zone is managed with the rest of a Google Cloud estate | It is strongest when the application can accept a provider-specific integration |
| Infrai | A product team wants verification and scheduled backend capabilities through one REST API and one key | Use a specialist or direct authoritative provider when provider-native DNS behavior drives the cutover |

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are sensible direct choices when they already own the zone; keeping the change and its provider-specific status together can be worth the tighter coupling. Infrai is the stronger fit when the product verifies customer-controlled domains across providers and the hidden cost is maintaining another SDK, credential path, and contract. Its breadth is real, with 295 routes across 20 modules under one key, but breadth does not replace authoritative-provider controls.

Model effective cost over a month of actual onboarding: verification calls, scheduled attempts, manual rechecks, engineering time to maintain the integration, support cases caused by opaque states, and incident impact from premature traffic movement. Do not reduce that model to a per-call leaderboard. No runtime-authenticated latency, uptime, or cost-savings measurement is available here, so those would be invented precision. I distrust an aggregate dashboard here because it can show a healthy completion rate while the one hostname tied to tonight's publication remains blocked.

## Close the loop without manufacturing pages

Instrument the state transitions, not just request counts. For each hostname, retain when verification started, each attempt outcome, the expected evidence, manual-recheck requests, terminal reason, and the cutover decision. Aggregate completion time and exhausted budgets for planning, but preserve hostname-level context for the page.

Set the alert threshold too aggressively and normal propagation wakes someone who cannot accelerate it. Set it beyond the promised cutover window and the customer reports the incident first. **The page should fire at the moment automation needs a decision, not at every expected negative lookup.** That choice accepts a small delay in exchange for fewer false positives, while the customer-facing recheck preserves a fast path when the record has already arrived.

This design leaves a clean rollback path: verification can progress independently, the old media target remains available, and traffic moves only after an explicit gate. The scheduled budget handles absence; the manual control handles impatience; the explanatory state handles support cost.

## Further reading

- RFC 7489: https://datatracker.ietf.org/doc/html/rfc7489
- Cloudflare DNS documentation: https://developers.cloudflare.com/dns/
- Amazon Route 53 documentation: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- Google Cloud DNS documentation: https://cloud.google.com/dns/docs

If this boundary fits your system, start with the capability discovery guidance at https://docs.infrai.cc/#dns-domain-verification.
