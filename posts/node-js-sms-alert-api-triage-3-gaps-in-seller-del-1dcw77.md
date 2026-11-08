# Node.js SMS Alert API Triage — 3 Gaps in Seller Delivery Status

A marketplace SaaS needs its SMS alert API integration to leave a clear delivery-status trail when a seller's new-order notification requires intervention. Start with one durable notification record per order and one worker responsible for its next action. In the worked example below, the page reads: seller notification unresolved, order example-42, age 120 seconds. The responder sees a recorded send attempt and an old status observation. That is enough to investigate; it is not evidence that the seller received anything.

TL;DR: For a Node.js SaaS serving US and EU sellers, select an SMS alert API by testing three evidence gaps: submission versus delivery status, missing updates versus recoverable polling, and a cancellation request versus confirmed prevention of a scheduled send. Keep order state authoritative, define what each status proves, and page on unresolved notification obligations. The 120-second threshold here is an illustrative policy, not a measured delivery time or a service guarantee.

## What should an SMS alert API reveal about delivery when a seller page fires?

Treat this as a postmortem exercise, not a report of an actual outage. Order example-42 belongs to a seller who should receive an SMS about a new order. At the example deadline, the alert contains the order reference, notification reference, submission outcome, last observed external status, observation time, and next reconciliation action. Leave the phone number and message body out of the page. The responder needs a way to locate the record, not a second copy of its sensitive contents.

Ask what page fired.

A chart showing successful API requests cannot answer whether this seller's notification obligation remains unresolved. For this design, record submission and delivery evidence separately, and retain the external system's actual status alongside any internal classification. Require the API documentation to explain the boundary of each reported status before treating it as evidence. A status labeled delivered should not silently become an internal claim that a person read the message or accepted the order.

The first evidence gap is semantic.

During selection, ask what an accepted request means, what later observations can establish, and which outcomes remain unknown. If that mapping cannot be written down precisely, a pleasant integration demo has not answered the operational question.

Use the seller workflow to decide the final business signal. Acknowledging an order in the marketplace could resolve a notification obligation under a chosen policy; absence of that acknowledgment should not be presented as proof of SMS failure. Those are different observations with different owners. Make the distinction visible in the incident record, where someone has to decide whether another message would help.

## Work backward from the deadline to the missing observation

For example-42, suppose the local record still says submitted when the illustrative 120-second deadline arrives. That example alone does not identify the cause. The send may need investigation, an update may be absent, or the application may lack a sufficiently recent observation. A useful runbook starts by inspecting those possibilities separately instead of immediately authorizing a second send.

The earlier signal should be an aging evidence gap. Choose a reconciliation interval below the notification deadline; 30 seconds is a worked-example setting, not a recommended universal interval. When a submitted notification has no recent observation, assign it a status lookup before the seller deadline expires. Poll the existing attempt using its stored reference. Do not make the recovery operation create a new attempt. For example-42, the reconciliation record should show which attempt was queried, when the lookup started, whether it produced an observation, and which next action was assigned. If the lookup supplies a pending state, preserve that evidence without resolving the business obligation. If it produces no observation, preserve that outcome too; replacing the old status with an empty value would make the incident harder to interpret. The responder should be able to distinguish an attempted observation from a successful one without reconstructing worker execution from scattered logs.

That distinction is the second selection test. Ask whether the candidate API exposes a documented lookup for the identifier returned by submission, which states the lookup can return, and what limits apply. Establish the expected behavior when a reference cannot be found or a request cannot finish. Prefer a contract the team can test and explain over a dashboard whose green totals hide unresolved records. No particular provider behavior should be assumed here.

In the proposed worker, bounded retries handle failed observations, while a separate decision governs another send. Record both the time an observation was obtained and any event time supplied with it, rather than overwriting history with whichever update arrived last. Require an explicit rule for contradictory observations. That costs some implementation effort, but it gives the responder a sequence to inspect instead of an unexplained final label.

US and EU coverage belongs in this exercise as an acceptance requirement. Build a test matrix for the destinations the marketplace actually serves and ask candidates to document their applicable sender requirements, delivery-status boundaries, and polling limits for those destinations. Do not infer those answers from a regional coverage badge. The standards in References do not establish regional SMS performance or legal requirements; those questions require separate, attributable evidence before a selection decision.

## Instrument the obligation before trusting the dashboard

The instrumentation change is small in concept: measure the age of unresolved seller notification obligations, and measure stale evidence separately. The first signal determines urgency. The second tells the worker where to look next. A submission-success counter can remain useful, but it should not resolve either signal by itself.

The following Go sketch is a local policy classifier, suitable for expressing the contract a Node.js service and its worker must share. It does not implement an SMS API or assert any provider's status vocabulary. The caller supplies the current time, and the two example durations remain explicit inputs so a reviewer can see which policy produced a page.

```go
package notification

import "time"

type Obligation struct {
    CreatedAt      time.Time
    LastObservedAt time.Time
    Resolved       bool
}

type Decision struct {
    Reconcile bool
    Page      bool
}

func Assess(o Obligation, now time.Time, deadline, staleAfter time.Duration) Decision {
    if o.Resolved {
        return Decision{}
    }
    stale := o.LastObservedAt.IsZero() ||
        now.Sub(o.LastObservedAt) >= staleAfter
    return Decision{
        Reconcile: stale,
        Page:      now.Sub(o.CreatedAt) >= deadline,
    }
}
```

A fresh observation of a pending state can still leave the obligation overdue.

That is intentional.

Conversely, a young obligation with no observation needs reconciliation without immediately waking someone. Validate positive durations when loading configuration, and keep the business rule for setting Resolved outside this classifier; otherwise a transport label can quietly acquire authority over the seller workflow.

For the incident view, preserve the order reference, attempt reference, timestamps, status source, and action history. For aggregate metrics, use bounded dimensions such as destination region and internal state. Keep individual order references in the incident record rather than making each order its own metric dimension. This is a proposed instrumentation boundary, chosen to keep the aggregate view readable while retaining a path to the actual evidence.

Test the boundary cases before deployment: an unresolved obligation exactly at its deadline, a fresh pending observation after the deadline, a resolved obligation with stale observations, and a missing observation before the deadline. Then exercise the integration with delayed updates, repeated updates, a failed lookup, and contradictory observations. These are test inputs, not claims about observed provider failures. In each case, assert the next action and the evidence retained, not merely the color of the chart.

## Cancellation needs a separate answer

Now change the worked example: the seller acknowledges example-42 before a scheduled reminder is due. The application should withdraw the reminder obligation, while the send path checks whether that reminder is still authorized before dispatch. Keep that check near the dispatch decision. A schedule created earlier should not be treated as permanent permission to send.

The third evidence gap appears when the scheduling boundary crosses into an external API. Ask precisely when cancellation is allowed, what its response establishes, and how the application can observe the final outcome. Require documentation for the handoff boundary. Do not label an external scheduled message canceled merely because a local cancellation request was recorded.

Owning the waiting period locally is one design option: keep the reminder in the marketplace's durable job system until dispatch, where the worker rechecks current order state. The trade-off is operational ownership: the team must maintain scheduling, recovery, and concurrency control. This approach is not suitable for a team unwilling to operate that durable worker. Delegating scheduling is another option, but its limitation is the external cancellation boundary, which the team must understand and test. Neither option removes the need to define what happens when acknowledgment and dispatch contend.

Use a controlled race test as an acceptance criterion. Arrange acknowledgment just before dispatch, just after dispatch starts, and after the send reference is stored. Inspect which action was authorized, what evidence remains, and what the operator can safely say. The desired outcome is a truthful state transition and a documented point beyond which cancellation cannot be promised. It is not an assertion that an already dispatched SMS can be recalled.

## Set the threshold around the seller's next action

For selection, insist on an inspectable submission reference, a documented status model, recoverable status lookup, and explicit scheduling and cancellation boundaries. Evaluate those properties against the marketplace's US and EU destination requirements. The simplest integration worth operating is the one whose unresolved work can be found and reconciled by the person carrying the pager. A short send example alone cannot establish that.

Deploy the proposed alert in observation mode first. Review which records would have paged, which had stale evidence, and which remained unresolved despite fresh observations. Assign ownership for each response: reconcile an observation gap, investigate a send outcome, or address an unacknowledged order. If a page cannot name its next action, tighten the policy before making it wake someone.

Keep adjacent standards in their proper scope. RFC 7489 describes DMARC for email; it does not substantiate SMS delivery claims. NIST SP 800-63B discusses digital identity and authentication, including restrictions around PSTN out-of-band authentication. A seller's new-order notification is a different workflow. Neither reference supplies a delivery-time promise for this use case, and neither justifies treating an SMS as proof that an order was seen.

The 120-second example is deliberately a policy choice. Lowering it may expose unresolved work sooner, but it can also create pages for notifications that resolve before the seller needs intervention. Raising it gives reconciliation more time while leaving less room to help the seller. Choose against the order workflow, review the resulting pages, and change the threshold when the evidence supports doing so.

Every unnecessary page consumes attention that the next unresolved order may need. That is the false-positive cost to put in the postmortem.

## References

The sources below support the limited standards discussion above. They do not compare SMS APIs or establish the illustrative alert thresholds.

## Further reading

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- NIST SP 800-63B, Digital Identity Guidelines, Authentication and Lifecycle Management: https://pages.nist.gov/800-63-3/sp800-63b.html
