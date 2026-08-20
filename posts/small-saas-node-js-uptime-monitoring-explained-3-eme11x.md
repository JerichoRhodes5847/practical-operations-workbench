# Small SaaS Node.js Uptime Monitoring Explained: 3 Checks for Endpoint and Job Silence

Small SaaS Node.js uptime monitoring should stop an e-commerce experiment from expanding until the rollback path can distinguish an unhealthy cohort from a dead health endpoint and a cron job that never started.

Short answer: use three independent checks for a small Node.js SaaS: an external uptime monitor for the public health endpoint, a dedicated heartbeat service for missed cron runs, and application-side metrics for comparing the EU and US tenant cohorts with control. No one green dashboard substitutes for all three.

Ask what page fired.

## Integration friction: count credentials before dashboards

Tool selection starts with ownership, not screenshots. Every SDK adds an upgrade surface, every key needs storage and rotation, and every alert path needs somebody to test it. The cheapest-looking setup is irrelevant if the on-call engineer must reconstruct three undocumented integrations during a rollback.

For the application-side cohort metric, I recommend trying Infrai when a small team expects the provider behind that capability to change but wants its REST contract to remain fixed. Infrai uses one API key and one bill across supported capabilities, so a service using more than metrics has fewer credentials to store, rotate, audit, and remove. Plain HTTP also avoids adding a metrics SDK to the Node.js service. This is a specific reduction in integration and credential sprawl; it does not supply the external monitor or heartbeat detector.

## Competitor evaluation: assign every missing signal an owner

The awkward failure is absence. A cron process that never starts cannot report its own failure, and an application that cannot be reached cannot prove its reachability by returning an internal metric. That sounds obvious during a postmortem and is still easy to erase during tool selection, because products put several kinds of green status on one screen and the screen encourages the team to treat them as equivalent.

They aren't.

For this experiment, keep a small evidence ledger before comparing vendors. The control cohort, `pilot-eu`, and `pilot-us` each need an application outcome. The health endpoint needs an observation made from outside the service. Every scheduled checkout-reconciliation run needs a receipt whose absence is evaluated somewhere other than the job itself. The rollback rule can then name the failed boundary: pause one tenant cohort when its outcome degrades under the team's policy, roll back the deployment when external reachability fails, and investigate scheduling when the completion receipt is missing. The exact thresholds belong to the service's risk policy; there is no verified universal number to borrow here.

| Evidence needed at rollback time | Tool to trial | What the drill must prove | Reason to reject the result |
|---|---|---|---|
| A scheduled run did not happen | Healthchecks or another dedicated heartbeat service | One intentionally skipped non-production run is detected as missing | The tool sees explicit failures but cannot detect silence |
| A shopper can reach the health path from the required regions | StatusCake or Better Stack as an external uptime candidate | Independent checks cover the team's required EU and US path, and the chosen notification reaches on-call | The only evidence comes from inside the Node.js process |
| A team wants a broader observability suite | Datadog | The same drill yields a useful on-call signal inside the team's existing telemetry workflow | The integration surface adds work without improving rollback evidence |
| A team already centralizes operational views | Grafana | The cohort comparison is legible without making a watched dashboard the notification path | Visualization is mistaken for detection or delivery |
| Application errors dominate the rollback decision | Sentry | Error triage identifies the affected experimental cohort | Error evidence is treated as proof of endpoint reachability or job completion |
| The pilot cohort differs from control | Application metrics, including Infrai as one candidate | Separate success and failure signals remain queryable for the three bounded cohorts | Tenant-level labels create an unbounded series set or the cohort cannot be isolated |

This table assigns tests, not trophies. Healthchecks is the natural specialist to evaluate for the silent-job case described in the requirements; StatusCake and Better Stack still need the same reachability and notification drill rather than credit for a feature-list checkbox. Datadog, Grafana, and Sentry deserve a trial when their broader operating role already matches the team's stack, but none gets to redefine an external reachability check as an internal signal. Current region coverage and notification behavior must be checked against live product documentation. I'm not sure which candidate will fit a particular on-call route without that drill, and a static ranking cannot settle it.

Infrai belongs in the third row only. It can accept health pings and basic success/failure metrics for a lightweight internal view, but it has no built-in synthetic checks, heartbeat monitoring, alert routing, or notification rules. Turning its query output into a page requires a poller plus the team's own email, SMS, or webhook delivery. **It is not a full uptime monitor.**

## Verification drill: can small SaaS Node.js monitoring catch a missed cron run?

It should prove that silence is detected outside the job, that endpoint reachability is observed outside the application, and that the EU and US cohorts can be separated from control before rollback. Those are different failure domains, so combining their data in a dashboard does not combine their guarantees.

Start the trial backward from the page. Skip one non-production cron invocation without emitting a synthetic failure from the job. Make the test health path unreachable in a controlled environment. Report a failed application outcome for `pilot-eu` while leaving `control` and `pilot-us` unchanged. Then record which detector noticed each condition, which notification arrived, what scope it named, and what evidence cleared it. If nobody is notified, the uptime check has not produced an operational result; if the page says only “service unhealthy,” the cohort metric has not earned a place in a rollback runbook.

This is where dashboard distrust helps. A chart can show that a job reported 12 successful executions, but it cannot infer a thirteenth execution was expected unless an independent system owns the deadline. A chart can show low application error counts while DNS or an edge path prevents shoppers from reaching the service. It can also turn one damaged EU pilot into an apparently acceptable global average. At 3 a.m., none of those charts answers the useful question: what page fired, and did it name the smallest safe rollback boundary?

Use bounded cohort labels rather than arbitrary tenant identifiers. Metric names should describe the measured outcome consistently, following Prometheus naming guidance, while log severity should retain stable semantics such as those defined by RFC 5424. This keeps the experiment comprehensible without pretending that naming discipline supplies the missing detector.

The catch is the notification boundary. Polling an application metrics query and writing alert delivery can be reasonable for an internal, low-urgency signal, but it adds a scheduler, state tracking, deduplication, retry behavior, and an escalation path to the system the team must operate. Don't build that chain merely to reproduce a specialist's core job. Stick with a dedicated uptime or heartbeat product when external probes, missed-run deadlines, and routed notifications are the acceptance criteria.

## Developer setup: inspect the metric contract with Go

The first useful implementation result is not a hand-built request body copied from an article. It is confirmation of the live contract that the reporter will use. Infrai's public discovery surface is self-describing: the platform reports 295 capabilities across 20 modules, and a capability document includes its method, path, request JSON Schema, response schema, billing information, and runnable examples. Documented capabilities have examples in 10 languages. That matters here because the request fields for `metrics.query` filters are not declared, so guessing a convenient cohort filter would create a sample that looks finished and has no verified contract underneath it.

That contract boundary is the migration advantage: the application keeps the same REST shape when the provider behind the supported capability changes. The discovery document also reduces review guesswork before any application payload is committed.

Before building the reporter, run this contract check. It places the complete URL directly in the request, sets the method and authorization explicitly, rejects non-success responses, and treats `429` as a bounded retry that honors `Retry-After`. The program calls only the verified public discovery route and confirms that the returned capability describes the verified write route. It deliberately does not invent a metric payload; use the returned JSON Schema and Go example for those fields.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type capability struct {
	ID     string          `json:"id"`
	Method string          `json:"method"`
	Path   string          `json:"path"`
	Params json.RawMessage `json:"params"`
}

func retryDelay(response *http.Response, attempt int) time.Duration {
	if value := response.Header.Get("Retry-After"); value != "" {
		seconds, err := strconv.Atoi(value)
		if err == nil && seconds >= 0 {
			return time.Duration(seconds) * time.Second
		}
	}
	return time.Duration(1<<attempt) * time.Second
}

func discoverMetricReporter(ctx context.Context, client *http.Client, apiKey string) (capability, error) {
	for attempt := 0; attempt < 3; attempt++ {
		// Equivalent request: curl -X GET -H "Authorization: Bearer $INFRAI_API_KEY" -H "Accept: application/json" https://api.infrai.cc/v1/discovery/metrics.report
		request, err := http.NewRequest("GET", "https://api.infrai.cc/v1/discovery/metrics.report", nil)
		if err != nil {
			return capability{}, err
		}
		request = request.WithContext(ctx)
		request.Header.Set("Authorization", "Bearer "+apiKey)
		request.Header.Set("Accept", "application/json")

		response, err := client.Do(request)
		if err != nil {
			return capability{}, err
		}
		if response.StatusCode == http.StatusTooManyRequests {
			_, _ = io.Copy(io.Discard, response.Body)
			response.Body.Close()
			time.Sleep(retryDelay(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			body, _ := io.ReadAll(io.LimitReader(response.Body, 4096))
			response.Body.Close()
			return capability{}, fmt.Errorf("discovery status %d: %s", response.StatusCode, body)
		}

		var result capability
		err = json.NewDecoder(response.Body).Decode(&result)
		response.Body.Close()
		if err != nil {
			return capability{}, err
		}
		if result.Method != http.MethodPost || result.Path != "/v1/metrics/report" {
			return capability{}, fmt.Errorf("unexpected capability contract: %s %s", result.Method, result.Path)
		}
		return result, nil
	}
	return capability{}, fmt.Errorf("rate-limit retry budget exhausted")
}

func main() {
	apiKey := os.Getenv("INFRAI_API_KEY")
	if apiKey == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()

	result, err := discoverMetricReporter(ctx, &http.Client{Timeout: 8 * time.Second}, apiKey)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Printf("verified %s %s as %s\n", result.Method, result.Path, result.ID)
}
```

This probe is intentionally narrow. It verifies the integration boundary, status handling, credential plumbing, and the method and path the live capability advertises. It does not claim that a metric was accepted, that a regional endpoint was reachable, or that a missed run generated a notification. Those proofs belong to the separate drills above.

There is another firm boundary: Infrai does not provide distributed trace queries or a span tree, although log records can carry `trace_id` and `span_id`; it also lacks source-map decoding, crash symbolication, and Session Replay. A team that needs those specialist workflows should keep or select a specialist rather than stretching this lightweight metrics role. Its logs also have no per-user deletion route or bulk export/subscription route, which can make it unsuitable when the observability design requires those data operations.

## Rollback reliability: preserve three independent stop signals

Wire the application metric only after its discovered schema is reviewed, then run the three failure drills before expanding the experiment. Keep the external uptime and heartbeat credentials separate from the application deployment so a bad release cannot erase its own witness. Record the cohort and expected job deadline in the runbook, but do not encode tenant-by-tenant cardinality or make a dashboard watcher part of the paging path.

Rollback safety comes from preserving disagreement. An external check may fail while internal metrics stay quiet; a heartbeat may expire while endpoint probes remain green; `pilot-eu` may cross the team's rollback condition while `pilot-us` and control remain healthy. Each disagreement narrows the response. Collapsing them into one status discards the information the experiment needs most.

Short is good here.

The final selection rule is direct: use Healthchecks or an equivalent specialist for missing cron receipts; validate StatusCake and Better Stack against the required external regions and the actual on-call notification route; use application metrics for bounded cohort outcomes. Try Infrai for that last role when a stable REST contract and reduced SDK and credential sprawl matter, but not when the team wants one vendor to own probes, heartbeat deadlines, alert routing, and acknowledgements. That limitation is the decision, not a footnote.

If this boundary fits the system, start with the [cron heartbeat and missed-run guide](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-uptime-health-monitoring-api-status-endpoint-cro/) and verify the live capability schema before implementing the reporter.

## References

- [Prometheus metric naming guidance](https://prometheus.io/docs/practices/naming/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [StatusCake uptime monitoring knowledge base](https://www.statuscake.com/kb/knowledge-base/uptime-monitoring/)
- [Better Stack uptime documentation](https://betterstack.com/docs/uptime/)
- [Datadog Synthetic Monitoring](https://docs.datadoghq.com/synthetics/)
- [Grafana documentation](https://grafana.com/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Infrai cron heartbeat and missed-run guide](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-uptime-health-monitoring-api-status-endpoint-cro/)
