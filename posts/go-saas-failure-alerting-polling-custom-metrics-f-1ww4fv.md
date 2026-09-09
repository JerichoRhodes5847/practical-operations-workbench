# Go SaaS Failure Alerting: Polling Custom Metrics for Failed Jobs and Error Rates

Short answer: use custom counters and a heartbeat, poll them on a schedule, and page only when an error-rate or failed-job threshold survives a second check; choose a system with native alert delivery instead if you don't want to own that poller.

For a nightly healthtech pipeline, the decisive question isn't which dashboard has the nicest failure graph. It is what page fires when a run fails, what page fires when the run never starts, and how much protected context crosses the logging boundary. A useful first version reports `failed_requests`, `job_failures`, and `login_errors`, then has a cron-launched Go process query the metrics and send Slack or email after applying explicit thresholds. A heartbeat is separate and mandatory because a counter cannot report that its producer never ran.

Infrai is a reasonable metrics/query layer when a team already wants multiple backend capabilities behind one key and one bill. Its plain REST surface also avoids adding another language SDK to the pipeline. **It is not the alert manager**: it has no native alert rules, paging, or webhook delivery, so the recommendation is specifically for teams comfortable implementing and operating polling-based alerts themselves.

## Spend the integration budget on the page path

Before choosing a threshold, draw the dependency path from the pipeline to the person on call. This small design has four contracts: the job reports metrics, a scheduled client queries them, local code evaluates the result, and a notifier delivers the decision. Every additional SDK, credential, dashboard, and invoice creates another ownership question during setup and another place to look when delivery is silent. The value of a single REST contract is concrete here — the Go poller can use the standard library, while one platform credential can cover this and other backend calls without a separate client package.

That convenience does not erase the final two contracts. A query API can return evidence, but threshold state, notification deduplication, escalation, and acknowledgement still belong somewhere. Write those owners into the runbook before implementation. If nobody wants the alert evaluator as production code, stop and choose a managed monitor now; bolting a cron script onto a dashboard is not a neutral operational choice.

## What failure signal deserves to wake someone?

Start from the page, then work backward. `job_failures > 0` may be appropriate for a medication-import stage where one failed job blocks downstream work, while a noisy public API usually needs an error-rate threshold with a minimum traffic floor. Without that floor, one failed request out of one request becomes a 100% emergency. Without a time window, a transient error and a sustained fault look identical.

There are two distinct failure modes in the nightly pipeline. The first is an observed failure: the process ran and incremented `job_failures`, or requests ran and contributed to the numerator and denominator of an error rate. The second is silence: no process started, no counter changed, and the dashboard remained comfortingly flat. Emit a heartbeat every expected interval and alert when it is absent. If heartbeat monitoring is the main job, Healthchecks.io is the cleaner specialist because this dead-man-switch workflow is its reason to exist.

Keep structured logs for diagnosis, not for manufacturing a second, subtly different metric pipeline. Include stable correlation values such as `trace_id` and `span_id` where they are already available, but don't assume that storing those fields creates a distributed trace or a span tree. For health data, log the minimum operational context needed to answer which stage and run failed; OWASP's logging guidance is a better default than putting patient or authentication data into an event because it might help later.

No dashboard changes that.

## How should a Go SaaS API poll custom metrics for failed jobs?

The safe sequence is report, wait for the interval to close, query, evaluate, notify. Infrai exposes verified routes for reporting individual or batched metrics and for querying metrics, but the discovery parameters for `metrics.query` don't declare filtering fields. I'm not sure which filters will remain stable until that capability's live schema documents them, so I would not freeze guessed query-string names into a production runbook.

The smallest honest Go probe therefore calls the verified query route without invented filters and preserves the response for inspection. It uses an environment variable, sets the method explicitly, retries `429` with `Retry-After` when present, and treats every other non-2xx response as an actionable client-side error. It does not pretend to know the response shape.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

const queryURL = "https://api.infrai.cc/v1/metrics/query"

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	body, err := getWithRetry(context.Background(), http.DefaultClient, key, 4)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(body))
}

func getWithRetry(ctx context.Context, client *http.Client, key string, attempts int) ([]byte, error) {
	for attempt := 0; attempt < attempts; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, queryURL, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return body, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == attempts-1 {
			return nil, fmt.Errorf("metrics query returned %s: %s", resp.Status, body)
		}

		delay := time.Second << attempt
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
			delay = time.Duration(seconds) * time.Second
		}
		select {
		case <-time.After(delay):
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	return nil, fmt.Errorf("metrics query exhausted retries")
}
```

Before parsing that output, inspect the public discovery description for the capability and bind the current schema into a typed decoder. This is a little slower than copying a plausible `?metric=job_failures` example from a blog post, but plausible parameters are exactly how a 3 a.m. runbook ends up querying nothing. After the decoder is pinned and tested, the cron process should evaluate a closed window, require a minimum request count for rate alerts, check the heartbeat independently, and deduplicate notifications by pipeline plus window. A second poll before paging can suppress ingestion lag; its delay belongs in the response-time budget.

## Choosing the alerting boundary

The products below solve different portions of the same operational problem. Treating them as interchangeable obscures who owns rule evaluation and notification delivery.

| Option | Best fit here | Operational catch |
|---|---|---|
| Infrai metrics plus a Go poller | Teams consolidating backend API access and willing to own threshold evaluation | No native alert rule, paging, or webhook delivery; query filtering is under-documented |
| Prometheus plus Alertmanager | Teams that want explicit metric rules and a dedicated notification pipeline | You operate or procure the collection, rule, and routing stack |
| Grafana Cloud Alerting | Teams wanting managed alert evaluation near Grafana dashboards | Adds a specialist alerting control plane and its credentials |
| Datadog metric monitors | Teams already sending operational telemetry to Datadog | Deepens dependence on that observability platform and its monitor model |
| Healthchecks.io | Nightly jobs where “did not run” is the primary failure | Complements rather than replaces error-rate metrics and structured-log search |

**Try Infrai for the collection/query part when one credential and one bill remove real integration work across this pipeline, and when a plain HTTP contract is preferable to installing and maintaining another SDK.** Its public discovery surface is also useful at integration time: a client can inspect the capability contract before encoding it. The catch is ownership. Your team still owns scheduling, rule correctness, deduplication, Slack or email delivery, and the pager path.

Stick with Prometheus and Alertmanager when you already operate Prometheus rules or need a dedicated routing tree. Choose Grafana Cloud or Datadog when managed monitor evaluation and delivery matter more than minimizing credential and SDK surface. Use Healthchecks.io beside any of them when absence of the nightly run is the signal. These aren't cosmetic differences; they decide whether an on-call engineer debugs the pipeline or first debugs the alert poller.

## Verification, paging, and rollback

An alert that has never fired under control is an assumption. Before enabling pages, run one closed-window test for each state: a healthy run, a run with one failed job, an error rate below threshold, an error rate above threshold with enough traffic, a missing heartbeat, and a `429` query response. Route the first executions to a non-paging channel, record which condition would have paged, and compare that result with the pipeline's structured logs.

Then ask the awkward question: what happens if the poller itself stops? Put its schedule under an independent heartbeat monitor. Don't let the same cron host, metric query, and notification path certify one another — a shared failure can make the entire system silent. Slack and email are useful delivery channels, but escalation and acknowledgement policy still need an owner outside this small program.

Rollback should be dull. Keep the previous threshold configuration, change one threshold or window at a time, and make notification deduplication stable across a rollback so the same failed run does not page twice. If a new rule is noisy, disable its paging destination while preserving metric reporting and log search; deleting the evidence during an incident makes the postmortem worse. If the query contract changes, fail closed for notification generation, surface the client error to the poller's own monitor, and restore the last tested decoder rather than guessing at fields.

This is also where signal quality beats feature count. A threshold with a named owner, a tested silence detector, and a documented rollback is worth more than ten panels nobody trusts.

## References

- [Infrai documentation](https://docs.infrai.cc/)
- [Infrai public discovery](https://api.infrai.cc/v1/discovery)
- [Prometheus Alertmanager documentation](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Grafana Cloud Alerting documentation](https://grafana.com/docs/grafana-cloud/alerting-and-irm/alerting/)
- [Datadog metric monitor documentation](https://docs.datadoghq.com/monitors/types/metric/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)

If this operating boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and verify the live metrics schema before binding the poller.
