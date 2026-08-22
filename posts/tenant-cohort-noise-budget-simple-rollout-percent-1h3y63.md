# Tenant Cohort Noise Budget: Simple Rollout Percentage and User Targeting Signals

Short answer: treat percentage rollout as a deterministic assignment, emit an exposure only when a request actually evaluates the flag, and compare tenant cohorts with guardrail and outcome signals that can be joined to that exact assignment. In a Node.js Express API, the handler is the easy part; preserving a stable unit of randomization, a recorded flag version, and an immediate rollback path is what keeps user targeting from producing confident-looking noise.

Do not page on the treatment-control delta alone.

## Failure mode: the global average hides the targeted tenant

For a developer-tools experiment, the first operational question is not whether a dashboard moved. It is what page would fire if the new behavior harmed one tenant cohort while the global average stayed flat. A useful rollout therefore separates a product outcome, such as successful workflow completion, from guardrails such as errors and latency, and it keeps diagnostic context available without turning every small fluctuation into an alert.

Write that page condition before choosing a rollout percentage. The condition should identify an urgent user effect, the affected tenant cohort, and the action available to the responder; a slow change in experiment conversion belongs in scheduled analysis, while a sharp violation of an established service guardrail can justify interruption. This exercise forces a useful distinction between signal quality and signal volume. Ten new panels add volume. A treatment-tagged guardrail tied to a rollback decision adds signal.

## Assignment contract: one tenant, one version, one variant

Start by choosing the assignment unit. If the experiment changes a shared developer-tools workflow, assign by tenant ID rather than request ID or user session; otherwise one organization can see both variants, caches can mix behavior, and repeated requests inflate the apparent sample size. User targeting can still determine eligibility, but eligibility and percentage assignment are separate decisions. A tenant either qualifies for the experiment or it does not, then a stable hash maps the qualifying tenant to a bucket.

The configuration needs a version as well as a percentage. Without the version on the exposure event, an analyst cannot distinguish observations made at 10% from observations made after a move to 30%, and a rollback followed by a relaunch can silently join two different populations. Martin Fowler's feature-toggle guidance distinguishes categories of toggles and describes cohort-based canary releases; the practical consequence here is to keep release control explicit rather than burying it in business logic.

A clean evaluation contract returns more than a Boolean. It should return the variant, configuration version, reason for the decision, and assignment unit. In Express, middleware can put that decision on the request context and the response path can emit the exposure after the flagged code is reached. Don't emit at login, SDK initialization, or configuration fetch time. Those events count people who never encountered the behavior.

This is the minimum event shape I would require before trusting a cohort comparison:

| Field | Purpose | Noise prevented |
| --- | --- | --- |
| `experiment_key` and `config_version` | Identify the decision that was evaluated | Mixing rollout phases |
| `tenant_id` and `variant` | Preserve the assignment unit and cohort | Cross-variant contamination |
| `decision_reason` | Separate targeted, percentage, and default decisions | Treating ineligible traffic as control |
| `exposed_at` and `request_id` | Join the decision to request outcomes | Counting configuration checks as exposure |
| outcome and guardrail fields | Separate product effect from operational harm | Promoting on one attractive metric |

Keep sensitive targeting attributes out of the event unless the analysis genuinely requires them. A derived cohort label is usually easier to govern than copying a whole user object into telemetry, and the assignment service does not need to become an identity warehouse.

## Evidence trail: one request from decision to outcome

The evaluator below shows the mechanism in Go because the decision core should be independently testable even when the serving layer is Express. The same contract can sit behind Node.js middleware: validate the tenant identity, evaluate once per request, attach the returned decision, execute the selected path, and record exposure only at the point where the behavior becomes observable to the caller.

```go
package flags

import (
	"errors"
	"hash/fnv"
)

type Config struct {
	Key             string
	Version         string
	RolloutBasisPts uint32
	EligiblePlans   map[string]bool
}

type Subject struct {
	TenantID string
	Plan     string
}

type Decision struct {
	ExperimentKey string
	ConfigVersion string
	Variant       string
	Reason        string
	AssignmentKey string
}

func Evaluate(cfg Config, subject Subject) (Decision, error) {
	if cfg.Key == "" || cfg.Version == "" || subject.TenantID == "" {
		return Decision{}, errors.New("flag key, version, and tenant ID are required")
	}
	if cfg.RolloutBasisPts > 10_000 {
		return Decision{}, errors.New("rollout basis points must be between 0 and 10000")
	}

	decision := Decision{
		ExperimentKey: cfg.Key,
		ConfigVersion: cfg.Version,
		Variant:       "control",
		Reason:        "not_eligible",
		AssignmentKey: subject.TenantID,
	}
	if !cfg.EligiblePlans[subject.Plan] {
		return decision, nil
	}

	h := fnv.New64a()
	_, _ = h.Write([]byte(cfg.Key + ":" + subject.TenantID))
	if uint32(h.Sum64()%10_000) < cfg.RolloutBasisPts {
		decision.Variant = "treatment"
	}
	decision.Reason = "percentage"
	return decision, nil
}
```

The denominator is 10,000 basis-point buckets, so a configured value of 1,000 represents a 10% assignment. That number is configuration, not an observed guarantee: a small eligible population will not land in an exact 90/10 split. I'm not sure any aggregate chart can reveal a bad assignment on its own; a pre-release test with fixed tenant IDs resolves that uncertainty much faster.

There is one subtle trap. Adding mutable attributes such as plan, region, or current user role to the hash key can move a tenant between variants during the same experiment. Use those attributes to determine eligibility, then hash a durable identifier. If the assignment algorithm itself changes, increment the configuration version and treat the resulting observations as a new phase rather than pretending continuity.

One request should tell the whole story.

Consider a hypothetical tenant assigned to treatment at configuration version 7. The tenant's first three requests evaluate the flag, but only the third reaches the changed repository-analysis path; that third request is the exposure, while the first two are merely decisions. The changed path completes, then its request outcome carries the same request ID and tenant ID into telemetry. Later, an operator lowers the percentage and the same tenant remains in treatment because stable assignment is preserved. If an analysis instead counted all three evaluations as exposures, joined outcomes by timestamp alone, or grouped version 7 with a later targeting rule, it would manufacture extra observations and blur the population being compared. None of those mistakes necessarily raises an application error. The numbers still render. That is precisely why an exposure trail must be verified on individual requests before anyone interprets the aggregate — the most dangerous experiment failure can be a polished chart backed by the wrong denominator.

Bad data wins quietly.

## Signal review: reconstruct harm before reading lift

Assume the rollout ended badly and work backward. The plausible failure modes are not limited to an elevated error count: treatment tenants may retry more often, latency may rise only for repositories above a certain size, a workflow may complete while requiring extra corrective actions, or a noisy tenant may dominate the aggregate. The comparison should answer whether the change helped the intended outcome, whether it violated an operational guardrail, and whether the result survives segmentation by the cohorts that mattered before launch.

For browser-facing portions of the developer tool, Core Web Vitals provide standardized user-experience signals: Largest Contentful Paint, Cumulative Layout Shift, and Interaction to Next Paint. The web.dev guidance evaluates these at the 75th percentile, segmented by mobile and desktop. That does not make p75 a universal experiment rule, and server-side API latency needs its own service objective; it does show why a percentile and a declared population are more informative than a single mean. The cohort definition, observation window, and guardrail direction should be written down before looking at the result.

Dashboards are evidence, not arbitration. A global green line can conceal a severe regression in one targeted plan, while dozens of per-segment panels can manufacture apparent discoveries from ordinary variation. Compare the predeclared tenant cohorts first, inspect exposure counts and assignment balance, then use request-level diagnostics to explain a change. If the experiment has too few exposed tenants to separate signal from noise, the honest result is “undetermined,” not “ship because nothing paged.”

The alert should also correspond to an action. Page only when a guardrail breach is urgent, user-affecting, and actionable during the current shift; route slow-moving experiment outcomes to review instead. What page fired? If nobody can answer that before rollout, the monitoring design is incomplete.

## How should a simple feature flags API verify rollout percentage and user targeting?

Run an offline assignment test over fixed tenant IDs before deploying. Verify that the same key and configuration version always return the same variant, ineligible plans always remain in control, missing identity fails without exposure, and boundary percentages behave correctly. Then deploy the evaluator with the treatment path disabled and confirm that decisions, exposures, request outcomes, and configuration versions join without duplicate counting.

At the initial live percentage, check four things before expansion: eligible tenants are the only experiment population; exposure volume matches actual encounters rather than all requests; guardrails are evaluated per variant and important tenant cohort; and the rollback control has been exercised. A rollback should change configuration so new evaluations select control while leaving historical events intact. Do not rewrite old treatment events, reuse the old version for a materially different rule, or delete the evidence needed for the eventual review.

The catch is that percentage flags are not suitable when tenants cannot tolerate mixed behavior across a multi-step workflow, when assignment identity is unavailable, or when the change alters persisted data in a way that control cannot safely read. In those cases, use an explicit allowlist, a migration state machine, or a maintenance boundary instead of a probabilistic rollout. Stick with a simple on/off release control when the experiment cannot produce a defensible outcome metric; extra targeting would create ceremony, not information.

Expansion is a decision, not a timer. Proceed only when telemetry joins correctly, the predeclared outcome is interpretable, no guardrail requires rollback, and the sample covers the tenant cohorts the decision claims to represent. Otherwise hold or return to control, write down what evidence was missing, and repair the measurement plan before trying again.

## References

- https://martinfowler.com/articles/feature-toggles.html
- https://web.dev/articles/vitals
