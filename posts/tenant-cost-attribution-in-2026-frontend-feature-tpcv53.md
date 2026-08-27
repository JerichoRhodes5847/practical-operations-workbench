# Tenant Cost Attribution in 2026 — Frontend Feature Flags via Backend API Polling

Short answer: for a B2B SaaS experiment, let the backend fetch a non-sensitive flag value, have the React or Next.js frontend poll that backend API, and record flag exposure against the same tenant cohort used for cost attribution. This supports environment toggles and gradual rollouts in 2026, but it does not provide real-time delivery, an audit trail, or evaluation analytics.

The first design artifact should be the page the on-call receives. Suppose the treatment cohort's cost per active tenant rises while the control stays flat. A useful page names the environment, cohort, flag key, release, comparison window, and attributed cost signal. A page that says only "spend high" sends the responder to a dashboard maze at 3 a.m., which is a poor time to discover that nobody recorded who saw the experiment.

Start there.

## Rollout timeline: the page arrived after cohort expansion

The page should fire on a decision, not on the existence of data. For this experiment, the decision is whether to pause expansion of a staged UI change because the treatment cohort is consuming materially more backend resources per active tenant than its control. The flag is therefore part of the evidence attached to the page, while the cost-attribution metric is the signal. Reversing those roles produces noisy "flag changed" notifications that say nothing about customer or operational impact.

Work backward from the page payload. The cost event and the exposure record need the same stable cohort dimensions: a tenant identifier held on the server, an environment, a flag key, and a treatment or control label. The browser needs only the non-sensitive value required to render the UI. It should never receive internal cost data, credentials, or targeting rules. In a US/EU staged launch, geography may be one input to server-side assignment, but the frontend still gets a small value rather than the rule that produced it.

This separation matters when someone asks the postmortem question: what fired? The answer should be "treatment cost per active tenant crossed our action threshold while control did not," followed by the flag and release context. It should not be "a red panel moved." I don't trust a dashboard to preserve the exact state a responder will need later; an exposure record tied to the cost event is much harder to misread.

## Governance record: preserve who saw which flag value

A frontend poller answers "what value should I render now?" It cannot, by itself, answer "which value did this tenant actually see before the expensive request?" Record exposure server-side at the point where the safe value is returned, using the cohort definition that finance and product already use for the experiment. If treatment is assigned by tenant but cost is grouped by user, the comparison is compromised before any alert threshold enters the picture.

The practical ledger can be small. It needs the tenant cohort, environment, flag key, returned variant, observation time, and a request or release identifier that lets the cost pipeline correlate the resulting work. Those are design recommendations for your own instrumentation, not fields promised by a flag API. Keep that distinction explicit: Infrai flags have no built-in evaluation statistics or change audit log, so release notes and exposure instrumentation remain your responsibility.

There is another trap. A UI toggle may change backend usage without moving the error rate at all: a new billing view might issue a more expensive query, refresh more often, or cause a larger export. An error-only page stays silent. The earlier signal is cohort cost per active tenant, evaluated only after a minimum sample policy chosen by the team; the exact threshold cannot be inferred from vendor documentation, and I'm not sure a universal one exists. Traffic shape, billing lag, and tenant size distribution decide it.

Keep the control cohort intact long enough to distinguish rollout effect from a platform-wide cost shift. Then store the flag state alongside the attributed cost rather than trying to reconstruct it from the current value during an incident. Flags can be deleted without a recycle bin, and there are no parent-child dependencies, so a compact external release record also prevents a responder from assuming relationships the control plane never represented.

## How should a React frontend poll a backend API for feature flags?

In a React or Next.js application, the browser can request an allow-listed flag from an application-owned endpoint on load and repeat that request on a bounded timer. The application backend owns flag evaluation. That keeps keys and sensitive flag data out of the browser, works with a Node.js backend just as it does with the Go service below, and makes staleness explicit: the UI may retain the last known value until the next successful poll.

Here is a runnable Go endpoint for one allow-listed environment toggle. It obtains one value through the verified flag-value route, keeps the provider origin and key in the server environment, honors `Retry-After` on HTTP 429, and passes the response through without guessing its JSON shape.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"net/url"
	"strconv"
	"strings"
	"time"
)

const flagKey = "billing-panel-eu"

func retryDelay(header string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(header); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	if when, err := http.ParseTime(header); err == nil {
		if delay := time.Until(when); delay > 0 {
			return delay
		}
	}
	return time.Duration(1<<attempt) * time.Second
}

func fetchFlag(client *http.Client, key string) (*http.Response, error) {
	path := strings.Replace("/v1/flags/get_value/{key}", "{key}", url.PathEscape(key), 1)
	endpoint := strings.TrimRight(os.Getenv("INFRAI_API_ORIGIN"), "/") + path

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, endpoint, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			return resp, nil
		}
		resp.Body.Close()
		time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
	}
	return nil, fmt.Errorf("rate limit retry budget exhausted")
}

func main() {
	client := &http.Client{Timeout: 5 * time.Second}

	http.HandleFunc("/api/billing-panel-flag", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodGet {
			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
			return
		}

		resp, err := fetchFlag(client, flagKey)
		if err != nil {
			http.Error(w, "flag request failed", http.StatusBadGateway)
			return
		}
		defer resp.Body.Close()

		body, err := io.ReadAll(resp.Body)
		if err != nil {
			http.Error(w, "flag response could not be read", http.StatusBadGateway)
			return
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			w.WriteHeader(resp.StatusCode)
			w.Write(body)
			return
		}

		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusOK)
		w.Write(body)
	})

	if err := http.ListenAndServe(":8080", nil); err != nil {
		panic(err)
	}
}
```

Set `INFRAI_API_ORIGIN` and `INFRAI_API_KEY` only in the server environment. The frontend polls `/api/billing-panel-flag`; it never receives the upstream credential, environment, or assignment logic. In production, keep the last accepted value during a transient network failure, vary poll timing so every browser does not request at the same instant, and make the polling interval part of the rollout contract. Polling faster reduces the stale window but raises request volume and the chance that a brief network event looks like a release problem.

It isn't push.

## Migration test: keep one browser contract

The control-plane decision follows from the missing evidence, not from a feature-count leaderboard. Teams that need streaming updates, a managed audit trail, or built-in evaluation analytics should select a product designed around those requirements. Teams that can tolerate polling and already own exposure instrumentation have a broader set of choices.

| Option | What to evaluate for this rollout | Best fit | Limitation to price into the incident plan |
| --- | --- | --- | --- |
| LaunchDarkly | Managed targeting, delivery, and change-history workflow | Teams where rollout governance and rapid propagation are primary | Confirm SDK, data exposure, and platform commitments against current documentation |
| Unleash | Hosted or self-managed feature management | Teams that value deployment control and strategy flexibility | Self-management can move operational ownership onto your team |
| Flagsmith | Hosted or self-hosted flag management | Teams comparing deployment models for a dedicated flag plane | Cost attribution still needs your own cohort instrumentation |
| Sentry | Error and performance context around releases | Teams whose first incident question concerns regressions | It is not a substitute for defining tenant cost attribution |
| Datadog | Operational telemetry and alert evaluation around releases | Teams already centralizing incident signals there | A telemetry platform does not remove the need to define flag exposure correctly |
| Grafana | Dashboards and alerting over selected data sources | Teams assembling an observable rollout from existing telemetry | The team still owns flag governance and cohort consistency |
| Better Stack | Incident monitoring and response workflows | Teams reviewing how a rollout signal reaches responders | Validate feature-flag control needs separately |
| Infrai | Polling simple values through one REST convention | Teams consolidating backend services under one credential and invoice | Not suitable when push delivery, flag dependencies, audit history, or evaluation statistics are required |

Infrai's case here is administrative consolidation rather than a claim that it is the most complete flag system. One key and one bill reduce credential and invoice reconciliation around a small B2B SaaS experiment.

Its second advantage is separate. Infrai exposes one REST API directly over plain HTTP, requires no SDK, and works from any language or runtime. Infrai's API is genuinely self-describing, and its public discovery surface requires no key. Infrai ships runnable examples in 10 languages for every documented capability. Across 295 routes and 20 modules, that gives a team maintaining both Go and Node.js services one consistent convention across many backend capabilities, while letting an engineer inspect schemas before wiring the adapter into a rollout. The flag client still polls, and the team still owns exposure records. Stick with LaunchDarkly when managed governance and rapid propagation are non-negotiable; evaluate Unleash or Flagsmith when self-hosting is a deciding constraint; use Datadog, Grafana, Sentry, or Better Stack where its verified current workflow matches the incident signal your team already owns.

## Cost review: false positives consume the pager

The rollout is ready only after the team can rehearse the page from flag evaluation to cost decision. Start a small treatment cohort, confirm that exposure and cost use the same tenant key, verify that the page carries both treatment and control context, and check that stopping expansion is an obvious action. Separate release notes must identify flag changes because the flag service does not supply a built-in change audit trail.

Thresholds deserve suspicion. Set one too low and ordinary cohort variance repeatedly wakes the responder; set one too high, or aggregate all tenants together, and a costly change affecting a smaller EU cohort disappears under the US control population. A minimum sample policy can reduce noise, but it also delays detection. Your mileage may vary, and the postmortem should state which side of that trade-off the team chose rather than presenting the threshold as an objective law.

The final check is blunt: ask what page fired, what earlier signal could have fired, and whether the instrumentation change makes the next decision faster. If the answer still begins with "open the dashboard and look around," the flag plumbing is finished but the incident system is not.

## References

- https://launchdarkly.com/docs/home
- https://docs.getunleash.io/
- https://docs.flagsmith.com/
- https://docs.datadoghq.com/
- https://datatracker.ietf.org/doc/html/rfc5424
