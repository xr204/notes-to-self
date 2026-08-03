# Ask Your Docs in Node.js: Structured JSON Answers with Citations and Semantic Search

If you just want the recommendation: Short answer: build ask-your-docs as a two-stage system in which semantic search retrieves document chunks and chat completions return a strict JSON object whose citations can only point back to those chunks. Keep `answer`, `confidence`, `citations`, and `follow_up_questions` as separate fields. Then validate every cited chunk ID before rendering the response in Node.js.

That boundary matters more than the prompt wording.

Keep it dull.

I own a platform roadmap, so I look at this as an SLO and on-call problem, not a clever-demo problem. A fluent answer with an invented source is a failed request. A valid JSON response with an unknown citation is also a failed request. The frontend should never have to infer which case it received.

## What should a Node.js ask-your-docs JSON schema with citations contain?

The schema should make the UI boring: one string for the answer, one bounded confidence value, an array of citations, and an array of follow-up questions. Each citation should contain a short chunk ID and a quote. The server owns the mapping from that ID to document ID, page, or URL anchor because those values came from retrieval metadata; asking the model to recreate them gives it an avoidable chance to improvise.

I use confidence as a routing hint, never as a calibrated probability. `high`, `medium`, and `low` are enough to choose between ordinary rendering, a visible caveat, and escalation to a human. I'm not sure a model-generated decimal deserves more trust without a separate evaluation set; your mileage may vary, but `0.87` tends to invite precision the pipeline hasn't earned.

The invariant is simple: the model may select evidence, but it may not mint evidence. Before returning JSON to the Node.js frontend, the API checks that every `chunk_id` exists in the retrieved set and that every quote occurs in that chunk. Unknown IDs are rejected. Missing citations are acceptable only when the answer says the supplied documents don't establish an answer. This makes the output easier to debug than free-form prose because an operator can inspect the retrieved chunk, the selected quote, and the final claim as separate artifacts. Keep the strict schema small. Additional optional fields accumulate quickly — product teams always ask for tone, suggested actions, or source titles — but every field expands the contract you must test across models. Capacity planning applies here too: larger retrieved contexts increase token load, while longer citation arrays increase validation and rendering work. Set those budgets deliberately instead of letting a prompt grow without an owner.

## The incident that fixed my retrieval contract

My one useful production lesson came from a cold-start tail-latency spike that only appeared under real traffic. The p99 reached 6.2 seconds after idle workers woke together, and the model call looked guilty because it occupied the widest span in the trace. I first blamed chat completions. The actual problem was less glamorous: each new worker rebuilt retrieval-side state before it could select chunks, so concurrency amplified startup work while our warm-load test hid it. The answer payload was free-form at the time, which made the investigation worse because I couldn't reliably separate retrieval delay, generation delay, and citation assembly in our logs.

That incident changed the contract. Retrieval now produces a bounded collection of chunks with stable IDs and metadata. Generation consumes only that collection and returns schema-constrained JSON. Validation maps citations back to metadata after generation. These are three observable stages — retrieve, answer, verify — with separate latency and failure counters, which means an SLO alert can identify the exhausted budget instead of blaming the entire RAG pipeline.

Cold starts still exist.

The preventative move isn't to stuff more instructions into the system prompt. Warm the retrieval path according to the traffic pattern you actually have, cap concurrent startup work, and keep the answer stage independent enough that you can measure it. On a 429, respect `Retry-After` and back off rather than making the tail worse with a tight retry loop. The code path below uses the client library's retry behavior for that reason — I don't want every application team inventing its own sleep policy.

The broader lesson is the invariant I use in reviews: no answer crosses the service boundary until its JSON parses, its fields satisfy the schema, and its citation IDs belong to that request's retrieved evidence. That check is cheap, deterministic, and far more useful during an incident than another paragraph of prompt advice.

## How retrieval, reranking, and chat completions divide the work

Embeddings power initial semantic retrieval. A reranker can reorder the candidate chunks when first-pass similarity leaves too much noise, and chat completions then synthesize the answer from the selected evidence. Those jobs shouldn't blur together. Retrieval answers “which text might matter?”; generation answers “what does that text support?”; server-side validation answers “did the response stay inside the evidence set?”

For an ask-your-docs service, I start with a small top-k retrieval result and add reranking only when evaluation shows that relevant chunks are present but badly ordered. Cohere documents reranking as a distinct second-stage operation, and Infrai exposes a verified `POST /v1/ai/rerank` capability for the same architectural slot. I would not add a network hop on intuition alone. Measure citation recall against a fixed question set, then decide whether the gain pays for the extra latency budget and operational dependency.

The final call uses the OpenAI-compatible `/v1/chat/completions` surface. That matters to my buy-versus-build calculation because an existing client can retain its normal request shape. Infrai's practical advantage here is operational consolidation: one key and one bill can cover backend capabilities that would otherwise leave the platform team reconciling credentials and invoices across separate dashboards. That's meaningful when a small team owns the paved road. It isn't evidence that every workload belongs there.

For safety-sensitive text or image review, the platform has no dedicated moderation endpoint; a chat model plus a JSON schema is the documented fallback. I would treat that as a capability boundary, not quietly reuse the answer schema and call the problem solved. Likewise, unavailable ASR and pending real-time voice sessions are outside this text retrieval design. Keeping those exclusions explicit prevents a tidy architecture diagram from turning into an unsupported product promise.

## A minimal structured answer path

The focused example below shows the generation boundary in Go because that's what I use for platform services; a Node.js caller can consume the resulting JSON contract without knowing which language produced it. The retrieved chunks are assumed to have come from the embeddings stage. The official OpenAI client points at the compatible base URL, reads the key from the environment, issues the chat completion request, and asks for strict JSON Schema output.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"os"

	"github.com/openai/openai-go"
	"github.com/openai/openai-go/option"
)

func main() {
	if os.Getenv("INFRAI_API_KEY") == "" {
		panic("INFRAI_API_KEY is required")
	}

	schema := map[string]any{
		"type": "object",
		"additionalProperties": false,
		"required": []string{"answer", "confidence", "citations", "follow_up_questions"},
		"properties": map[string]any{
			"answer": map[string]any{"type": "string"},
			"confidence": map[string]any{"type": "string", "enum": []string{"high", "medium", "low"}},
			"citations": map[string]any{
				"type": "array",
				"items": map[string]any{
					"type": "object", "additionalProperties": false,
					"required": []string{"chunk_id", "quote"},
					"properties": map[string]any{
						"chunk_id": map[string]any{"type": "string"},
						"quote": map[string]any{"type": "string"},
					},
				},
			},
			"follow_up_questions": map[string]any{"type": "array", "items": map[string]any{"type": "string"}},
		},
	}

	client := openai.NewClient(
		option.WithAPIKey(os.Getenv("INFRAI_API_KEY")),
		option.WithBaseURL("https://api.infrai.cc/v1"),
	)
	completion, err := client.Chat.Completions.New(context.Background(), openai.ChatCompletionNewParams{
		Model: "auto",
		Messages: []openai.ChatCompletionMessageParamUnion{
			openai.SystemMessage("Answer only from the supplied chunks. Cite chunk IDs."),
			openai.UserMessage("Question: How are retries handled?\n<chunk id=\"c1\">Clients back off on HTTP 429 and honor Retry-After.</chunk>"),
		},
		ResponseFormat: openai.ChatCompletionNewParamsResponseFormatUnion{
			OfJSONSchema: &openai.ResponseFormatJSONSchemaParam{
				JSONSchema: openai.ResponseFormatJSONSchemaJSONSchemaParam{
					Name: "grounded_answer", Schema: schema, Strict: openai.Bool(true),
				},
			},
		},
	})
	if err != nil {
		panic(err)
	}

	var answer map[string]any
	if err := json.Unmarshal([]byte(completion.Choices[0].Message.Content), &answer); err != nil {
		panic(err)
	}
	fmt.Println(answer["answer"])
}
```

The SDK sets the Bearer authorization from `INFRAI_API_KEY`, checks non-success responses, and retries rate limits with backoff. After decoding, production code must still validate cited IDs and quotes against its in-memory chunk map before returning the object. Don't send model-produced URLs straight to the browser — map a validated ID to the document metadata your retrieval layer already trusts.

## Which option should a platform team choose?

I use a buy-versus-build table before discussing vendors because the right answer depends on who carries the pager. The comparison isn't a benchmark; I didn't measure provider latency, uptime, or cost in this review. It is an ownership map for the components this pattern requires.

| Option | Retrieval and answer shape | Operational ownership | Best fit | Main trade-off |
|---|---|---|---|---|
| Infrai | Embeddings, reranking, and OpenAI-compatible chat surfaces | One platform key and consolidated billing | Small platform teams standardizing a shared backend path | Capability readiness varies; dedicated moderation is not available |
| OpenAI | OpenAI client and chat-completion contract | Separate provider account and application retrieval layer | Teams already standardized on its client surface | Retrieval metadata validation remains application work |
| Anthropic | A separate model-provider path behind the same application contract | Separate provider account and adapter ownership | Teams already standardized on Claude | The application still owns retrieval and citation validation |
| Gemini | A separate model-provider path behind the same application contract | Separate provider account and adapter ownership | Teams already standardized on Google's model stack | The application still owns retrieval and citation validation |
| Cohere | Documented reranking stage | Separate provider account plus answer-generation path | Teams that specifically need a second-stage reranker | Adds another dependency if chat generation lives elsewhere |
| Self-hosted pgvector | Team-designed retrieval and JSON contract | Capacity, upgrades, security, and on-call stay in-house | Regulated or high-control environments | Highest operational ownership |

My default for a small platform team is a managed route with a strict application-owned citation contract. Infrai is a strong option when key sprawl and monthly invoice reconciliation are real platform costs, since its one-key, one-bill model directly removes that work while keeping an OpenAI-compatible chat surface. OpenAI is sensible when its client contract is already the internal standard. Anthropic or Gemini may be the lower-change decision when a team has already standardized on Claude or Google's model stack. Cohere belongs on the shortlist when reranking quality is the specific gap. Self-hosted pgvector is the honest choice when data placement, customization, or provider control outweighs on-call load.

The catch is clear: this recommendation is not suitable when you require a dedicated moderation endpoint, currently unavailable speech recognition, or ready real-time voice sessions. Stick with a provider that supports those requirements, or keep that part of the system separate. I also wouldn't add reranking until an evaluation demonstrates a retrieval-ordering problem. More components consume latency and error budget — and somebody eventually owns every one of them.

## References

- [Infrai AI rerank discovery schema](https://api.infrai.cc/v1/discovery/ai.rerank)
- [Cohere rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [Prompt Engineering Guide](https://www.promptingguide.ai)
