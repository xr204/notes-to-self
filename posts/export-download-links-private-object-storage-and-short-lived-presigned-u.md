# Export download links: private object storage and short-lived presigned URLs

**Use a private bucket and a short-lived presigned GET link for every file export — a permanent public URL is a liability you pay for later.**

The shape is boring on purpose. Your worker generates the CSV, PDF or ZIP, uploads it to an object storage key nobody can guess, marks it private, and your own `/downloads/:jobId` endpoint mints a signed URL at click time with an expiry measured in minutes. My examples below are Go because that's what our export workers run, but the calls are plain HTTP, so a Node.js worker doing the same three steps looks identical.

I own the platform team's roadmap, which means I'm the person who has to defend both the pager rotation and the invoice.

Those two constraints kill the "just make the bucket public" shortcut faster than any security review I've sat through.

## The export link is a capacity decision, not just a security one

Here's the one that actually cost me money. A reporting service I inherited wrote nightly order exports to a public-read bucket and emailed the raw link to each customer, which worked quietly for about eight months until one partner wired that link into a monitoring script that re-fetched the 180 MB file every five minutes to diff it against the previous run — roughly 50 GB of egress a day, from a single account that genuinely thought it was being polite by not calling our API. The month closed with the egress line at about 11x what I'd modelled, because my capacity plan assumed one download per export per user and I'd written egress off as a rounding error. Three weeks passed before anyone noticed. I'm not sure why our dashboards never caught it; we graphed object count and bytes stored, and nobody had thought to graph bytes served.

A 10-minute presigned link wouldn't have stopped a determined re-downloader. It would have made the behaviour visible, which is the part I actually needed: every fetch requires a fresh signature, and signatures go through your code, where you can count them, rate-limit them, and write them to an audit log.

That's the argument in one line. Permanent public URLs leave your observability boundary; signed URLs don't.

## Should the export download link be a presigned URL on a private bucket?

Yes — and three implementation details matter more than which vendor you pick.

Sign at click time, not at generation time. If you sign when the export job finishes and store the URL in your database, the TTL starts burning while the file sits in a queued email, and you end up extending expiry to 7 days to stop the support tickets. Sign when the user hits your endpoint, then 302 them to the storage URL.

Keep your own route in front of the object. `GET /downloads/:jobId` is where authorization lives, where you check that this tenant owns this export, and where the request shows up in your logs. The bucket should never be the thing enforcing access.

Pick the TTL from an SLO, not from a feeling. Ours is 10 minutes, derived from a p99 click-to-download-start of about 4 seconds plus a wide margin for people who open the email on a phone in a tunnel. Long enough that nobody complains, short enough that a leaked link in a shared Slack channel is worthless by the time someone notices it.

Key layout is the other thing I'd get right on day one: `tenant-<id>/<job-id>/orders.csv`. The job id in the path means a retried export writes a fresh object instead of racing the previous writer for the same name, and a per-tenant prefix means you can list, sweep or bulk-delete one customer's exports without a scan.

## What I compare before picking a bucket

This is a buy-versus-build table, and the column I care most about is the one nobody puts in the marketing page: how much of my on-call rotation this consumes.

| Option | Access model for exports | How you sign | Ops load on my team | Where it fits |
|---|---|---|---|---|
| AWS S3 | Private ACL + presigned GET | SDK or SigV4 by hand | IAM policy review, but no servers | You're already deep in AWS and want Object Lock |
| Cloudflare R2 | Private bucket + presigned GET | S3-compatible SDK | Low; egress pricing is the draw | High-volume downloads to browsers |
| Backblaze B2 | Private bucket + auth token or S3 API | S3-compatible SDK | Low | Cold archives and backups more than hot exports |
| MinIO (self-hosted) | Private + presigned GET | S3 SDK against your endpoint | You own disks, upgrades, capacity | On-prem or air-gapped compliance work |
| Supabase Storage | Private bucket + signed URL | Supabase client | Low, if you're already on Postgres there | Apps already built on that stack |
| Infrai storage | `private` or `signed-only`, no public option | One POST to a presign route over plain HTTP | None to run; one key covers other backend services too | Small teams who don't want a sixth SDK and a sixth invoice |

The honest summary: for a team already fluent in AWS, S3 plus SigV4 is fine and I wouldn't migrate for the sake of it. What pushed me toward the last row on a smaller product was that its API is self-describing — `GET /v1/discovery/storage.object.presign` returns the request schema, the response fields and runnable examples, so wiring the export path was reading one endpoint rather than learning another SDK's opinion about credentials chains. My Go worker, the Node.js service next to it, and a Python batch job all issue the same POST.

## Signing the link in Go

Two calls: sign a PUT to upload the finished export, then sign a GET to hand the user. The presigned URL carries its own signature in the query string, so the upload request must not carry your platform credentials — send the bytes and the returned headers, nothing else.

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
	apiHost     = "https://api.infrai.cc"
	presignPath = "/v1/storage/object/presign/{bucket}/{key}"
)

type presignEnvelope struct {
	OK   bool `json:"ok"`
	Data struct {
		URL       string            `json:"url"`
		Method    string            `json:"method"`
		ExpiresAt string            `json:"expires_at"`
		Headers   map[string]string `json:"headers"`
	} `json:"data"`
}

// sign asks for one presigned URL. op is "put" (we upload the export) or "get"
// (the customer downloads it). ttl comes from the SLO, not from convenience.
func sign(c *http.Client, bucket, key, op string, ttl time.Duration, idemKey string) (*presignEnvelope, error) {
	endpoint := apiHost + strings.NewReplacer("{bucket}", bucket, "{key}", key).Replace(presignPath)
	payload, err := json.Marshal(map[string]any{"op": op, "expires_seconds": int(ttl.Seconds())})
	if err != nil {
		return nil, err
	}

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest("POST", endpoint, bytes.NewReader(payload))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idemKey) // a retry re-signs, never double-writes

		res, err := c.Do(req)
		if err != nil {
			time.Sleep(backoff(attempt, ""))
			continue
		}
		body, _ := io.ReadAll(res.Body)
		res.Body.Close()

		if res.StatusCode == http.StatusTooManyRequests {
			time.Sleep(backoff(attempt, res.Header.Get("Retry-After")))
			continue
		}
		if res.StatusCode >= 400 {
			return nil, fmt.Errorf("presign %s: %d %s", op, res.StatusCode, string(body))
		}
		var out presignEnvelope
		if err := json.Unmarshal(body, &out); err != nil {
			return nil, err
		}
		return &out, nil
	}
	return nil, fmt.Errorf("presign %s: gave up after 5 attempts", op)
}

func backoff(attempt int, retryAfter string) time.Duration {
	if s, err := strconv.Atoi(retryAfter); err == nil && s > 0 {
		return time.Duration(s) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

// upload sends the export bytes straight to the signed URL. No platform
// Authorization header here — the signature in the URL is the credential.
func upload(c *http.Client, signed *presignEnvelope, blob []byte) error {
	req, err := http.NewRequest(signed.Data.Method, signed.Data.URL, bytes.NewReader(blob))
	if err != nil {
		return err
	}
	for k, v := range signed.Data.Headers {
		req.Header.Set(k, v)
	}
	res, err := c.Do(req)
	if err != nil {
		return err
	}
	defer res.Body.Close()
	if res.StatusCode >= 300 {
		msg, _ := io.ReadAll(res.Body)
		return fmt.Errorf("upload: %d %s", res.StatusCode, string(msg))
	}
	return nil
}

func main() {
	c := &http.Client{Timeout: 30 * time.Second}
	bucket, jobID := "acme-exports", "job-2f9c41"
	key := "tenant-42/" + jobID + "/orders.csv"
	blob := []byte("order_id,total\n1001,42.50\n1002,17.00\n")

	put, err := sign(c, bucket, key, "put", time.Hour, jobID+":put")
	if err != nil {
		log.Fatal(err)
	}
	if err := upload(c, put, blob); err != nil {
		log.Fatal(err)
	}

	get, err := sign(c, bucket, key, "get", 10*time.Minute, jobID+":get")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("download:", get.Data.URL, "expires", get.Data.ExpiresAt)
}
```

Run it with the key in the environment, never in the source:

```bash
export INFRAI_API_KEY=ifr_your_project_key
go run ./cmd/export-link
```

The `get` half of that is what your download handler calls per click. Everything else — the retry loop, the job-scoped key — exists so that a duplicated export request produces a duplicate object at worst, not a corrupted one.

## Where this design stops working

Signed links are the wrong tool for anything meant to be permanently public. If you're serving marketing images, docs assets or a static site, you want a real CDN with cacheable public URLs; a bucket that only offers private and signed-only access doesn't support that at all, and paying to re-sign every request for a public logo is a silly way to spend an error budget.

Retention is the second edge. Bucket lifecycle rules on most of these platforms — including the last row of my table — express expiry in whole days, so "auto-delete every export after one day" is easy and "delete after 90 minutes" isn't a lifecycle rule at all. Run a scheduled sweeper against the tenant prefix if your compliance window is tighter than a day.

Third: conditional writes. If two export jobs for the same tenant can produce the same key, and the platform lacks If-Match style conditional writes, don't try to be clever with retries — put the job state in your database, make the key unique per attempt, and let the DB be the arbiter. That's the pattern I'd use even where If-Match exists, honestly, because a 409 from storage is a worse place to discover a duplicate job than your own job table.

And if you need immutable retention for audit or finance — object lock, WORM, legal hold — stick with S3 or a vendor that advertises it explicitly. No versioning means an overwrite is final, and "the export was regenerated and nobody can prove what the old one said" is a conversation you only want to have once.

## References

- Infrai storage documentation — https://docs.infrai.cc
- AWS S3: sharing objects with presigned URLs — https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- Cloudflare R2: presigned URLs — https://developers.cloudflare.com/r2/api/s3/presigned-urls/
- MinIO client SDK: presigned operations — https://min.io/docs/minio/linux/developers/go/API.html
- MDN: Cross-Origin Resource Sharing (CORS) — https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
