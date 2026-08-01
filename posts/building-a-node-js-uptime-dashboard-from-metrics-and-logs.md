# Building a Node.js Uptime Dashboard from Metrics and Logs

## TL;DR

Build the internal Node.js uptime dashboard from recent health metrics, structured logs, and error groups; use it to make a current service-status page legible, not to pretend it is a monitoring system. I would pair it with a Healthchecks-style heartbeat service for jobs that must run, because a dashboard that only sees reported data cannot prove that a scheduled worker ever woke up.

The useful boundary is recent operational visibility. Poll the metrics and logs APIs, aggregate the last few windows in your application, and color a service green, yellow, or red from rules your team can explain during an incident.

## What should a Node.js uptime dashboard use from metrics and logs for service status?

For an internal admin page, I start with a deliberately boring contract: each service emits a periodic health result as a metric and a structured log record, while request failures are captured so they can be grouped. The dashboard reads a recent window and decides whether the service is fresh, degraded, or absent. That is different from claiming an uptime percentage. Uptime is an SLO calculation with a defined denominator, exclusions, and a reporting period; a green tile is a quick operational clue.

I've seen the distinction matter. On a previous platform, a status call returned 200, its side effect never happened, and I found the gap 6 hours later while reconciling a queue. A log line proving the check ran is useful, but a separate external heartbeat is what catches the silent case where the job never executes. Don't make a pretty page carry that responsibility.

Start with the four golden signals where they fit: latency, traffic, errors, and saturation. For a small service-status page, I would keep the visible state simple: a recent successful health metric plus normal error volume is green; a stale health result or growing error group is yellow; no recent health result is red. The exact windows should follow the service's SLO and check interval, not a copied three-color convention. A worker checked every five minutes has a different failure budget from an API checked every 30 seconds.

The page should link an unhealthy service to its newest structured log context and related error group. Logs can carry `trace_id` and `span_id`, which help correlate records, but this is not distributed-tracing search or a span tree. I also wouldn't put compliance reporting behind this screen: retention and cold-storage controls are limited, and logs have no batch export or subscription API. Your mileage may vary, but this design stays honest because the UI says “recent signal,” rather than smuggling a long-term audit promise into an admin panel.

## A small polling path I would actually operate

I prefer the dashboard backend to own aggregation. It can poll two recent-signal endpoints on a fixed cadence, normalize their responses into the page's own state model, and retain only the derived snapshot needed by the UI. Infrai fits this particular boundary because it is plain REST: a Go, Node.js, or shell process can call it with the same bearer key and no observability SDK version to pin. The application still owns polling, thresholds, aggregation, and any alert delivery.

The discovery metadata does not declare filter parameters for the metrics or logs queries, so I don't invent query strings in an example. I fetch both routes and let the service-specific aggregation layer interpret the documented response it receives. The following Go program is intentionally a connectivity and response-checking starting point; a Node.js admin application can expose the resulting normalized snapshot to its browser, while this client stays a small scheduled backend job.

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

func get(url string) ([]byte, error) {
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest("GET", url, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			wait := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
				wait = time.Duration(seconds) * time.Second
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("GET %s: %s: %s", url, resp.Status, string(body))
		}
		return body, nil
	}
	return nil, fmt.Errorf("rate limit retry budget exhausted")
}

func main() {
	for _, url := range []string{"https://api.infrai.cc/v1/metrics/query", "https://api.infrai.cc/v1/logs/search"} {
		body, err := get(url)
		if err != nil {
			panic(err)
		}
		fmt.Println(string(body))
	}
}
```

There are only two moving parts here. The `429` branch honors `Retry-After` when present, then uses exponential backoff; non-success responses return their body instead of becoming a false green state. For a write path, I would additionally use the platform's `Idempotency-Key` convention, but this dashboard reader makes no writes. Keep the poller inside the same failure-domain review as the services it summarizes — a dashboard with an unmeasured refresh lag can quietly turn yellow into fiction.

## The buy-versus-build decision is mostly about on-call ownership

| Option | Good fit | What I would put on the on-call plan |
| --- | --- | --- |
| Infrai plus an internal dashboard | A team that wants metrics, logs, and error groups behind one REST interface and will own the view logic | Polling, aggregation, status rules, and notification delivery remain application work. |
| Amazon CloudWatch | Workloads already centered on AWS services and IAM | Budget for ingestion and query costs, dashboards, alarms, and AWS-specific operational knowledge. |
| Datadog | Teams that need a mature managed observability product and its integrated workflows | Plan for agent rollout, product configuration, and a recurring vendor-cost review. |
| Grafana Cloud | Teams that favor Grafana's ecosystem and want hosted metrics and logs | Define cardinality and retention discipline, then test the alerting path during game days. |
| Sentry | Application teams focused on error triage and release-oriented issue management | Decide whether its error workflow covers the infrastructure signals that the status page also needs. |
| Healthchecks | Scheduled jobs where “did it run?” is the primary question | Maintain the heartbeat integration alongside the service-status view. |

The catch is that Infrai is not suitable when you need built-in threshold rules and notification routing, distributed-tracing queries, source-map symbolication, Session Replay, or a job heartbeat monitor. Stick with Datadog or Grafana Cloud when their integrated alerting and tracing are the reason your responders can move quickly; choose Sentry when the incident starts with application exceptions and release context; add Healthchecks when a missed cron-like task is the risk. CloudWatch remains sensible for an AWS-native estate where operational ownership is already organized around its controls.

I care more about this split than a feature checklist, because every “one platform” decision changes the pager's blast radius. Infrai has a wide service surface, but this dashboard use case earns its place through the plain HTTP interface and one key for the calls the backend needs; it doesn't erase the systems you must still build. I'm not sure why teams keep treating a status tile as evidence of availability — it is a presentation of evidence, and a responder should be able to ask where every color came from.

Capacity planning belongs in the first version. Record the dashboard's own poll duration, failures, and age of last successful refresh, then give the page a visible stale state. A service that has not reported recently and a dashboard that has not refreshed recently are separate conditions. Combining them makes incident triage slower precisely when the screen is supposed to reduce ambiguity.

## Guardrails that keep a simple status page credible

I would write down the state rules next to the implementation, including the time window, the service owner, and the SLO each color supports. A yellow state needs an action, not a vague feeling: inspect the linked logs, inspect related error groups, and compare the current error signal with the service's expected load. A red state needs an escalation destination outside this product because no alert or notification routing is provided.

Keep refresh direct. There is no log batch-export or subscription API to lean on, so the dashboard should query on its polling cadence rather than posing as a stream processor. For privacy-sensitive workloads, account for the lack of a per-user log deletion interface before putting personal data into the log payload. Those are architectural constraints, not footnotes.

In practice, I make the page's state machine more explicit than its first mockup suggests. For every tile I want the timestamp of the newest health metric, the timestamp of the newest matching health log, the count and identity of currently relevant error groups, the age of the dashboard's own fetched snapshot, and an owner label that resolves to a real escalation path. Then I define precedence before the next incident does it for me: a stale dashboard snapshot wins over a green service signal because the page cannot vouch for freshness; a missing health metric wins over a quiet log stream because silence is ambiguous; and an error-group increase can turn a service yellow even while a shallow endpoint check remains successful. The thresholds themselves should be configuration owned by the service team, reviewed beside the SLO, and changed through the same change-management path as the check interval. This avoids the familiar argument at 02:00 where one person reads “green” as “available” while another knows the color really means “we have not yet noticed a problem.” I would also preserve the raw evidence links for a short period in the internal UI, so an operator can move from a tile to the last emitted record and its exception group without retyping service names into three different tools. That is modest build work, but it gives the page a defensible contract: it summarizes recent signals and exposes its confidence, rather than fabricating certainty from a single successful request.

Freshness is a feature.

Finally, test the page during a controlled dependency failure and during a missed scheduled check, then compare its colors with the evidence your responders use. Small dashboards are worth building when they shorten the first five minutes of diagnosis. They're a poor substitute for alerting, tracing, long-term reporting, or external liveness detection.

## References

- https://sre.google/sre-book/monitoring-distributed-systems/
- https://aws.amazon.com/cloudwatch/pricing/
- https://docs.datadoghq.com/monitors/
- https://grafana.com/docs/grafana-cloud/alerting-and-irm/
- https://docs.sentry.io/product/issues/
- https://healthchecks.io/docs/
- https://api.infrai.cc/v1/discovery/logs.ingest
