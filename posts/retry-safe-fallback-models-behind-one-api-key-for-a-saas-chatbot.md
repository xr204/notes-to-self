# Retry-Safe Fallback Models Behind One API Key for a SaaS Chatbot

**Short answer:** Choose the chatbot API architecture that preserves request identity, exposes provider and model outcomes, and lets you bound retries; one credential and a long model catalog are useful operationally, but they do not make fallback safe by themselves.

I learned that distinction from an incident, so my evaluation starts with failure semantics rather than a provider matrix. For an in-app chatbot, the real unit of reliability isn't the HTTP request. It's the user turn, including its identity, deadline, accumulated output, usage record, and final state. If a gateway can't show me how that unit behaves during ambiguity, I won't put its catalog size into a capacity plan.

## How should a SaaS chatbot API retry across fallback models?

A fallback chain should retry only an operation whose outcome is known to be absent, or repeat an operation carrying the same idempotency identity. Those are different guarantees. A timeout says the caller stopped waiting; it doesn't prove that the upstream model never accepted the request. Switching immediately from one model to another can therefore create two live generations, two usage events, or two attempts to persist an assistant message. I've had the database version of this mistake in production: our naive retry ran the same write twice and created 2 assistant rows for one user turn. The first insert committed, its acknowledgment missed our deadline, and the retry used a fresh identifier. Nothing exotic happened. Our code had confused an unknown result with a failed operation — a small wording error in a runbook that became a customer-visible duplicate. The invariant I use now is blunt: one logical turn gets one stable operation ID from ingress through storage, routing, and accounting. The router may record several attempts under it, but exactly one attempt may win the right to publish. If streaming has emitted a visible token, automatic model switching stops; resuming a partial natural-language answer against another model is not a retry in any useful sense. It's a new generation with different context. This makes the selection question testable. Ask how the API represents an attempt, whether the caller supplies a stable key, what happens after a caller deadline, and whether usage can be reconciled by logical turn. Don't accept “automatic fallback” as a complete answer. I'm not sure why retry counts still appear in so many architecture diagrams without an ambiguity state, but your mileage may vary; I need that state because the on-call engineer will eventually meet it at 03:00.

## The control loop needs a budget, not a model list

I define a user-turn SLO before choosing the runtime: a latency objective, a success objective, and an explicit ceiling on work amplification. The numbers depend on the product, so I won't invent universal targets. A support assistant that can wait several seconds has a different error budget from inline completion, and a regulated workflow may prefer a clean failure over an answer from a model that hasn't passed the same evaluation suite.

Capacity planning follows the retry tree. If peak accepted turns are `R`, the maximum configured attempts are `A`, and the fraction reaching each attempt is `f(i)`, upstream request demand is `R * sum(f(i))`, not `R`. That sum must include hedges, timeout overlap, and shadow evaluation. Otherwise fallback quietly consumes the headroom intended to save you during an incident. Keep it bounded.

The routing policy should classify outcomes rather than treating every non-success alike. A local queue deadline means no upstream request should start. A definitive client-input rejection should return without fallback. A throttling response can move to an eligible alternate if enough end-to-end deadline remains. An ambiguous timeout should close publication for late losers and retain the same operation ID. Policy also needs a circuit breaker per route and model, because repeatedly probing a degraded dependency spends both latency and error budget.

I want traces to carry the logical turn ID, attempt number, chosen route, time-to-first-token, completion state, and a normalized outcome class. Logs alone aren't enough for overlapping attempts. Metrics then answer the questions I use in review: how often did fallback rescue a turn, how often did it amplify work, which path exhausted the deadline, and what fraction of turns ended after partial output? Raw prompts and responses need separate privacy controls; observability does not grant permission to retain customer text.

## A preventative Go path for duplicate-safe publication

The following sketch keeps provider calls behind a generic interface. It doesn't pretend to offer exactly-once execution across a network. Instead, it gives every attempt the same operation ID, limits total work, and uses an atomic publisher so only one completed result becomes visible. Production code also needs persistent attempt records, cancellation propagation, circuit breakers, and streaming rules, but this is the part I insist on seeing in a design review.

```go
package runtime

import (
    "context"
    "errors"
    "fmt"
)

type Request struct {
    OperationID string
    Prompt      string
}

type Result struct {
    Text  string
    Usage int64
}

type Provider interface {
    Generate(context.Context, Request) (Result, error)
}

type Publisher interface {
    PublishOnce(context.Context, string, Result) (bool, error)
}

type Route struct {
    Name string
    Call Provider
}

var ErrAmbiguous = errors.New("generation outcome is unknown")

func Run(ctx context.Context, req Request, routes []Route, pub Publisher) (Result, error) {
    if req.OperationID == "" {
        return Result{}, errors.New("operation ID is required")
    }

    const maxAttempts = 2
    for i, route := range routes {
        if i == maxAttempts {
            break
        }
        if err := ctx.Err(); err != nil {
            return Result{}, err
        }

        result, err := route.Call.Generate(ctx, req)
        if err != nil {
            if errors.Is(err, ErrAmbiguous) {
                return Result{}, fmt.Errorf("%s: %w", route.Name, err)
            }
            continue
        }

        won, err := pub.PublishOnce(ctx, req.OperationID, result)
        if err != nil {
            return Result{}, err
        }
        if won {
            return result, nil
        }
        return Result{}, errors.New("operation already published")
    }
    return Result{}, errors.New("attempt budget exhausted")
}
```

Notice the conservative handling of `ErrAmbiguous`: it does not fan out again. A deployment can reconcile that attempt asynchronously or let the user start a new turn, but it must not manufacture certainty. Also notice that `maxAttempts` is a policy ceiling, not the length of `routes`. A catalog can contain dozens of eligible models while a latency budget permits two tries.

This pattern is not suitable when a user explicitly requests multiple independent candidates, because publication is intentionally singular. It also doesn't solve semantic portability: tool schemas, safety behavior, tokenization, context limits, and streaming events can vary across model families. Those differences belong in adapters plus contract tests, not in the retry loop.

## Buy versus build depends on who carries the pager

“One key” can mean two architectures. A managed gateway owns the external credential and gives the application one credential. A self-hosted gateway gives the application one internal credential while your team still manages upstream credentials, rotation, scaling, and upgrades. The application experience may look similar; the ownership model is not. LiteLLM is an open-source, self-hosted LLM gateway example, useful as evidence that the self-host route is real rather than a whiteboard abstraction.

| Decision axis | Managed gateway | Self-hosted gateway | Direct provider adapters |
|---|---|---|---|
| On-call boundary | Vendor runs the gateway; your team still owns policy and app behavior | Your team owns gateway availability and upgrades | Your team owns every adapter and routing path |
| Credential scope | One application credential can front upstream access | One internal credential can front credentials you operate | Separate upstream credentials reach the application boundary |
| Change control | Faster adoption, less infrastructure control | Full policy and deployment control | Maximum per-provider control, highest integration surface |
| Lock-in pressure | Gateway request, policy, and telemetry contract | Self-hosted gateway contract and its data model | Application abstractions and provider-specific behavior |
| Best fit | Small platform team buying operational coverage | Team with gateway expertise and a reason to own it | Narrow model set or strict provider-specific requirements |

The catch is straightforward: a managed option is not suitable when policy requires upstream credentials and request data to remain inside an environment you operate, or when gateway-level customization is central to the product. Stick with self-hosting in that case, accepting the pager and capacity burden. Direct adapters remain reasonable when the model set is deliberately small and provider-specific features matter more than portability.

My buy-versus-build review asks for failure-domain diagrams and staffing, not feature checkmarks alone. Who rotates credentials? Who tests a model retirement? Who reconciles usage? Who owns regional capacity? If the honest answer is “the application team” for most rows, the supposed managed layer has not removed much toil. Conversely, building a gateway to avoid contractual lock-in can create a very immediate operational dependency on two engineers who understand it.

## Prove fallback before production traffic does

A bake-off should replay representative, consented prompts through adapters and score behavior at the turn boundary. I include short and long contexts, tool calls, structured output, cancellation, slow first tokens, mid-stream disconnects, throttling, and a response that arrives after the caller deadline. The expected result is not always an answer. Sometimes the correct result is a bounded, observable failure with no duplicate publication.

Run fault injection below the router so tests can delay an acknowledgment after work completes, reject an attempt definitively, or consume the remaining deadline. Then verify invariants in storage: one published assistant message per operation ID, no attempt after budget expiry, losing attempts unable to publish, and usage records attributable to attempts. Averages hide this. I use latency distributions and failure-class counts, then review fallback rescue rate beside work-amplification rate.

Model quality needs its own gate. A backup that returns fluent but unacceptable answers has reduced transport errors while violating the product SLO. Evaluate every eligible route against the same task set, safety policy, tool contract, and output parser before adding it to policy. For an audio-input product, speech recognition is a separate preprocessing boundary; the open-source Whisper repository is one reference implementation, but audio transcription failure should not be confused with chat-model fallback.

Ship the policy in stages: record-only classification, a small traffic slice with publication disabled for alternate attempts, then bounded live fallback. Rollback should disable routes without an application release. The durable conclusion is deliberately vendor-neutral: select the ownership model whose ambiguity handling, telemetry, contract tests, and pager load your team can verify. A shared credential reduces integration surface. Reliability comes from the control loop around it.

## References

- https://github.com/BerriAI/litellm
- https://github.com/openai/whisper
