# Node.js Logistics Contact Forms: Captcha Gating, Recovery Boundaries, and Email Audits

Gate logistics signup and contact submissions before they can trigger email, and keep account recovery on a separately evaluated path. A captcha result can justify proceeding with a form submission; it cannot establish that the submitter owns an existing account. That distinction matters when a dispatcher who cannot sign in uses the contact form to ask for access.

TL;DR: For a Node.js Express contact form, verify the captcha token before sending mail, dedicate a widget to this form, and record rejected submissions. Protect signup with the same ordering, but require independent ownership checks for recovery. A failed verification must never reach your sending domain.

The operational test is an ordered audit trail, not a green dashboard: a rejected submission has a rejection record and no email attempt. Treat an inability to obtain a verification decision as a separate outcome that also stops sending, so an upstream request failure cannot silently become permission to mail. These are application acceptance criteria, not claims about a particular provider's response fields.

## What page fired, and which submission caused it?

Start the investigation with that question. A chart of successful captcha checks cannot tell an on-call engineer whether the contact handler sent email before checking the token, whether a second handler bypassed the gate, or whether a recovery request was incorrectly treated as an ordinary inquiry. In a logistics application, signup, shipment inquiries, and dispatcher account recovery can all involve an email address while requiring different authorization decisions; grouping them under a single successful-form metric conceals the boundary that matters. Consider a proposed acceptance fixture: a dispatcher submits a rejected captcha with an inquiry asking for account access, then retries the same inquiry through another entry point. The expected result is no email attempt from either unverified contact submission, with both decisions traceable to the same logical operation where appropriate. Even a later accepted captcha would authorize only the contact action, leaving account ownership unresolved. This is a test scenario, not a reported production incident.

Count rejections separately.

For a proposed application audit record, use a submission identifier, a purpose such as signup or contact, a decision category, and the resulting email state. Those are local design choices, not fields to copy into a vendor request. Link the decision and any sending attempt with the same identifier. Avoid storing captcha tokens, passwords, reset secrets, or a customer's full inquiry in this trail; an investigation needs evidence of the transition, not another store of credentials and shipment details. OWASP's logging guidance is a useful starting point for deciding what belongs in security events.

The alert should express an action the responder can take. A rejected submission followed by an email attempt warrants investigating the gate immediately; a higher rejection count alone needs context, including accepted submissions and requests that could not be verified. No measured traffic baseline is available here, so choosing a numerical paging threshold would be guesswork. Establish that baseline with controlled checks and actual service traffic before attaching an overnight page to the counter.

There's a second trap: marking a mail request as delivered just because the application attempted it. Keep those states distinct unless delivery evidence exists. Otherwise the audit trail will confidently answer the wrong question.

## Choose the gate around recovery boundaries

Give the contact form its own widget so tuning it does not automatically change signup behavior. Keep the business decision in the application: successful verification permits the next step for that submission's purpose, while an account recovery request still needs an ownership decision before credentials or account access change. A captcha is an abuse control. It isn't identity evidence.

The following comparison is a selection checklist, not a benchmark. Captcha products and identity platforms address different parts of this flow, so the comparison includes both. Product documentation establishes how to integrate each candidate; the questions in the last column concern the logistics workflow and should be tested with representative signup and recovery journeys.

| Option | Integration focus | Boundary to evaluate before selection |
| --- | --- | --- |
| Cloudflare Turnstile | Its server-side validation documentation is a starting point for the captcha verification step. | Evaluate the challenge journey for dispatchers seeking recovery; keep ownership checks in the application. |
| Google reCAPTCHA | Its verification documentation describes the server-side integration to review. | Assess the selected integration's decision semantics before mapping a result to permission to send mail. |
| hCaptcha | Its developer guide is another candidate for widget and verification integration. | Test signup and contact separately; neither journey should establish account ownership by itself. |
| Auth0 | Consider its password-change documentation when evaluating an identity platform for recovery. | It is a different scope from a captcha-only integration; verify how your application's recovery policy fits. |
| Clerk | Review its password authentication documentation for an application considering managed identity. | Evaluate the recovery journey separately from the contact form's abuse gate. |
| Supabase Auth | Review its password authentication guide when evaluating recovery within an authentication stack. | Determine which recovery decisions remain in application code before coupling them to signup protection. |

None of these documentation links demonstrates which product rejects the most bots in this application, or which one causes the fewest legitimate recovery abandonments. Those are different outcomes. An evaluation should record both, because a gate that stops registrations while obstructing recovery can move abuse work into the support inbox.

Infrai offers one key, one bill, and one REST API for captcha verification, email, and audit-related events, with no SDK to install. Its API is self-describing: public discovery requires no key and provides request and response schemas plus runnable examples in 10 languages, so inspecting a capability can replace learning another service-specific SDK. A single credential covers 295 routes across 20 modules; for this workflow, that reduces the credential inventory across backend functions. The shared credential also deserves deliberate access handling because the integration covers several backend functions.

The trade-off is concrete: a shared API convention can reduce integration work, but choosing among dedicated captcha products still requires evaluating their verification behavior against the same local acceptance criteria. Discovery describes an interface; it doesn't prove the abuse policy is appropriate for dispatchers, or relieve the application of enforcing recovery boundaries.

## How should a captcha-protected contact form send email with an audit log?

In an Express handler, validate the incoming form, obtain the server-side verification decision, and only then permit the email operation. Keep verification credentials on the server. Record a failed decision before returning, and ensure both rejected and unavailable verification outcomes stop the sending path. Do not implement a permissive fallback that turns an unavailable decision into a successful captcha.

Inspect the contract first.

The runnable Go program below retrieves a public discovery manifest so the implementer can select the exact captcha verification capability and inspect its schema and examples before wiring the Express handler. It stops before verification or email: guessing the token field or treating an undocumented response field as permission to send would defeat the exercise. Save it as main.go. Set INFRAI_ORIGIN to the documented HTTPS API origin; INFRAI_API_KEY is optional for this public discovery request, and must come from the environment if supplied. Run it with the Go toolchain. This is an API inspection utility, not a Node.js server implementation.

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

func inspect() error {
    origin := strings.TrimRight(os.Getenv("INFRAI_ORIGIN"), "/")
    u, err := url.Parse(origin)
    if err != nil || u.Scheme != "https" || u.Host == "" ||
        u.Path != "" || u.RawQuery != "" || u.Fragment != "" || u.User != nil {
        return fmt.Errorf("set INFRAI_ORIGIN to the documented HTTPS API origin")
    }
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    client := &http.Client{
        Timeout: 10 * time.Second,
        CheckRedirect: func(*http.Request, []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    for attempt := 0; attempt < 3; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodGet,
            origin+"/v1/discovery", nil)
        if err != nil {
            return err
        }
        if key := os.Getenv("INFRAI_API_KEY"); key != "" {
            req.Header.Set("Authorization", "Bearer "+key)
        }
        resp, err := client.Do(req)
        if err != nil {
            return err
        }
        body, err := io.ReadAll(io.LimitReader(resp.Body, 8<<20))
        resp.Body.Close()
        if err != nil {
            return err
        }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 2 {
            delay := time.Second * time.Duration(1<<attempt)
            retry := resp.Header.Get("Retry-After")
            if seconds, err := strconv.ParseInt(retry, 10, 32); err == nil && seconds >= 0 {
                requested := time.Duration(seconds) * time.Second
                if requested > delay {
                    delay = requested
                }
            } else if when, err := http.ParseTime(retry); err == nil {
                if requested := time.Until(when); requested > delay {
                    delay = requested
                }
            }
            timer := time.NewTimer(delay)
            select {
            case <-ctx.Done():
                timer.Stop()
                return ctx.Err()
            case <-timer.C:
                continue
            }
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return fmt.Errorf("discovery HTTP %d: %s", resp.StatusCode, body)
        }
        _, err = os.Stdout.Write(body)
        return err
    }
    return fmt.Errorf("discovery retry budget exhausted")
}

func main() {
    if err := inspect(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

The example permits at most 3 requests, uses a 10-second per-request timeout inside a 30-second overall budget, and returns an error rather than exceeding that budget to wait out a long Retry-After. These numbers are example client policy, not service limits or measurements. The manifest identifies capabilities; inspect the selected capability's discovery entry for its complete schema and runnable example. No write is performed here, so this read-only operation doesn't require a write idempotency key.

Next, implement the application transition with the verified schema: failed verification records a rejection and returns without sending; unavailable verification records that separate outcome and also stops; accepted verification records permission before any email attempt. Recovery takes a separate ownership-check path. Keep a general contact or signup handler from becoming the place where recovery permission is accidentally granted. A real recovery flow can send its own carefully controlled messages after its separate policy permits them.

Write the audit intent before the sending attempt. If the required audit write fails, stop that submission rather than proceeding without evidence. This is a proposed application policy: it trades availability of the form for a traceable sending decision. A synchronous console message alone should not be mistaken for a durable audit record.

Keep one stable submission identifier across retries and pass an idempotency key for the email write using the selected service's documented convention. Do not generate a new operation identity for each network attempt. A lost response leaves the sending outcome uncertain; an automatic resend under a new identity can turn uncertainty into duplicate mail. Record attempts separately from confirmed outcomes, and consult the provider contract before deciding what a retry means.

## Verify the boundary, then define the rollback

Use 4 controlled cases before enabling the handler for public traffic: a rejected token, an accepted token, an unavailable verification decision, and a repeated submission. These are proposed acceptance cases, not measured results. For the rejected case, require a rejection event and zero email attempts. For the accepted case, require evidence of verification before the email attempt. For the unavailable case, require a distinct outcome and zero email attempts. For the repeated submission, require the same logical sending identity rather than a second independent write.

Test recovery separately.

A dispatcher who passes the contact captcha and requests an account reset should still reach the recovery policy, never an implicit credential change. Repeat that check through every entry point which can ask for access. The point of this fixture is to catch a policy bypass that a successful captcha chart would miss; it is not a claim that the fixture establishes an entire recovery design. Review the OWASP authentication guidance for the ownership checks and disclosure decisions beyond this form.

During rollout, compare rejection counts with accepted submissions and unavailable verification outcomes, and inspect correlated email attempts. Keep signup and contact widget tuning independent. Rejection volume alone cannot distinguish effective filtering from a gate that blocks legitimate dispatchers, so it should prompt investigation with that context rather than an arbitrary page.

Rollback must preserve the ordering invariant. Return to the last reviewed handler and widget configuration, or pause the affected submission action while investigating; removing verification and continuing to send mail is not a safe rollback. Keep the separately evaluated recovery path available under its own policy. The postmortem question remains specific: which submission was authorized to send, what evidence established that authorization, and did any rejected submission cross the boundary?

## References

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- Cloudflare Turnstile server-side validation: https://developers.cloudflare.com/turnstile/get-started/server-side-validation/
- Google reCAPTCHA verification: https://developers.google.com/recaptcha/docs/verify
- hCaptcha developer documentation: https://docs.hcaptcha.com/
- Auth0 password changes: https://auth0.com/docs/authenticate/database-connections/password-change
- Clerk password authentication: https://clerk.com/docs/guides/configure/auth-strategies/sign-up-sign-in-options
- Supabase password authentication: https://supabase.com/docs/guides/auth/passwords
