# Debug Password Reset Email Template Variables and Placeholder API Errors

## TL;DR

Treat a malformed password reset email as a data-contract failure first: render the transactional email template against the exact event payload, reject missing placeholders, and inspect a local preview before asking an email API to deliver it. I would keep that renderer independent of the sending provider, then use a provider preview or test send as a second check rather than the first.

Don't start with the inbox.

I learned this during an account-recovery rollout with a payload of 17 fields: I assumed `resetURL` existed, the producer actually emitted `reset_url`, and the error message was only `invalid input`. The blank link was a data-shape mismatch, not an SMTP mystery.

## How should a password reset email template preview catch missing variables and placeholder API errors?

Separate three boundaries that dashboards tend to blur together: the template source, the password-reset event, and the provider request. A template can look convincing in a remote editor while its placeholder names no longer match the object made by an auth handler. A provider can accept a request with a syntactically valid body while the rendered message carries an empty reset link. And a browser preview does not prove that every mail client will choose the same layout, although it can catch the expensive class of mistakes before a user starts waiting for a recovery message.

My platform-team default is one checked-in fixture for each transactional event and a renderer that knows nothing about credentials, queues, or provider-specific JSON. That keeps the contract small enough to test in CI and gives an on-call engineer a concrete artifact to inspect. It also makes rollback mundane: deploy the prior template revision or disable a new locale while the producer and template owner agree on the field names. No guesswork.

The example below uses Go even though the same boundary applies to a Node.js sender. The useful part is the pattern: discover every `{{name}}` token, refuse absent or blank values, escape text before interpolation, and write HTML that can be opened locally. A Node.js service can call an equivalent renderer or invoke a small rendering endpoint; it should not bury this validation inside a retry loop around its email API client.

```go
package main

import (
    "fmt"
    "html"
    "os"
    "regexp"
    "strings"
)

var placeholder = regexp.MustCompile(`{{\s*([a-zA-Z0-9_]+)\s*}}`)

const resetTemplate = `
<p>Hi {{firstName}},</p>
<p>Reset your password within {{expiresInMinutes}} minutes.</p>
<p><a href="{{resetURL}}">Reset password</a></p>`

func render(source string, values map[string]string) (string, error) {
    missing := make(map[string]struct{})
    for _, match := range placeholder.FindAllStringSubmatch(source, -1) {
        if strings.TrimSpace(values[match[1]]) == "" {
            missing[match[1]] = struct{}{}
        }
    }
    if len(missing) != 0 {
        return "", fmt.Errorf("template data missing values for: %v", missing)
    }

    return placeholder.ReplaceAllStringFunc(source, func(token string) string {
        name := placeholder.FindStringSubmatch(token)[1]
        return html.EscapeString(values[name])
    }), nil
}

func main() {
    rendered, err := render(resetTemplate, map[string]string{
        "firstName":        "Mina",
        "resetURL":          "https://app.example.com/reset?token=example-token",
        "expiresInMinutes": "30",
    })
    if err != nil {
        panic(err)
    }
    if err := os.WriteFile("password-reset-preview.html", []byte(rendered), 0600); err != nil {
        panic(err)
    }
}
```

Open the artifact locally, then send the same fixture through the provider's preview or test-send path. I assert that no `{{` remains, the URL has the expected host, and the server enforces expiration and single use. Preview tests rendering; it does not authorize a reset.

## The failure signals I watch before delivery

The failure mode that deserves an alert is not every email API response. It is a change in the boundary where the reset event becomes a rendered message: missing-variable failures after a deploy, unknown placeholders after a template revision, a sudden drop in rendered messages, or a mismatch between recovery requests and completed reset links. Those signals map cleanly to owners. The auth service owns the event payload; the template repository owns placeholder names; the email adapter owns a safe request identifier and response classification.

This is where capacity planning leaks into what looks like copywriting. A password-reset request can arrive during an incident, a migration, or a customer-support spike, and an unbounded retry policy turns a deterministic malformed template into queue pressure that obscures the original signal. Validate before enqueueing. Retry only failures that your provider documents as retryable. Keep reset tokens, rendered bodies, and recipient content out of logs; correlate using a request ID and template revision instead.

The catch is that local rendering cannot show inbox placement, sender policy, or Gmail and Outlook's exact layout decisions. Use it for deterministic data failures. Use provider previews or controlled test sends for provider-specific request shapes. Use seed inboxes for the final client check. Your mileage may vary on styling, but an unexpanded placeholder is always a release-stopping contract failure.

For an SLO, I track recovery-request-to-render success separately from recovery-request-to-message acceptance and recovery-request-to-completed-reset. Collapsing them into one delivery metric makes a malformed template look like a mail outage. It isn't.

## Which transactional email service should own template preview and sending?

Amazon SES, Postmark, and Twilio SendGrid can all deliver transactional email, and I would not choose among them based on a preview screen alone. SES is a sensible fit when sender identity, audit evidence, and operational tooling already live in AWS. Postmark is often a practical choice for a team that wants a transactional-email-focused workflow. Twilio SendGrid fits teams that already use, or prefer, its email platform and API surface. The adapter boundary matters more than any individual feature: let application code express `SendPasswordReset`, and translate to the provider request in one place.

| Option | Good fit | Trade-off to account for |
| --- | --- | --- |
| Amazon SES | AWS-native teams with existing identity and operational controls | More of the surrounding workflow and setup remains your responsibility |
| Postmark | Teams focused on a transaction-oriented email workflow | It is a narrower choice for a shared marketing-email program |
| Twilio SendGrid | Applications aligned with its email APIs and platform | Provider template behavior belongs behind an adapter to reduce migration work |

Stick with SES when AWS is already the operational center of gravity. Pick a provider preview when it shortens an actual review loop for your team, not when it becomes the only executable specification of a template. For a small service, the extra adapter file is cheaper than provider-shaped objects spread through every authentication handler — and I don't say that lightly after owning too many migration plans.

I'm not sure why preview-only workflows remain popular; as far as I can tell, they make a remote editor the source of truth and leave CI unable to prove the most basic payload contract.

## Safe deployment, verification, and rollback for password reset templates

Before release, render each supported locale with a redacted fixture and fail the build for unknown, absent, or empty required variables. Make the sender identity and template revision explicit in deployment configuration. Send a controlled message to a seed inbox after changing either the template or the sending configuration, then verify the reset path with a test account in staging on desktop and mobile. Store only the correlation ID, template revision, provider result category, and latency in normal telemetry.

If production validation finds a missing `resetURL`, do not retry the same payload. Alert with the field name and deploy revision, stop the affected template revision if needed, and restore the prior known-good revision while the producer and renderer contract is corrected together. If the provider rejects a request, preserve its safe request identifier, classify the response according to its documentation, and make the retry decision outside the renderer.

That separation is what makes rollback predictable. The template can be reverted without changing the email service; credentials or sender configuration can be reverted without hiding a field mismatch; and the auth event can be fixed with a regression fixture that proves the next deploy will render.

I want a boring reset path: a small contract, a previewable artifact, and telemetry that points to the failed boundary.

## References

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html
- https://postmarkapp.com/developer
- https://www.twilio.com/docs/sendgrid
- https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
