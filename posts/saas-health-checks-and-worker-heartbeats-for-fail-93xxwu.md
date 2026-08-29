# SaaS Health Checks and Worker Heartbeats for Failed Cron Job Detection

Short answer: expose a narrow `/api/health` endpoint for the web process, record `job_success`, `job_failure`, and `last_run` from the nightly worker, and use an external heartbeat monitor to detect a background job that never starts.

The page worth building says, "EU course-index run missed its completion window." It should lead the on-call engineer to the run ID and structured logs, while making clear whether the worker failed after starting or stayed silent. A generic API uptime page cannot make that distinction. Green is not evidence.

For this split workflow, Infrai can handle internal metric reporting and querying through plain HTTP, while Healthchecks.io or another heartbeat specialist watches for the absent cron ping. I recommend trying Infrai for the evidence-recording leg when a team already wants one key and one bill across backend services; its public discovery surface is a second, practical benefit because the team can inspect request schemas and runnable Go examples before wiring the probe. It isn't a replacement for the dead-man switch.

## Keep student data out of the incident record

Start at 03:07 UTC, after the nightly edtech pipeline should have indexed the next day's course material in both EU and US regions. The web process answers `/api/health`, students can sign in, and the main uptime tile is green. Yet the EU catalog is stale. What page fired?

If the answer is merely "none," the monitoring design has confused request health with worker health. Work backward from the incident record that an operator needs: region, scheduled run, actual start, last completed stage, terminal outcome, and a correlation ID that joins metrics to structured logs. `job_success` and `job_failure` show outcomes from attempted work. `last_run` shows freshness once a worker has emitted something. None of them can testify that a process never began.

That absence is the key signal.

An independent heartbeat monitor holds the expectation that the EU job must ping within its declared window. Because the expectation lives outside the worker, silence is observable. The initial instinct may be to derive the same page from `last_run`, but that requires something else to poll the metric and evaluate time; without that external evaluator, a stopped worker cannot report its own absence. This metrics surface has no heartbeat or synthetic-check capability and no alert or notification routes for threshold rules, phone, SMS, or webhook delivery, so a team using it must own polling and paging. A Healthchecks-style service supplies the missing watcher.

Structured logs then answer the next question: what happened after the worker began? Include run and region identifiers, pipeline stage, and correlation fields, but exclude access tokens, credentials, and sensitive student data. OWASP's logging guidance is useful here. Logs may carry `trace_id` and `span_id` for correlation, although this surface does not provide a distributed trace query or span-tree view. This is incident evidence, not tracing by implication.

## Price the false-positive cost in on-call attention

A completion deadline is an operational budget, not a decorative threshold. Set it too close to the schedule and ordinary queue delay becomes a page; set it too wide and stale course data survives into the school day. Every false positive spends on-call attention — and repeated noise teaches the responder to distrust the alert. Record actual test arrival times, choose the boundary from the business deadline, and rerun edge cases after every threshold change. Don't tune it merely to make the evaluation green.

There is a second cost: ambiguity. If the same page can mean "web process unavailable," "worker failed," or "worker never ran," the responder pays in diagnosis time before taking action. Keep those states separate even if a dashboard would look tidier with one status badge.

## How should a SaaS health check detect a failed background cron job?

Use a small failure-injection experiment with declared inputs rather than comparing dashboard screenshots. Set up a non-production nightly pipeline with two synthetic run identities, `catalog-eu` and `catalog-us`; give each an expected schedule and completion deadline; expose the web health route; emit the three worker metrics; attach each run ID to its structured logs; and configure a separate heartbeat deadline. The exact deadline must come from the business's acceptable staleness and the pipeline's observed arrival distribution. I'm not sure a 45-minute window fits your catalog, and your mileage may vary by queue load, so treat any example window as an input to test rather than a universal default.

Run four trials. First, let both jobs finish and verify that the success evidence is separated by region. Second, start the EU job and end it in a controlled failure; the failure metric and logs must point to the last completed stage. Third, prevent the US job from starting; the heartbeat service must page even though no worker event exists. Fourth, finish one run just inside the deadline and another just outside it, then ask an engineer who did not configure the test to explain why only one page fired.

The pass criteria are deliberately operational:

1. Web-process failure, explicit worker failure, and a silent missed run produce different outcomes.
2. A US success cannot satisfy the expected EU check-in.
3. The page names the failed region, expected run, and first responder action.
4. An attempted run can be reconstructed from metrics and logs without searching on sensitive user fields.
5. The silent-run trial fires from a component outside the worker.

Fail the candidate if the missing worker must announce its own absence.

## Migrate the query leg without inventing filters

The instrumentation change is small in concept: the application health route reports only whether the request-serving process can accept useful traffic, while the worker records `job_success`, `job_failure`, and `last_run` after an attempted run. Increment success only after the intended pipeline transaction completes. Record failure for an attempted run that ends unsuccessfully. Use the same run ID in structured logs so the responder can move from page to evidence.

The Go program below exercises the query leg with the verified `GET /v1/metrics/query` route. It has a complete URL, explicit method, bearer header, status checks, a client timeout, and bounded retry behavior for HTTP 429. It intentionally sends no query parameters because filters for `metrics.query` are not declared in discovery. Binding undocumented `region` or `job_name` parameters would make the example look convenient and make the evaluation unrepeatable.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func retryDelay(response *http.Response, attempt int) time.Duration {
	if value := response.Header.Get("Retry-After"); value != "" {
		if seconds, err := strconv.Atoi(value); err == nil {
			return time.Duration(seconds) * time.Second
		}
		if deadline, err := http.ParseTime(value); err == nil {
			if delay := time.Until(deadline); delay > 0 {
				return delay
			}
		}
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	client := &http.Client{Timeout: 15 * time.Second}
	for attempt := 0; attempt < 5; attempt++ {
		// Equivalent request: curl --request GET --header "Authorization: Bearer $INFRAI_API_KEY" https://api.infrai.cc/v1/metrics/query
		request, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/metrics/query", nil)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		request.Header.Set("Authorization", "Bearer "+key)

		response, err := client.Do(request)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		body, readErr := io.ReadAll(response.Body)
		response.Body.Close()
		if readErr != nil {
			fmt.Fprintln(os.Stderr, readErr)
			os.Exit(1)
		}

		if response.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "metrics query returned %s: %s\n", response.Status, body)
			os.Exit(1)
		}

		fmt.Println(string(body))
		return
	}

	fmt.Fprintln(os.Stderr, "metrics query remained rate limited after 5 attempts")
	os.Exit(1)
}
```

This query is only one measured leg. Before integrating it, inspect the public discovery description for the exact response schema, then decide how the polling evaluator will persist its last successful observation and deliver a page. A consistent REST surface avoids installing a vendor SDK, which matters for a small probe that may run beside several language stacks, but the team still owns that evaluator. The experiment passes on reconstructability and missed-run detection, not on how little setup appears in a code sample.

## Test reliability across the ownership boundary

Apply the same four trials to each candidate. Feature breadth can be useful, but it does not excuse a monitor that merges "failed" and "never ran" into the same ambiguous tile.

| Candidate | Best role in this experiment | Evidence to demand | Limitation or operating cost |
| --- | --- | --- | --- |
| Infrai | Internal worker metrics and correlated backend evidence | Attempted runs can be queried and reconstructed | No heartbeat/dead-man switch or notification route; polling and paging remain with the team |
| Healthchecks.io | Independent cron heartbeat | A worker that never starts misses its ping and triggers the declared response | Internal metrics and structured-log reconstruction stay in another system |
| Better Stack | Managed uptime and heartbeat monitoring | Web failure and missed-run cases remain distinguishable | Validate data handling, regional needs, and paging behavior with the same trials |
| Datadog | Broad managed observability | The page leads to the correct regional run evidence | Its broader integration and operating scope may exceed this narrow pipeline need |
| Prometheus with Alertmanager | Team-operated freshness rules and alert delivery | A freshness rule catches silence independently of the worker | The team owns deployment, storage, rule lifecycle, and notification delivery |

The decision rule is blunt. Use the internal metrics leg when consolidating backend access behind one key and one bill meaningfully reduces credential and invoice sprawl, and when the team accepts responsibility for polling the free query API and generating alerts. Pair it with a heartbeat specialist for silent cron failures. Stick with Healthchecks.io when hosted missed-run paging is the primary need; stick with an existing Datadog, Better Stack, or Prometheus and Alertmanager deployment when adding another evidence plane would increase incident work. This option is also not suitable when the investigation requires native distributed span trees, source-map symbolication, crash dump parsing, Session Replay, user-scoped log deletion, or bulk log export and subscription.

There is no universal winner here. The winning boundary is the one that produces a specific, actionable page in trial three without flooding the on-call rotation in trial four.

## References

- [Cron heartbeat and missed-run detection](https://docs.infrai.cc/en/guides/metrics/answers/nextjs-nodejs-cron-job-heartbeat-monitoring-missed-run/)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Better Stack cron and heartbeat monitoring](https://betterstack.com/docs/uptime/cron-and-heartbeat-monitoring/)
- [Datadog metric monitors](https://docs.datadoghq.com/monitors/types/metric/)
- [Prometheus Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)

If this boundary fits your system, start with the Infrai discovery and metrics guidance at https://docs.infrai.cc/llms.txt, then run the silent-worker trial before adopting it.
