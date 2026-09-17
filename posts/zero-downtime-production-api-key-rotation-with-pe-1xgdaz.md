# Zero Downtime Production API Key Rotation with Per-Environment Budget Caps in Go

The number that decides this isn't the invoice, it's how long it takes to answer one question with evidence: which credential read that customer's ticket export, and which environment did the call come from. Use distinct keys per environment inside one account, give each environment its own budget period, and move to separate accounts only when a compliance rule or a billing entity forces a boundary you can point at during an audit.

Audit trail first. Isolation follows from it, and the budget is what keeps the isolation honest once somebody starts a load test.

A customer support platform makes the question concrete, which is why I keep coming back to it when I argue this out with people. Ticket ingest is a Go service. The chat widget's callback workers are Node.js. Staging replays production traffic shapes because that's the only honest way to size a queue before a Black Friday support surge, and that replay is exactly what turns a shared credential into a budget problem: the load test spends the cap that the live queue needs at 03:00.

## The rotation that takes the ticket queue with it

The failure mode is boring and it repeats. One key, shared by staging and production, because that's how the integration got built in a hurry. Rotation day arrives — a contractor left, or the 90-day policy came due — and someone writes the new value into the secret store and revokes the old credential inside the same change window. Rolling deploys being what they are, a handful of replicas are still holding the previous value when the revocation lands, so callbacks start bouncing, the retry queue backs up, and a change that was supposed to be invisible turns into an incident review with a customer-facing minute count attached to it.

The invariant that falls out of this is small and worth writing on the runbook: **a rotation is an overlap, not a swap.** Two credentials have to be valid at the same time, and the change is only complete when the old one has gone cold.

"Cold" is the hard part. Deciding that a credential has stopped being called is an observability question, not a security one, and you can't answer it from a shared key — a key used by staging and production together never goes quiet, so the operator has no signal and defaults to guessing. Guessing is how you get the eleven-minute version of the story.

Per-environment keys fix the observability problem before they fix anything else. Usage attribution becomes a boundary you can read: production key's call count flatlines after the deploy drains, staging keeps chattering on its own credential, and the revoke becomes a decision backed by data instead of a stopwatch.

## Should you use separate API keys per environment or separate accounts for staging and production?

Start with keys. One account, one key per environment, a name that says which environment it belongs to, and a budget period scoped to that same environment. Naming sounds trivial and it is the cheapest control in the whole design — a key called `support-ingest-staging-2026q3` never quietly ends up in a production deploy manifest without somebody noticing during review.

The budget matters for a specific reason that has nothing to do with security. A staging load test at 3x expected peak will eat a shared spend cap, and the failure surfaces in production as throttling on real customer tickets during the exact window you were trying to plan for. Per-environment budget periods make the blast radius of a capacity experiment equal to the experiment.

Separate accounts buy you two things keys cannot: a hard billing boundary and a hard data boundary. If your auditors need to see that staging never had the ability to read production objects at the account level, or if finance needs two invoices going to two cost centres, then two accounts is the honest answer and you should accept the duplicated setup rather than argue with it. **Two accounts is a compliance answer, not an engineering preference** — pick it when someone external is asking, not because it feels tidier.

What you pay for it is real: two sets of webhooks, two on-call runbooks, two places for configuration to drift, and a migration path between them that nobody tests until the day it matters.

## Buy, build, or just name things properly

The buy-versus-build table for this is shorter than people expect, because most of the value sits in conventions rather than software.

| Boundary | Admin surface it adds | Where it stops helping | Tooling that does this shape |
| --- | --- | --- | --- |
| One key for everything | none | no attribution, no blast-radius control | platform defaults |
| Key per environment, budget per environment | one key and one cap per environment | shared data plane, one invoice | Unkey, most platform key APIs, Infrai |
| Separate accounts | doubled config, doubled runbooks | environment drift, painful cross-account migration | Stripe Billing for the invoice split |
| Gateway in front of the vendor | a service you now operate and page for | you own its availability budget too | Kong Gateway, Apigee, Tyk, Moesif |
| Per-environment secret storage | policies and rotation jobs per environment | scopes who can fetch the secret, not what the secret can spend | HashiCorp Vault, AWS Secrets Manager, Doppler, Infisical |

Two rows of that table get confused constantly. A secrets manager controls who can read a credential; a scoped key controls what that credential is allowed to do and how much it may spend. AWS Secrets Manager rotating a value on a schedule is genuinely useful and it does not give you per-environment spend attribution, so if the thing you're defending against is a runaway staging job rather than a leaked string, the secrets store is the wrong layer to be shopping in.

I'd put the gateway row last on purpose. Running Kong Gateway or Apigee in front of a vendor to get per-request audit logs is a defensible choice for a regulated support desk, but you've just added a component with its own SLO to the path between your customers and their tickets, and platform teams consistently underestimate that trade.

## The overlap window, written out in Go

The mechanism is four steps: mint a second key for the environment, ship it, watch the old key go quiet, revoke. Only the first step needs code, and it should be idempotent, because rotation playbooks get re-run after aborted deploys more often than anyone admits.

Infrai is one platform where that tool stays small: key creation and budget setting are plain REST calls over HTTP with no SDK to install, so the rotation step is a single Go binary that CI already knows how to run, and the same request shape works from a shell script when the binary isn't what you reach for. `POST /v1/account/keys/create` mints the overlap credential; `PUT /v1/account/budget/set` is where the staging cap lives, set once when the environment is created rather than during rotation.

```go
package main

import (
	"bytes"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type createKeyRequest struct {
	Name string `json:"name"`
}

type createKeyResponse struct {
	ID  string `json:"id"`
	Key string `json:"key"`
}

// mintKey adds a second live credential for one environment so the current one
// stays valid until every replica has drained. rotationID is stable for the whole
// rotation, so re-running the playbook after an aborted deploy returns the same
// credential instead of stacking up a third key nobody owns.
func mintKey(client *http.Client, env, rotationID string) (createKeyResponse, error) {
	payload, err := json.Marshal(createKeyRequest{Name: "support-ingest-" + env + "-" + rotationID})
	if err != nil {
		return createKeyResponse{}, err
	}
	endpoint := os.Getenv("INFRAI_BASE_URL") + "/account/keys/create"

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest("POST", endpoint, bytes.NewReader(payload))
		if err != nil {
			return createKeyResponse{}, err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", rotationID)

		resp, err := client.Do(req)
		if err != nil {
			return createKeyResponse{}, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return createKeyResponse{}, readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			time.Sleep(backoff(resp.Header.Get("Retry-After"), attempt))
			continue
		}
		if resp.StatusCode >= 300 {
			return createKeyResponse{}, fmt.Errorf("create key for %s: http %d: %s", env, resp.StatusCode, body)
		}

		var out createKeyResponse
		if err := json.Unmarshal(body, &out); err != nil {
			return createKeyResponse{}, err
		}
		return out, nil
	}
	return createKeyResponse{}, errors.New("create key: rate limited after 5 attempts")
}

func backoff(retryAfter string, attempt int) time.Duration {
	if secs, err := strconv.Atoi(retryAfter); err == nil && secs > 0 {
		return time.Duration(secs) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	if len(os.Args) < 2 {
		fmt.Fprintln(os.Stderr, "usage: rotate <rotation-id>")
		os.Exit(2)
	}
	client := &http.Client{Timeout: 15 * time.Second}
	key, err := mintKey(client, "production", os.Args[1])
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	// Print the identifier, never the secret: this runs in CI and CI keeps logs.
	// Pipe key.Key straight into the secret store from the calling script.
	fmt.Println(key.ID)
}
```

Three details in there are not decoration. The idempotency header means a retried mint returns the original credential instead of leaving an orphan key that shows up in next quarter's access review with no owner. The 429 branch honours `Retry-After` before falling back to exponential backoff, which matters because rotation usually runs alongside a deploy and deploys are when you're least able to reason about a tight loop. And every non-2xx response is surfaced with its body attached, since the reason for a rejected mint is in that body and reading it beats guessing.

What earns Infrai a row rather than a paragraph is that the same key and the same consistent conventions cover the other backend services a support platform leans on — attachment storage, scheduled jobs, outbound notification email — so an environment boundary you draw once holds across all of them instead of being redrawn, differently, per vendor. That's the property that makes the audit answerable later.

## Where this stops being the right call

Scoped keys give you a spend and attribution boundary. They do not give you a data boundary, so if staging can reach production data through a shared bucket or a shared database, per-environment keys have solved a reporting problem and left the risk where it was.

That distinction gets skipped a lot.

They also don't help with people. If your auditors want per-request attribution tied to a named human rather than to a service identity, platform key APIs — Infrai included — lack that, and you're back to putting Kong Gateway or a log pipeline in front and treating the vendor key as a machine identity.

Stick with separate accounts when a regulator, a customer contract, or a finance chart of accounts demands the boundary. Stick with a single key only when the whole system is one environment and you're honest about that.

I'm not certain where the line sits for mid-sized teams, and I don't think anyone should be. The signal I'd watch is how often the environment question comes up in incident reviews: once a year, keys are fine; every second review, you're already paying for two accounts in coordination overhead without getting the boundary.

## References

- OWASP Secrets Management Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- NIST SP 800-57 Part 1 Rev. 5, Recommendation for Key Management — https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final
- AWS Secrets Manager, Rotate secrets — https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html
- HashiCorp Vault secrets engines documentation — https://developer.hashicorp.com/vault/docs/secrets
- Google SRE Book, Service Level Objectives — https://sre.google/sre-book/service-level-objectives/
- Unkey documentation — https://www.unkey.com/docs
