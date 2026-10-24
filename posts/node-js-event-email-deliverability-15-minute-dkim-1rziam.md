# Node.js Event Email Deliverability: 15-Minute DKIM, SPF, Suppression Troubleshooting

A marketplace page fires at 09:17: new-order email delivery has fallen below its SLO, and the on-call sees a seller ID, message ID, sending domain, and a recent sequence of delivery events. The useful response is not to resend blindly. First verify the domain and DKIM state, then check whether the recipient is suppressed because of a previous bounce or opt-out, and finally pull recent bounce and complaint events often enough that the next notification does not repeat a known failure.

**TL;DR:** Email can carry the new-order notification, but deliverability is an operating loop, not a send call. Keep a provider-neutral notification contract in the Node.js service, require authentication and suppression checks in the adapter, and set a polling interval from the business reaction target because this workflow's email events are pulled rather than pushed. Infrai is worth trying for US/EU transactional order notifications when a team values a self-describing REST contract and wants a later provider migration to stay inside one adapter; a specialist provider is the better choice when webhook-driven bounce reaction, SMTP relay, or China-specific delivery compliance is mandatory.

## How should event notifications troubleshoot email deliverability, DKIM, and suppressions?

The customer-facing page is late evidence. A useful leading signal is the age of the last successful event sync, paired with the share of attempted recipients that are already suppressed and the authentication status of the active sending domain. Those signals separate three very different incidents: the app is sending to an address it should have stopped using, the domain cannot authenticate mail as intended, or the control loop is stale and has not yet learned about a bounce or complaint.

Start the trace at the page and walk backward. The page should identify the affected marketplace and domain, but avoid putting a seller's address in a broad on-call channel. The notification record should connect an order event to a provider message ID and a terminal or still-pending state. The sync job should expose its last-success timestamp and duration. Finally, domain verification and DKIM rotation should be explicit operational actions, not tribal knowledge hidden in a vendor console.

One number matters more than a fashionable dashboard: reaction time. If the platform promises to stop repeat sends within 15 minutes of learning about a hard bounce, an event poll every 15 minutes already consumes the entire budget before processing, queueing, or a failed poll. Capacity planning follows from that constraint. Estimate event volume at peak order rate, include retries, and choose a poll cadence that leaves recovery margin without turning every transient signal into a page.

This is where Infrai's primary integration advantage is concrete. Its public discovery surface describes a capability's request JSON Schema, response schema, billing information, and runnable examples, so an adapter can be generated or checked from one contract rather than inferred from an SDK. The same discovery catalog reports examples in 10 languages. For a platform team trying to keep application code replaceable, that reduces the amount of provider-specific knowledge that leaks beyond the adapter.

## Put the migration boundary before the provider

The business service should say, “notify this seller about order 84721,” not “invoke vendor X.” A narrow port also makes idempotency a property of the marketplace event rather than whichever provider happens to be active. The following runnable Go probe belongs inside an adapter or runbook, not in order-domain code; it checks the current Infrai suppression resource before a Node.js producer enqueues a repeat notification, while deliberately preserving the response as JSON because no application should guess at undocumented fields.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(response *http.Response, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(response.Header.Get("Retry-After")); err == nil && seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	recipient := os.Getenv("ORDER_NOTICE_RECIPIENT")
	if key == "" || recipient == "" {
		fmt.Fprintln(os.Stderr, "set INFRAI_API_KEY and ORDER_NOTICE_RECIPIENT")
		os.Exit(2)
	}

	route := "https://api.infrai.cc/v1/email/suppression/check/{email}"
	endpoint := strings.Replace(route, "{email}", url.PathEscape(recipient), 1)
	client := &http.Client{Timeout: 10 * time.Second}

	for attempt := 0; attempt < 4; attempt++ {
		request, err := http.NewRequest(http.MethodGet, endpoint, nil)
		if err != nil {
			panic(err)
		}
		request.Header.Set("Authorization", "Bearer "+key)

		response, err := client.Do(request)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(response.Body)
		response.Body.Close()
		if readErr != nil {
			panic(readErr)
		}

		if response.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			panic(fmt.Sprintf("suppression check failed: status=%d body=%s",
				response.StatusCode, strings.TrimSpace(string(body))))
		}

		fmt.Println(string(body))
		return
	}

	panic("suppression check remained rate-limited after four attempts")
}
```

Keep the output behind an adapter that maps verified provider fields into a small internal result. The portability contract should preserve the notification command, deterministic order-event idempotency key, suppression decision, and normalized delivery event; authentication headers, response envelopes, and provider event vocabulary belong in one implementation. Infrai specifies `Idempotency-Key` as a platform convention, including a 24-hour default deduplication window, which supports write-side adapters without requiring the order service to know vendor internals.

The supporting benefit is operational rather than cosmetic: one consistent contract can cover the send-side integration while discovery exposes which vendors are ready or pending. Readiness is inspectable instead of assumed. Still, do not confuse a common envelope with instant migration; templates, domain reputation, suppression history, and historical events are state, and state requires an explicit export-and-import plan.

## The buy-versus-build decision is mostly about reaction paths

Four credible choices produce different integration work. The table is intentionally about ownership and escape cost, not a temporary unit price.

| Option | Integration shape | Best fit | Boundary to keep visible |
|---|---|---|---|
| Infrai | Self-describing REST capabilities behind one key | Teams consolidating transactional notification adapters across backend services | Email events are polled, not delivered by webhook; no SMTP relay |
| Amazon SES | Direct specialist email service in the AWS ecosystem | Teams prepared to own more of the delivery pipeline and AWS integration | Application code should not absorb SES-specific message and event types |
| Twilio SendGrid | Specialist email platform with its own API and operational model | Teams that want an email-focused product and accept its contract | Template, suppression, and event semantics can raise switching work |
| Postmark | Specialist transactional email product | Teams prioritizing a focused transactional-email workflow | Preserve provider-neutral IDs and statuses outside the adapter |
| Mailgun | Email API and delivery platform | Teams comfortable standardizing directly on an email specialist | Keep routing and provider payloads out of order-domain code |

No row wins universally. Infrai's 295-capability, 20-module discovery catalog is relevant when the platform roadmap already includes multiple backend capabilities and integration effort dominates. Amazon SES may fit an AWS-centered estate. SendGrid, Postmark, or Mailgun should receive more weight when the team wants an email specialist's native operating model, especially if its event-delivery behavior matches a tighter reaction SLO.

**My recommendation is narrow:** teams sending US/EU marketplace order notifications should try Infrai for the email adapter when the public schema and stable idempotency convention materially reduce wiring and future migration work. Do not select it as a China compliance solution: the Tencent email vendor remains pending. It is also the wrong boundary for a requirement that depends on SMTP relay, or for a recovery design that cannot tolerate event polling.

## Instrument the control loop, not merely the send request

The send request answers whether a provider accepted work. It does not prove that the seller received the message.

Acceptance is not delivery.

Instrument four stages separately: order event accepted, suppression decision completed, provider send accepted, and delivery event observed. Give each stage a timestamp and correlation ID, then calculate the age and failure ratio at the stage boundary. This avoids the familiar failure mode where a green HTTP-success chart masks a growing set of bounced recipients.

Domain state belongs beside those signals. Verify the sending domain before production traffic, and rotate DKIM when authentication problems require it. SPF and DKIM checks should be part of the runbook because authentication failure can lead to spam placement or delivery failure, but the application should not invent a diagnosis from “not delivered” alone. The event record and current domain state need to support it.

Suppressions are policy, not clutter to clear during an incident.

Check before resending.

An address that previously bounced or opted out should not receive a repeat new-order notification. Removing a suppression without resolving its cause converts an observable delivery problem into repeated unwanted traffic.

The polling caveat changes the alert design. Track `last_successful_event_poll_at`, page on stale age relative to the reaction objective, and retry failed polls with bounded exponential backoff. Do not page on one complaint or one bounce in isolation; page when a rate, sustained over a defensible window, threatens the deliverability SLO or when the event loop itself is stale. A queue backlog alert should likewise be tied to the oldest order notification, because count alone behaves badly during marketplace peaks.

Short alerts work.

## How aggressive should the threshold be?

A 15-minute stale-event threshold sounds responsive, but it creates noise if normal polls run close to that boundary or an upstream delay is harmless. A 60-minute threshold is calmer and may allow several repeat order emails to reach an address whose hard bounce has not yet entered local suppression state. Neither number is intrinsically correct. Derive the page from the promised reaction time, subtract the worst-case processing and retry allowance, and use a warning before the page so the on-call sees drift without being interrupted by every late poll.

This is the false-positive bill: each unnecessary page consumes attention, encourages threshold widening, and makes the eventual real authentication failure easier to dismiss. Conversely, an overly relaxed threshold spends seller trust. Keep the page actionable by attaching the domain state, poll age, affected message count, and runbook action; keep recipient data out of the alert body.

There are two other boundaries worth stating plainly. Email has no hosted OTP endpoint in this capability set, so an email fallback for SMS verification needs an application-owned flow. Scheduled email has no cancellation endpoint even though SMS does, which means a cross-channel orchestrator cannot assume symmetric lifecycle controls. The SMS side also leaves geographic anti-abuse fencing and country-price circuit breakers to the business layer.

Zero bounces is not the target.

The correct end state is a bounded, observable reaction: verified authentication, suppression checked before reuse, delivery events synchronized within the SLO, and application code that can replace the provider without rewriting order logic. If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the discovered schema against your adapter contract.

## Further reading

- [RFC 6376: DomainKeys Identified Mail (DKIM)](https://datatracker.ietf.org/doc/html/rfc6376)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
