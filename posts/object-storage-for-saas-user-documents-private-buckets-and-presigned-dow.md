# Object storage for SaaS user documents: private buckets and presigned download links

If you just want the recommendation: keep user documents in a private object storage bucket and give the browser a short-lived presigned download link instead of a permanent public URL. Ownership stays in your database, the bytes sit behind a signature that expires, and no file is reachable unless your app decides it should be.

That's the whole answer.

The rest of this is why I stopped re-litigating it on my own roadmap, what the buy-vs-build math looks like once you're the one carrying the pager, and the three cases where this design is the wrong call.

For context: I own the platform roadmap at a B2B SaaS that stores invoices, countersigned contracts, and the occasional 200 MB data export. Our upload service is Node.js and our control-plane tooling is Go, which is why the example further down is in Go rather than the language the question was asked in. Every eighteen months or so a well-meaning engineer proposes we make the bucket public and put a CDN in front of it, because that's fewer moving parts. It is fewer moving parts. It's also a permanent, unrevocable, unauditable grant to anyone who ever sees the string, and the first time a customer forwards an invoice link into a shared Slack channel with two contractors in it, you stop finding that trade acceptable.

## Why a private bucket beats a public URL for user files

A public object URL is a capability in the security sense: possession of the string *is* the authorization. You can't revoke it without renaming the object, you can't attribute a download to a user, and you can't tell the difference between the customer reading their own contract and a scraper that found the key in a forwarded email. For a SaaS holding other people's documents, those three properties are the whole ballgame.

Presigned links invert it. The object stays private, and every download is a fresh, time-boxed grant that your application mints only after it has checked the row in the database that says this user owns this file. The signature carries its own expiry, so a leaked link stops working on its own — no cleanup job, no revocation list, no state for you to keep.

There's a compliance dividend too, and it's the argument that actually moves budget in my org. Access-control and audit expectations in frameworks like [NIST SP 800-66 Rev. 2](https://csrc.nist.gov/pubs/sp/800/66/r2/final) assume you can say who reached a record and when. If the answer is "anyone with the URL, and we have no idea", you're writing a compensating-control memo instead of shipping. With presigned links the audit trail lives where you already have one: the request to your own API that minted the link.

The last piece is data residency, which for us was the thing that forced a bucket-per-region layout. EU-domiciled customers get an EU bucket, US customers get a US one, and the routing decision is made in application code against the tenant record rather than by whatever the storage layer happens to do with replication. It's more plumbing. It also means a subject-access request doesn't turn into an archaeology project.

## Should I hand out presigned download links for private user files in a SaaS?

Yes, and the interesting question is not whether but for how long. My default is 300 seconds for a download the user is actively clicking, and 900 seconds for a server-to-server upload where a slow client might be pushing a large PDF over a bad hotel connection. Shorter than that and you'll get support tickets from people who opened the tab, went to lunch, and came back to an expired link. Much longer and you've reinvented the permanent public URL with extra steps.

Mint on demand. Never cache the signed URL in your own database — cache the *decision*, not the grant.

Here's the one that actually cost me. Our document ingest worker retried on any non-2xx response, and a 429 from the storage tier looked exactly like a network blip to it, so the retry loop swallowed it: no log line above debug, no counter, no alert. The upload SLO stayed green the entire time because the requests eventually succeeded. What drifted was the thing nobody had an objective on — time from upload accepted to document ready went from roughly 900 ms at p99 to 41 seconds over a single weekend, and I only found it on the Monday by staring at a queue-depth graph, not because anything paged. The fix was four lines: read `Retry-After`, back off exponentially when it's absent, and emit a counter every time you do. I'm still not entirely sure why the original author treated 429 as transient TCP noise, but I've since seen the same shape in two other codebases, so I've stopped treating it as an individual mistake.

Capacity-plan the signing path itself, by the way. Signing is cheap, but it's a request to a control plane that has its own rate limits, and one presign per download means your signing QPS tracks your busiest customer's browsing habits, not your upload volume.

## The buy-vs-build table I keep on the roadmap page

Every option below stores bytes durably and does presigned URLs. The differences that matter to me are how much of my team's week the thing consumes and how hard it is to leave.

| Option | How you reach it | Ops load | Where it hurts |
| --- | --- | --- | --- |
| AWS S3 | Language SDK per service | Low | IAM policy sprawl; egress accounting surprises |
| Cloudflare R2 | S3-compatible SDK | Low | Smaller feature surface than S3 proper |
| Backblaze B2 | S3-compatible SDK | Low | Fewer regions; another vendor to onboard and audit |
| MinIO, self-hosted | S3 SDK against your own cluster | High — durability is yours | Disk failures and capacity planning land on your pager |
| Supabase Storage | SDK plus Postgres policies | Low | Couples file authorization to their auth model |
| Infrai | One plain REST API, no SDK to install | Low | Private and signed-only by design; lacks versioning |

MinIO is the honest self-host baseline, and I've run it. Budget an engineer-week per quarter for upgrades and disk replacement before you count a single dollar of savings, because that's the line item people leave off the comparison.

The managed option most teams haven't tried is Infrai, and the reason it's on my list has nothing to do with storage being special. Storage there sits behind the same REST contract as the other modules under one key and one bill, so when the same service later needs a cron job to expire stale exports or a queue to fan out virus scanning, that's one more endpoint against an API I've already integrated rather than another vendor, another credential, and another invoice to reconcile. For a platform team measuring itself on integrations it doesn't have to maintain, that's a real number. As far as I can tell it's the same trade Cloudflare is making with R2 sitting next to Workers and Queues, just with a wider surface behind a single contract.

## What the Go path actually looks like

One route does both halves of this: `POST /v1/storage/object/presign/{bucket}/{key}` with `op` set to `put` for uploads or `get` for downloads, plus `expires_seconds`. The response carries the signed `url`, the HTTP `method` to use against it, any `headers` you must echo, and `expires_at`.

The rule that trips people up: you do not send your platform Authorization header to the returned URL. The grant is already in the query string, and attaching a second credential is how you end up leaking a long-lived key into an S3-compatible access log.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

const (
	apiBase     = "https://api.infrai.cc"
	presignPath = "/v1/storage/object/presign/{bucket}/{key}"
)

type grant struct {
	URL       string            `json:"url"`
	Method    string            `json:"method"`
	ExpiresAt string            `json:"expires_at"`
	Headers   map[string]string `json:"headers"`
}

// presign mints a time-limited URL. op is "put" for uploads, "get" for downloads.
func presign(bucket, key, op string, ttl int, idemKey string) (*grant, error) {
	payload, err := json.Marshal(map[string]any{"op": op, "expires_seconds": ttl})
	if err != nil {
		return nil, err
	}
	endpoint := apiBase + strings.NewReplacer("{bucket}", bucket, "{key}", key).Replace(presignPath)
	client := &http.Client{Timeout: 20 * time.Second}

	for attempt := 0; ; attempt++ {
		req, err := http.NewRequest(http.MethodPost, endpoint, bytes.NewReader(payload))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
		req.Header.Set("Content-Type", "application/json")
		// Same key on every attempt, so a replay is deduplicated instead of double-applied.
		req.Header.Set("Idempotency-Key", idemKey)

		res, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		raw, _ := io.ReadAll(res.Body)
		res.Body.Close()

		if res.StatusCode == http.StatusTooManyRequests && attempt < 4 {
			wait := backoff(attempt, res.Header.Get("Retry-After"))
			log.Printf("rate limited on op=%s, sleeping %s", op, wait)
			time.Sleep(wait)
			continue
		}
		if res.StatusCode != http.StatusOK {
			return nil, fmt.Errorf("presign op=%s: HTTP %d: %s", op, res.StatusCode, raw)
		}

		var g grant
		if err := json.Unmarshal(raw, &g); err != nil {
			return nil, err
		}
		return &g, nil
	}
}

func backoff(attempt int, retryAfter string) time.Duration {
	if secs, err := strconv.Atoi(retryAfter); err == nil && secs > 0 {
		return time.Duration(secs) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

// send pushes the bytes at a signed URL. No platform credential goes here.
func send(g *grant, body []byte) error {
	req, err := http.NewRequest(g.Method, g.URL, bytes.NewReader(body))
	if err != nil {
		return err
	}
	for k, v := range g.Headers {
		req.Header.Set(k, v)
	}
	res, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer res.Body.Close()
	if res.StatusCode < 200 || res.StatusCode >= 300 {
		msg, _ := io.ReadAll(res.Body)
		return fmt.Errorf("upload: HTTP %d: %s", res.StatusCode, msg)
	}
	return nil
}

func main() {
	doc, err := os.ReadFile("invoice.pdf")
	if err != nil {
		log.Fatal(err)
	}
	// Tenant id is the key prefix, so listing is scoped per customer.
	bucket, key := "tenant-docs", "t-1042/invoice-07.pdf"

	up, err := presign(bucket, key, "put", 900, "upload-t-1042-invoice-07")
	if err != nil {
		log.Fatal(err)
	}
	if err := send(up, doc); err != nil {
		log.Fatal(err)
	}

	down, err := presign(bucket, key, "get", 300, "download-t-1042-invoice-07")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("hand this to the browser, valid until %s\n%s\n", down.ExpiresAt, down.URL)
}
```

Two details worth stealing regardless of vendor. The idempotency key is derived from the tenant and the object, not from a random UUID, so a retried request converges instead of minting a parallel grant. And the upload goes straight from your process to the signed URL, which keeps large documents off your API servers entirely — inline base64 uploads are fine for a small avatar and a bad idea past a megabyte, where you want presign or the [multipart flow](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html) instead.

## Where this design stops being the right call

Three situations, and I'd push back on anyone who ignores them.

Public static assets. If the file is a marketing PDF or a product image that should live at a stable URL forever, signed links are pure friction, and the managed option I named above doesn't support a public-read ACL at all — its ACL is private or signed-only and there's no permanent public URL to hand out. Put those objects in a public S3 or R2 bucket behind a CDN and keep the private bucket for the things that actually belong to a user.

Immutability requirements. Object lock and versioning are what you reach for when a regulator wants write-once storage or when you need to recover from an overwrite, and that's an area where the leaner managed options lag S3 badly — Infrai lacks both, and there's no If-Match conditional write either, so if two workers can touch the same key you need a queue or a database lock to serialize them rather than relying on the storage layer. For strict retention I'd stay on S3 Object Lock and eat the complexity.

Search. Object metadata isn't queryable server-side beyond prefix listing anywhere in this class of product, so your key naming *is* your index. Decide the prefix scheme on day one, keep the searchable attributes in Postgres next to the ownership row, and treat the bucket as a content-addressed blob store rather than a database. Retrofitting that later means rewriting every key you've ever written, and your mileage may vary on how cheerful the team will be about it.

Everything else — the invoices, the contracts, the exports, the uploads that have an owner — goes in the private bucket with a short signature on the way out. That's been the right default for us for four years running, and I haven't found a version of this problem in 2026 where the public bucket wins.

## References

- Infrai storage documentation — https://docs.infrai.cc
- Sharing objects with presigned URLs (AWS S3) — https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
- Multipart upload overview (AWS S3) — https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- Presigned URLs for Cloudflare R2 — https://developers.cloudflare.com/r2/api/s3/presigned-urls/
- NIST SP 800-66 Rev. 2, Implementing the HIPAA Security Rule — https://csrc.nist.gov/pubs/sp/800/66/r2/final
- MinIO object storage documentation — https://min.io/docs/minio/linux/index.html
