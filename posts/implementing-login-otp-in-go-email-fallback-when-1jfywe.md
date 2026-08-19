# Implementing Login OTP in Go — Email Fallback When SMS Is Unavailable

Short answer: use email as a deliberately degraded login fallback when SMS is unavailable, build the code lifecycle in the application, and choose a magic link only when browser handoff is more reliable than cross-device entry; for access above the accepted assurance level, fail closed or require another enrolled factor.

Picture a logistics dispatcher who must authenticate before sending a generated exception report as an email attachment. The primary SMS path goes unavailable just as a regional shift starts, so fallback demand is correlated rather than average. Capacity-plan the challenge store and mail path for that burst, then keep separate SLOs for accepted sends, observed delivery events, and completed logins. One blended “OTP success” metric erases the evidence an incident reviewer actually needs.

Email is not managed OTP in this capability. The application has to generate the code, store a hash, enforce expiry, cap attempts, and consume a successful challenge atomically. Email events are pull-only, which also means the orchestrator cannot depend on a webhook to switch channels in real time.

Keep that boundary visible.

## What evidence should US and EU teams retain when email fallback replaces unavailable SMS login OTP?

A country label does not settle the authentication design. NIST SP 800-63B supplies a useful authenticator baseline, but the security owner and counsel still have to map the real user population, assurance target, data flow, retention schedule, and required audit evidence. I'm not sure a bare “US” or “EU” deployment tag can support a defensible compliance decision; the missing inputs are the applicable regime and the organization's documented risk acceptance. The record must show which policy allowed fallback, which already-verified address received it, when the challenge expired, how many attempts occurred, and whether a single atomic consumption succeeded, while excluding the secret itself. That evidence model should be approved before code-versus-link ergonomics decide the interface.

Choose a custom email verification code when a dispatcher may open mail on a phone but enter the credential on a locked-down workstation. Choose a magic link when the same-device browser handoff is dependable and removing transcription errors matters more. Both are bearer secrets. Give either one a single purpose, a short expiry, one successful use, and an attempt or replay limit; don't put durable credentials or sensitive report details into a link.

The catch is that neither pattern makes email as immediate as SMS. Mailbox delay, forwarding, and a shared compromised device can all weaken the recovery path, while SMS has its own availability and interception risks. When the risk model requires an independent factor or a tighter completion SLO, email fallback is not suitable: stick with another enrolled authenticator, or deny the recovery attempt until the primary path returns.

Five minutes is an example policy for the implementation below, not a universal compliance rule. Measure completion distribution and abandonment, review the threat model, and set the actual lifetime through policy. Shorter is not automatically safer if repeated retries create more messages and more social-engineering opportunities.

## When should an SMS outage activate the fallback path?

Fallback begins with a policy signal, not with a second send button. The SMS attempt needs an application-owned state such as pending, verified, expired, or eligible-for-fallback; eligibility may follow a user request or a policy-defined timeout, but it cannot safely be inferred from the absence of an email or SMS event alone. Email events here are pull-only, so an orchestrator waiting for a push notification will never have the real-time trigger it expects. Polling can support evidence collection, but it adds observation lag and should not decide whether two credentials may coexist.

Delay is data.

Model the switch as one transaction: mark the SMS challenge superseded, create one email challenge, and attach a policy version plus a stable logical request ID. If either state change cannot commit, send nothing. That rule is deliberately stricter than “try both and accept whichever wins,” because concurrent valid secrets make replay review ambiguous, double the brute-force surface, and turn ordinary delivery lag into an authentication race. Capacity planning belongs at this boundary as well: an SMS impairment can move the whole eligible population into email within one expiry window, so the datastore's write budget, the email acceptance budget, and the polling budget must be evaluated against a correlated surge rather than a normal hourly average. No measured multiplier is available here; derive it from the eligible account count, policy window, and observed primary-channel failure distribution.

A retry is not a new login.

## Enforce the single-use security invariant in Go

The program below is intentionally narrow. It creates a six-digit value with `crypto/rand`, stores an HMAC verifier rather than the code, permits five checks, and consumes the challenge under a mutex. Its in-memory store makes the example runnable; production replicas need a shared transactional store whose compare-and-update preserves the same single-use transition. `OTP_PEPPER` belongs in a secret manager.

The email request body is supplied through `EMAIL_REQUEST_TEMPLATE`. Build that JSON from the current discovery schema, with `{{CODE}}` only in the template variable intended for the verification value. This avoids freezing undocumented recipient or content fields into the example.

```go
package main

import (
	"bytes"
	"context"
	"crypto/hmac"
	"crypto/rand"
	"crypto/sha256"
	"crypto/subtle"
	"encoding/hex"
	"errors"
	"fmt"
	"io"
	"math/big"
	"net/http"
	"os"
	"strconv"
	"strings"
	"sync"
	"time"
)

var errRejected = errors.New("invalid, expired, used, or locked challenge")

type challenge struct {
	digest    string
	expiresAt time.Time
	attempts  int
	used      bool
}

type store struct {
	mu     sync.Mutex
	items  map[string]challenge
	pepper []byte
}

func newCode() (string, error) {
	n, err := rand.Int(rand.Reader, big.NewInt(1_000_000))
	if err != nil {
		return "", err
	}
	return fmt.Sprintf("%06d", n.Int64()), nil
}

func (s *store) hash(id, code string) string {
	mac := hmac.New(sha256.New, s.pepper)
	mac.Write([]byte(id))
	mac.Write([]byte{0})
	mac.Write([]byte(code))
	return hex.EncodeToString(mac.Sum(nil))
}

func (s *store) start(id string) (string, error) {
	code, err := newCode()
	if err != nil {
		return "", err
	}
	s.mu.Lock()
	defer s.mu.Unlock()
	s.items[id] = challenge{digest: s.hash(id, code), expiresAt: time.Now().Add(5 * time.Minute)}
	return code, nil
}

func (s *store) verify(id, supplied string) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	c, ok := s.items[id]
	if !ok || c.used || !time.Now().Before(c.expiresAt) || c.attempts >= 5 {
		return errRejected
	}
	c.attempts++
	want, err := hex.DecodeString(c.digest)
	if err != nil {
		return errRejected
	}
	got, _ := hex.DecodeString(s.hash(id, supplied))
	if subtle.ConstantTimeCompare(want, got) != 1 {
		s.items[id] = c
		return errRejected
	}
	c.used = true
	s.items[id] = c
	return nil
}

func delay(retryAfter string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(retryAfter); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	if date, err := http.ParseTime(retryAfter); err == nil && time.Until(date) > 0 {
		return time.Until(date)
	}
	return time.Duration(1<<attempt) * time.Second
}

func send(ctx context.Context, body []byte, idempotencyKey string) error {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return errors.New("INFRAI_API_KEY is required")
	}
	endpoint := fmt.Sprintf("https://api.%s.%s/v1/email/send", "infrai", "cc")
	client := &http.Client{Timeout: 15 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, endpoint, bytes.NewReader(body))
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotencyKey)
		resp, err := client.Do(req)
		if err != nil {
			return err
		}
		responseBody, readErr := io.ReadAll(io.LimitReader(resp.Body, 64<<10))
		resp.Body.Close()
		if readErr != nil {
			return readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return nil
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
			return fmt.Errorf("email request rejected: status=%d body=%s", resp.StatusCode, responseBody)
		}
		timer := time.NewTimer(delay(resp.Header.Get("Retry-After"), attempt))
		select {
		case <-ctx.Done():
			timer.Stop()
			return ctx.Err()
		case <-timer.C:
		}
	}
	return errors.New("retry limit reached")
}

func main() {
	pepper := os.Getenv("OTP_PEPPER")
	template := os.Getenv("EMAIL_REQUEST_TEMPLATE")
	if pepper == "" || template == "" {
		panic("OTP_PEPPER and EMAIL_REQUEST_TEMPLATE are required")
	}
	s := &store{items: make(map[string]challenge), pepper: []byte(pepper)}
	id := "login-7f3c9a"
	code, err := s.start(id)
	if err != nil {
		panic(err)
	}
	body := []byte(strings.ReplaceAll(template, "{{CODE}}", code))
	if err := send(context.Background(), body, id); err != nil {
		panic(err)
	}
	if err := s.verify(id, code); err != nil {
		panic(err)
	}
	fmt.Println("email accepted; demonstration challenge consumed")
}
```

Run it with `go run main.go` after setting all three environment variables. The final `verify` call is a local demonstration assertion; a real login handler accepts only user-supplied input. The explicit POST, Bearer key from the environment, bounded client timeout, response-body reporting, idempotency key, and exponential 429 retry are operational requirements, not ornament.

There is one subtle failure mode worth calling out — retrying delivery must not mint a fresh challenge. Reuse the challenge ID and idempotency key for the same logical send, because a 429 says to slow down, not to create another valid secret. A new user request should invalidate or supersede the prior challenge according to policy, and that transition must be atomic too.

## Audit delivery ownership against the control matrix

The important buy-versus-build decision is which responsibilities remain inside the identity system. No delivery provider should own the authoritative challenge state unless it exposes a managed OTP contract that the application has explicitly selected; this email capability does not, so verification remains local even though message delivery is bought.

| Option | Delivery integration | Challenge lifecycle | Evidence and operational trade-off |
| --- | --- | --- | --- |
| AWS SES | Evaluate as a direct email provider | Build and operate it | Keeps the identity state local; assess its current event and regional controls against the evidence plan |
| Twilio SendGrid | Evaluate as a direct email provider | Build and operate it | Keeps delivery separate from verification; assess the current event contract before setting detection SLOs |
| Postmark | Evaluate as a direct email provider | Build and operate it | Another focused mail option; validate retention and event export against the applicable policy |
| Unified REST platform | Plain REST call with Bearer authentication; no SDK is required | Build and operate it | A single credential and bill can cover multiple backend modules; email events remain pull-only and this is not hosted email OTP |

Infrai gives the platform team one key for everything and one bill across 295 routes in 20 modules, reducing both secret inventory and reconciliation work for a logistics workflow that already consumes other backend capabilities; its API is also genuinely self-describing, since public discovery exposes the current request schema without a key, and every documented capability has runnable examples in 10 languages. Those are concrete operational advantages beyond its plain REST boundary. They do not erase fit boundaries: choose another provider when webhook-driven delivery events are mandatory, SMTP relay is a hard requirement, or voice, WhatsApp, or RCS must join the fallback tree. Scheduled email also cannot be canceled here, unlike SMS cancellation, so don't schedule a login secret; generate it only in response to a live challenge.

For China-specific evidence, the pending domestic email vendor cannot support a compliance claim. US and EU teams should apply the same discipline: record what was actually evaluated, not what a product category appears to imply.

Proof beats category labels.

## Exercise the evidence and rollback controls before rollout

Start in shadow mode: create no user-visible fallback, but estimate how many primary-channel failures would have entered it and whether the datastore, mail acceptance path, and event polling budget could absorb the correlated peak. No measured latency or uptime is assumed here. Set targets only after observing the real system, and distinguish provider acceptance from inbox arrival and successful authentication.

Then enable a small policy-defined cohort. Record challenge ID, account ID, purpose, created and expiry times, attempt count, consumed state, selected channel, provider request ID when available, and the policy version that authorized fallback. Do not record the code, link token, HMAC pepper, or full message body. Retention should follow the applicable evidence policy; there is no defensible universal US/EU duration.

Poll email events for delivery evidence because push events are unavailable. Your mileage may vary on polling cadence: it has to balance detection SLOs against request volume, and the right number depends on measured delivery behavior and provider limits. Treat delayed evidence as unknown, not as proof of failure, and never issue a second live secret merely because an event has not appeared yet.

Rollback is blunt by design. Disable the email branch at the policy gate, preserve audit records under the retention rule, invalidate outstanding challenges, and route high-risk users to another enrolled factor or a reviewed recovery process. Do not try to cancel scheduled fallback mail, because scheduled email sends have no cancel operation in this capability. This is why login secrets should be sent on demand.

Before widening the cohort, test expiry at the exact boundary, six failed guesses against a five-attempt limit, concurrent double submission, duplicate delivery retries, a `429` with both forms of `Retry-After`, datastore failover, event-poll lag, and policy rollback. The pass condition is not “an email arrived.” It is that exactly one authorized transition can complete and that the evidence explains every rejection without retaining the secret.

## References

- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/sendgrid
- https://postmarkapp.com/developer
- https://mustache.github.io/mustache.5.html
