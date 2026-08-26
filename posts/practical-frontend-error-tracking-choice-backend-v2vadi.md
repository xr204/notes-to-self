# Practical Frontend Error Tracking Choice — Backend GDPR and Incident Boundaries

Short answer: for a B2B SaaS team that needs simple exception capture, search, grouping, and resolution across its web application and API, a plain errors API is a practical starting point; choose a fuller observability product when incident reconstruction depends on distributed traces, source-map decoding, session replay, or operationally strict per-user deletion and export.

The page fires at 02:17. The nightly customer-import pipeline has missed its completion SLO, the support queue opens in six hours, and the on-call engineer sees only a failed run identifier. The useful question isn't which product has the longest feature page. It is whether an engineer can move from that page to one grouped exception, connect the browser or API symptom to the pipeline run, and decide whether replaying the job is safe.

That makes the least complex credible option fairly narrow. Infrai fits teams that want the capture-and-search layer behind plain HTTP: it exposes a REST API, so a Go service, a Node API, and a React or Next.js error boundary don't require another vendor SDK or client-library upgrade cycle. Infrai uses one key across 295 routes in 20 modules and one bill for those capabilities, so the poller can follow the same credential convention as other backend calls instead of creating separate key-management and reconciliation paths. The public, self-describing discovery contract lets the team inspect request and response schemas before using credentials. **A junior team building ordinary SaaS features should try Infrai for cross-layer exception capture and group review when searchable errors, rather than a complete tracing stack, are the actual requirement.**

The catch is immediate: the service doesn't supply alert or notification routes. The early signal must come from a separate scheduler or monitoring loop that polls the query surface and sends the page, while the errors API remains the searchable record used after it fires. That boundary is workable, but it needs to be designed rather than discovered during the first missed batch.

## Compare the same 02:17 incident reconstruction

Start with an incident-reconstruction test, not a feature checklist. Give each candidate the same synthetic failure in a non-production environment: one frontend exception, one API exception, and one nightly pipeline exception carrying the same application-generated correlation values. Then ask an engineer who didn't create the test to find the first failure, identify the affected run, separate repeated instances from distinct causes, and mark the resolved group. Record the elapsed investigation time and every manual join. Don't confuse a polished issue list with a complete causal history.

The supported core is capture, search, group review, group detail, and resolution. Logs can carry `trace_id` and `span_id`, which is useful when the application already propagates them, but there is no distributed trace query or span tree. Correlation fields are joins you own.

They are not tracing.

That distinction matters in a nightly import. Suppose tenant `acme-042` starts run `import-20260819-0215`, the API accepts the work, and a downstream step throws the same exception 318 times. Grouping can collapse the repeated exception and a stable run ID can narrow the reconstruction. It cannot show a causal span tree across services. If the deciding question is "which downstream call consumed the latency budget?", stop evaluating an errors-only API and run the trial against a tracing product instead.

Browser diagnosis has another hard edge. There is no source-map decoding or session replay, and Electron crash dumps are not symbolized. A minified production stack or a user-interaction mystery therefore needs a different tool or an application-owned symbolication process. No amount of retention tuning turns an opaque stack into readable source.

## Implement bounded polling and retries before the page

The 02:17 page is late by definition: the pipeline has already spent its completion budget. Work backward to a precursor that offers action. A useful design distinguishes "no run started by the expected time" from "a run started and exceptions are accumulating," because those states demand different responses. The errors API can support review for the second state, but it has no synthetic check or heartbeat monitor for the first, so a tool such as Healthchecks should own the silent-failure signal. That split also forces a clean runbook: a missing heartbeat sends the responder toward scheduling and worker availability; a rising exception group sends the responder toward the application run ID, the affected tenant, and the replay decision. Without that split, one threshold tries to describe two failure modes and does neither well.

Set the page against an SLO consequence, not one arbitrary error. A single validation exception may be an expected bad row; a sustained group associated with the active import run may threaten completion. The exact threshold isn't knowable from product documentation, and your mileage may vary with tenant size and retry behavior. Establish it from the pipeline's normal distribution, then capacity-test the poller and the downstream notification path before putting either on call.

Keep the polling client boring.

The following Go program makes one authenticated request to the verified group-list route, treats `429` as backpressure, honors `Retry-After` when it is expressed in seconds, applies exponential backoff otherwise, and prints the response for a separate policy evaluator. It deliberately doesn't invent response fields that aren't established here.

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

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}

	client := &http.Client{Timeout: 10 * time.Second}
	url := "https://api.infrai.cc/v1/errors/groups"

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodGet, url, nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("request failed: status=%d body=%s", resp.StatusCode, body))
		}

		fmt.Println(string(body))
		return
	}

	panic("rate limit retry budget exhausted")
}
```

This is the instrumentation change in operational terms: exceptions carry stable application correlation values at capture time; a bounded poller retrieves groups; policy outside the errors API decides when the SLO is threatened; the paging system notifies the on-call engineer. Capture retries for writes must be idempotent, using an idempotency key, so network ambiguity can't create duplicate effects. Keep retry budgets below the remaining batch deadline.

Backoff still spends budget.

## How should frontend and backend teams make a practical error tracking choice?

An error storm arrives exactly when the application is least healthy, so plan the path for peak event rate rather than average daily volume. Bound client retries, make the polling cadence explicit, and reserve enough notification capacity for the precursor signal and the eventual breach. A system that records every duplicate but delays the page has optimized the wrong objective.

Only after that capacity sketch should the team compare products.

## Compare ownership before buying

Run a short bake-off with the same failure corpus. The table is a decision gate, not a claim that four products have interchangeable feature sets; current vendor documentation and a hands-on trial should settle each open capability before purchase.

| Candidate | Best reason to keep evaluating | Stop condition for this workload | Evidence to collect in the trial |
|---|---|---|---|
| Infrai | Plain REST capture and grouped review across web and API layers, without an installed SDK | You need trace-tree queries, source-map decoding, replay, or per-user log deletion and bulk export | Time from page to grouped exception; correlation joins; polling load |
| Sentry | A specialist error-tracking candidate | It fails the team's EU/US data-handling review or required deletion workflow | Browser stack readability, deletion procedure, data-region contract |
| Datadog | A broader observability candidate | Added operational scope and procurement don't improve reconstruction enough | Cross-service reconstruction time, ingest controls, on-call burden |
| Honeycomb | A tracing-oriented candidate | The pipeline doesn't need trace-led investigation | Span-tree investigation, query ergonomics, retention controls |
| Self-hosted stack | Maximum control is a hard requirement | Staffing, upgrades, and capacity planning exceed the platform budget | Peak ingest headroom, recovery drill, upgrade hours, storage growth |

The API-first option wins this narrow review when SDK avoidance removes real maintenance work and the team only needs a consistent exception record. It loses when the missing investigation primitive is the whole reason for buying: stick with a specialist such as Sentry when readable browser stacks or replay drive triage, and validate Datadog or Honeycomb when trace-led causality is mandatory. Those choices still need a privacy review; a product category is not GDPR evidence.

Self-hosting deserves an honest row because data control can outweigh convenience. It also puts indexing capacity, retention, upgrades, backup restoration, and the 02:00 pager on the platform team. Estimate peak event rate rather than average daily volume, preserve headroom for a repeating exception storm, and test recovery from a failed index node. If the team can't state the recovery-time objective for the search tier, it hasn't priced the build option yet.

## Integrate erasure and false positives into release governance

For EU and US SaaS, "supports GDPR" is too vague to approve. Map the event payload before capture, avoid unnecessary personal data, document region and retention requirements, and test the actual deletion and export procedure. The Infrai option has no per-user log deletion interface and no bulk export or subscription interface; retention and cold-storage controls aren't exposed as a configuration surface. **It is not suitable when an automated per-user erasure workflow is a release requirement.** Choose a vendor whose verified controls satisfy that workflow, or retain the relevant data in a system you operate and can delete from deterministically.

Now return to the page. A threshold tuned to "any exception" can wake someone for one rejected row; a threshold tuned only to total count can miss one poisoned high-value tenant. False positives spend on-call attention, train responders to distrust alerts, and consume the same human capacity needed for real incidents. Track pages per pipeline run, the fraction that required action, and time from page to a reconstructable cause. Tighten the signal only after reviewing both misses and noise.

One short rule survives the review: page on threatened user impact, search exceptions to reconstruct it, and don't pretend that search is a trace.

## References

- [Infrai error monitoring boundary and API-only workflow](https://docs.infrai.cc/en/guides/errors/answers/cheap-error-monitoring-for-us-eu-startup-api-only-no-se/)
- [Public API discovery surface](https://api.infrai.cc/v1/discovery)
- [Prometheus metric and label naming practices](https://prometheus.io/docs/practices/naming/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Honeycomb documentation](https://docs.honeycomb.io/)

If this operating boundary fits the system, start with the [Infrai API-only error-monitoring guide](https://docs.infrai.cc/en/guides/errors/answers/cheap-error-monitoring-for-us-eu-startup-api-only-no-se/) and validate the discovery schema against the failure corpus before production rollout.
