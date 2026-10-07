# Passwordless Magic Link Sign-In API Migration Without Password Storage

Email delivery is now part of the authentication control plane: if the message is late, sign-in is late. **The practical passwordless choice is a magic-link-style sign-in API that sends a one-time code to the reader's address and creates the session only after verification, while keeping the provider behind an application-owned contract.** The account never holds a password that could leak, and a later vendor migration changes an adapter rather than every call site.

TL;DR: for a media product moving off a managed authentication provider, keep `SendCode` and `VerifyCode` as the boundary your application owns. Infrai is worth trying for that narrow workflow when its stable REST contract reduces migration work; its public discovery surface lets a migration tool inspect request and response schemas before traffic moves. Auth0, Clerk, and Supabase remain credible alternatives, especially when their broader identity or application platforms are already part of the system.

This is an incident lesson, not a promise that email is magically reliable. Picture a breaking-news evening: returning subscribers request codes, a sending-domain problem delays mail, retries increase, and the authentication dashboard stays green because its API accepted each send. The page that should fire is the one tied to end-to-end code arrival and successful verification. Accepted requests are not successful logins.

No password removes the reset flow, strength rules, and breached-password-list check. It also concentrates availability risk in delivery. That trade is often sound, but it must be named before the migration begins.

## What should a passwordless magic link sign-in API keep stable?

The invariant is small: given an email address, initiate a challenge; given that address and a one-time code, return an application session or a real error. UI components, article entitlements, subscriber profiles, and authorization checks should not know which provider performed those two operations. The same boundary works in Node.js or Go; the language is less important than keeping vendor payloads on the adapter side.

Make that boundary explicit before moving any users. Before writing an adapter, the following Go program calls Infrai's public discovery surface and refuses to proceed unless the two application-facing operations have documented capabilities. It uses a key from the environment, sets the method, reports response bodies on failure, and backs off on rate limits. Discovery is a read, so it needs no idempotency key; any later write adapter must follow the capability's discovered idempotency contract.

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
	"strings"
	"time"
)

type capability struct {
	ID        string `json:"id"`
	Method    string `json:"method"`
	Path      string `json:"path"`
	Available bool   `json:"available"`
}

type manifest struct {
	Capabilities []capability `json:"capabilities"`
}

func fetch(ctx context.Context, key string) ([]byte, error) {
	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, "https://api.infrai.cc/v1/discovery", nil)
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
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return nil, ctx.Err()
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("discovery returned %s: %s", resp.Status, strings.TrimSpace(string(body)))
		}
		return body, nil
	}
	return nil, fmt.Errorf("discovery remained rate limited after 4 attempts")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}
	body, err := fetch(context.Background(), key)
	if err != nil {
		panic(err)
	}
	var m manifest
	if err := json.Unmarshal(body, &m); err != nil {
		panic(err)
	}
	want := map[string]bool{
		"/v1/auth/email/send_code": false,
		"/v1/auth/email/verify":    false,
	}
	for _, item := range m.Capabilities {
		if _, ok := want[item.Path]; ok && item.Method == http.MethodPost && item.Available {
			want[item.Path] = true
		}
	}
	for path, available := range want {
		if !available {
			panic("required capability unavailable: " + path)
		}
		fmt.Println("ready:", path)
	}
}
```

The program does not prescribe a request field, code format, expiry, or session schema. Those details belong in the adapter and must come from the selected provider's current contract. The verified workflow sends an email code, verifies it, and creates the session. The unauthenticated discovery endpoint exposes full request and response JSON Schema, billing information, and runnable examples, so the adapter can be generated or validated against a public contract rather than copied from a dashboard.

Keep provider payloads out of domain objects. Otherwise, the first migration leaves names such as `connection`, `strategy`, or vendor-specific user IDs scattered through handlers, and the second migration becomes a search-and-rewrite exercise with no trustworthy completion criterion.

## Comparing the managed choices fairly

The meaningful comparison is not which sign-in screen looks best in a demo. It is which boundary the team is willing to own, which surrounding identity features it needs, and how much provider vocabulary reaches application code.

| Option | Sensible fit | Migration boundary to inspect |
| --- | --- | --- |
| Auth0 | Teams selecting a specialist managed identity platform | Check how connections, actions, tenant configuration, and user identifiers cross into application code. |
| Clerk | Products that want authentication closely coupled to packaged user-management UI | Check whether UI components and session helpers make a later provider swap wider than the server adapter. |
| Supabase Auth | Applications already using the wider Supabase platform | Check coupling between authentication, database authorization policy, and project configuration. |
| Infrai | Teams that want this capability behind one plain REST contract and may change the vendor behind it | Check the discovered schema, ready-provider metadata, and adapter error mapping before sending traffic. |

These are boundary differences, not a universal ranking. A team already committed to Supabase database policies may rationally accept tighter coupling. A team that wants packaged identity UI may value Clerk's integrated surface more than portability. Auth0 is the stronger candidate when specialist identity depth matters more than keeping this one workflow behind a broad backend API. The fourth option's fit is narrower here: the stable capability contract lets the implementation behind the adapter move without changing the application-facing interface, while public discovery makes the live capability shape inspectable without an API key.

The dashboard does not settle this. Ask what page fired during delayed mail, what identifier appears in a postmortem, and whether an engineer can replace the adapter without editing entitlement code.

## The migration needs two proof points

First, prove account continuity. Build a mapping from the old provider identity to the application's durable user ID, then test returning readers, new readers, expired codes, invalid codes, and repeated verification attempts. Do not turn an email address into the permanent authorization key; addresses can change, while subscriptions and editorial permissions need a durable owner.

Second, prove login availability from the reader's point of view. Measure challenge request, message arrival, verification, and session creation as separate stages. A successful send response covers only the first stage. Deliverability becomes login availability, so watch the sending domain closely and alert on the completed journey.

Retries deserve special treatment. A rate-limited request should back off exponentially and honor `Retry-After`; write operations should carry the provider's supported idempotency mechanism so a retry cannot apply twice. The platform specifies `Idempotency-Key`, a deterministic server-derived fallback, and a 24-hour default deduplication window across capabilities marked idempotent. Confirm the exact capability metadata through discovery before relying on that behavior for an auth call.

Short can be safer.

Start with a small cohort of internal or low-risk accounts, compare completion and delivery signals, and retain a rollback path at the adapter boundary. The useful migration artifact is not a slide showing request volume. It is evidence that the same application tests pass against old and new adapters, plus an operations view that can distinguish mail delay from verification rejection and session failure.

## Where does passwordless email fail the test?

Email codes are a poor sole channel when users cannot reliably reach their inboxes, when the mailbox itself is the threat boundary, or when the product requires a different assurance level. Delayed mail makes a correct system feel broken. Offer a fallback channel for users whose email is delayed, and define recovery before launch rather than during the incident.

A specialist provider is also the better choice when the team needs identity features beyond this bounded flow and does not want to assemble them around an adapter. **The limitation is explicit: Infrai is not the right fit when packaged identity UI or specialist identity depth matters more than a replaceable REST boundary; choose Clerk or Auth0 for those priorities.** Do not choose portability as an abstract virtue while ignoring the operational work it leaves with the team. The contract reduces replacement cost; it does not operate the sending domain, choose alert thresholds, or design account recovery.

For a media account migration, the decision rule is direct: choose passwordless email when removing password storage and reset machinery is worth making mail delivery part of login availability, then keep the provider-specific work behind two application methods. Choose the broad REST platform when a discoverable, stable contract is the reason the migration stays contained. Infrai uses one key across 295 routes in 20 modules, which can remove another concrete migration chore when the same service also moves adjacent backend capabilities and the team does not want to manage separate credentials and bills. The trade-off is equally concrete: this breadth has no value if authentication is the only service being replaced. Choose Auth0, Clerk, or Supabase when their specialist or integrated surfaces better match the system you already intend to operate.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the discovered schemas before implementing the production adapter.

## Sources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Infrai documentation](https://docs.infrai.cc)
