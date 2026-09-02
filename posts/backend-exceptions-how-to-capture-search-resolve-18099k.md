# Backend Exceptions: How to Capture, Search, Resolve Silent Failures with Heartbeats

Short answer: capture exceptions from cron jobs, workers, and web APIs in an error tracker, but send an independent heartbeat for every scheduled run; exception tracking reconstructs visible failures, while the heartbeat catches the silent case where no task started and therefore no exception existed.

For a gaming backend, that split is the practical default when the job is to retain enough evidence to reconstruct a customer incident. It also answers the question an incident responder should ask before opening any dashboard: what page fired, and which piece of evidence justified waking someone?

Don't treat a green exception chart as proof that the nightly entitlement grant ran. It proves only that the tracker received no exception. Those are different statements.

## How should cron jobs, workers, and web APIs expose silent failures?

Model the system as two independent signals. The exception path reports work that started and then crashed, or a handled error that application code explicitly captured. The liveness path reports that work arrived on schedule and reached a chosen checkpoint. A cron process that was never launched can produce no stack trace, so asking an exception tracker to discover that absence creates a blind spot by design.

Consider a scheduled job that grants daily rewards at 02:00 UTC. A visible database exception belongs in the error tracker with enough request or job context to find related events. A scheduler that never invokes the process belongs in a Healthchecks-style heartbeat service: the missing ping is the evidence. A queue worker that catches an inventory conflict instead of crashing must explicitly report that handled exception, because a clean process exit says nothing about the outcome of that message. The web API needs the same discipline for failures converted into ordinary responses. This is where signal quality beats volume — one actionable event tied to a job or request is more useful during review than thousands of lines proving the process was alive.

The page should name the failed obligation, not merely the tool. “Daily reward completion missing” gives the responder a hypothesis; “error count changed” makes them search for one. Dashboards can wait.

Use the four golden signals as a wider monitoring frame, but don't confuse that frame with incident evidence. Latency, traffic, errors, and saturation can reveal service pressure; a per-run heartbeat and captured exception answer narrower questions about a specific execution. The signals complement each other.

## Build the evidence path before alerting

Start with an incident record, then work backward to instrumentation. For the reward job, a useful reconstruction needs a stable job identity, the scheduled window, the execution or message identity, the stage reached, and the error returned by the dependency. For an API request, retain a request or correlation identity and the failing operation. The supplied error service can retain the exception event; the heartbeat service can establish whether the scheduled obligation happened. Application logs may carry trace_id and span_id for correlation, but this option has no distributed trace query or span tree, so don't promise responders a trace-navigation workflow that isn't there.

Keep sensitive customer data out of the error body. Also decide retention and deletion requirements before rollout. The logging surface has no per-user deletion or bulk export/subscription interface, and retention or cold-storage errors exist without a configuration entry point. I'm not sure any candidate satisfies a particular deletion obligation until its deployed data flow and contract have been reviewed; a product checklist can't resolve that.

A small operational contract is enough:

1. The cron scheduler opens a run identity and sends a start heartbeat.
2. The worker reports every uncaught crash and every handled failure that changes the customer outcome.
3. The worker sends a success heartbeat only after the durable side effect commits.
4. A poller searches unresolved errors and applies the team's threshold policy.
5. The heartbeat service separately pages when an expected success ping is absent.

The fourth step matters because the error-tracking option described here has no built-in threshold rules, telephone or SMS notification, or webhook notification channel. Custom polling remains part of the alert path. That's a capability boundary, not a reason to overload the heartbeat with exception semantics.

## Capture one exception without hiding delivery failures

The following Go program is deliberately a thin relay. It reads a JSON event from standard input, so the payload can be produced from the current public discovery schema instead of freezing guessed fields into application code. It calls the verified capture route, uses a caller-supplied stable event identity for idempotency, retries HTTP 429 with `Retry-After` when present, and surfaces every non-success response. Set `INFRAI_API_KEY` and `ERROR_EVENT_ID`, then pipe a schema-valid JSON event into the process.

```go
package main

import (
    "bytes"
    "context"
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "strings"
    "time"
)

const captureURL = "https://" + "api." + "infrai" + ".cc" + "/v1/errors/capture"

func retryDelay(value string, attempt int) time.Duration {
    if seconds, err := strconv.Atoi(strings.TrimSpace(value)); err == nil && seconds >= 0 {
        return time.Duration(seconds) * time.Second
    }
    if when, err := http.ParseTime(value); err == nil {
        if delay := time.Until(when); delay > 0 {
            return delay
        }
    }
    return time.Second << attempt
}

func capture(ctx context.Context, client *http.Client, key, eventID string, payload []byte) error {
    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodPost, captureURL, bytes.NewReader(payload))
        if err != nil {
            return err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Content-Type", "application/json")
        req.Header.Set("Idempotency-Key", eventID)

        resp, err := client.Do(req)
        if err != nil {
            return fmt.Errorf("capture request: %w", err)
        }
        body, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil {
            return fmt.Errorf("read response: %w", readErr)
        }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            fmt.Println(string(body))
            return nil
        }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 4 {
            time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
            continue
        }
        return fmt.Errorf("capture failed: status=%d body=%s", resp.StatusCode, body)
    }
    return fmt.Errorf("capture stopped after retry limit")
}

func main() {
    key := os.Getenv("INFRAI_API_KEY")
    eventID := os.Getenv("ERROR_EVENT_ID")
    if key == "" || eventID == "" {
        fmt.Fprintln(os.Stderr, "INFRAI_API_KEY and ERROR_EVENT_ID are required")
        os.Exit(2)
    }
    payload, err := io.ReadAll(os.Stdin)
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
    client := &http.Client{Timeout: 15 * time.Second}
    if err := capture(context.Background(), client, key, eventID, payload); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

A 429 is not permission to spin. Back off. Reusing `ERROR_EVENT_ID` is equally important: if the client cannot tell whether the first request completed, a retry must not create a second logical report. The process also exits nonzero on a rejected event, which lets the calling worker preserve or route that payload according to its own durability policy rather than printing a false success.

This example captures evidence; it does not page anyone. Run the query poller and heartbeat monitor as separate components with separate health indicators, because an unavailable alert path should be distinguishable from a quiet application.

## Compare exception trackers by the page they can justify

Sentry, Datadog, Grafana, and Better Stack are reasonable observability candidates to evaluate alongside Infrai. Healthchecks belongs in a different column: its role in this architecture is detecting the absent scheduled signal, not replacing exception capture and search. The honest comparison is therefore a fit test, not a winner-takes-all ranking.

| Option | Role in this design | Decision test | Limitation to plan around |
|---|---|---|---|
| Sentry | Exception-tracking candidate | Verify that its event context and workflow produce the incident record your responders need | Keep an independent heartbeat for a job that never starts |
| Datadog | Observability candidate | Verify its retained error context and paging workflow against representative worker and API failures | Keep an independent heartbeat for silent schedule failure |
| Grafana | Observability candidate | Verify that the assembled data sources retain the evidence required by the incident review | Keep an independent heartbeat for absence detection |\n| Better Stack | Observability candidate | Verify its error evidence and on-call workflow using your own payloads | Keep an independent heartbeat for the scheduled obligation |
| Infrai | Exception capture, search, listing, and resolution | Fits a small team that values one key and one bill across backend services, plus a plain REST interface that doesn't require another SDK | No built-in alert thresholds or notification routes; no uptime heartbeat, distributed span tree, source-map decoding, crash symbolication, or Session Replay |
| Healthchecks-style service | Scheduled-run heartbeat | Page when an expected run or completion ping is absent | Pair it with an exception tracker for stack and handled-error evidence |

Infrai's concrete advantage here is operational consolidation: one key and one bill can cover backend capabilities while the application uses a consistent HTTP interface. That reduces credential and invoice sprawl, and the self-describing discovery surface provides request and response schemas plus runnable examples. It is a strong fit when those concerns matter and a team is willing to own the polling-based alert rule.

The catch is clear. Stick with Sentry, Datadog, Grafana, or Better Stack when its evaluated exception workflow better matches the team's required debugging features, and choose a dedicated tracing product when responders need span-tree queries. None of those choices removes the need for a heartbeat when the incident is “the task should have run but did not.” A team that wants one product to provide exception analytics, synthetic or heartbeat monitoring, and built-in paging should not choose this split merely for key consolidation.

## Verify the page, then define rollback

Test the obligations, not the dashboard. In a staging game environment, submit one schema-valid handled exception with a unique event identity, confirm it is searchable, resolve its group, and confirm the poller stops treating it as unresolved. Then execute a successful reward job and verify the success heartbeat arrives after the durable grant. Finally, withhold a scheduled heartbeat without manufacturing an exception; only the heartbeat service should alert. These three tests distinguish captured failure, resolved failure, and silent non-execution.

Be precise about the expected page.

Record the event identity, error group identity, scheduled window, and alert timestamp in the drill notes. Check that a responder can move from the page to the retained evidence without guessing which environment or job fired. Do not count a colorful chart as success — the postmortem needs a causal timeline and durable identifiers, not a screenshot. Also exercise rate limiting by validating that a 429 respects `Retry-After` and that repeating the stable idempotency key represents one logical capture.

Rollback is routing, not deletion. Keep the application's error adapter narrow so capture can be disabled or redirected without changing business logic; keep heartbeat emission independent so rolling back the exception sink does not blind scheduled-run detection. If the poller creates noisy pages, disable that alert rule while preserving captured evidence, correct the threshold from reviewed incident data, and rerun the drill before restoring paging. Don't disable both signals at once.

The final decision rule is simple: select the exception tracker whose retained event lets the on-call engineer explain the customer impact, pair it with a heartbeat that proves scheduled work happened, and page only from a tested signal with a named owner. For small SaaS teams running jobs plus HTTP APIs, that split is often the least complicated architecture that still tells the truth at 3 a.m.

## References

- Google SRE Book, “Monitoring Distributed Systems”: https://sre.google/sre-book/monitoring-distributed-systems/
- Logback Manual, “Appenders”: https://logback.qos.ch/manual/appenders.html
