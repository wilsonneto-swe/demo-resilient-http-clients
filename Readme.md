# Resilient HTTP clients in .NET

Companion code for the video *Resiliência em Integrações HTTP com .Net* (in Portuguese). <!-- add the video link -->

Three small services: a flaky dependency, an API that calls it through a resilience pipeline, and a consumer that hits the API once a second and logs what it gets back. Run them side by side and watch each strategy kick in.

## The problem

A service that calls another service over HTTP inherits every failure of that dependency: errors, slow responses, outages. Without a policy, one slow call holds a thread for as long as the dependency wants, a burst of errors turns into a burst of retries, and the outage walks upstream into your own API.

`External` simulates that dependency on purpose: on each call it fails about a third of the time, hangs for 10 seconds another third, and answers normally the rest.

## The decision

One resilience pipeline per typed client, declared where the client is registered, with `Microsoft.Extensions.Http.Resilience` (Polly v8). No try/catch loops scattered through the code: the policy lives in one place and the client stays plain.

```csharp
builder.Services.AddHttpClient<WeatherForecastClient>(...)
    .AddResilienceHandler("WeatherForecastClientResilience", pipeline =>
    {
        pipeline.AddHedging(...);           // 300 ms without an answer: fire a parallel attempt, up to 5
        pipeline.AddTimeout(8s);            // total budget for the whole call
        pipeline.AddRetry(...);             // 5 retries, exponential backoff with jitter, max 5 s
        pipeline.AddTimeout(1s);            // budget for each attempt
        pipeline.AddCircuitBreaker(...);    // 90% failures over 10 s opens it for 20 s
        pipeline.AddRateLimiter(...);       // 4 calls per 10 s window, no queue
        pipeline.AddConcurrencyLimiter(2);  // at most 2 in flight
    });
```

Strategies run outer to inner in the order they're added. That order is the design: the total timeout wraps the retries, the per-attempt timeout sits inside them, and the breaker, rate limiter and concurrency limiter guard the actual call.

## The trade-off

Every strategy that protects you costs the dependency something.

- **Retry and hedging multiply load.** Hedging with 5 attempts on top of 5 retries can turn one call into dozens against a service that is already struggling. That's the exact moment it can least afford it.
- **The circuit breaker is the counterweight.** It stops the multiplication by failing fast for 20 seconds, which means refusing calls that might have succeeded.
- **The two timeouts have to nest.** If the per-attempt timeout isn't shorter than the total, retries never get a turn. Here it's 1 s inside 8 s.
- **Rate and concurrency limits protect the dependency, not you.** They reject your own calls when the window is full; the consumer sees those as errors too.

The numbers in this demo are deliberately aggressive so the behavior is visible within a minute. In production they come from the dependency's real latency and error rate, and from how much extra load it can take from you when it's degraded.

## What to watch

- `External` logs every injected error and delay.
- `Api` logs each outcome; failures come back as `ProblemDetails` with the exception type (timeout, broken circuit, rate limited).
- `ConsumerClient` logs status and elapsed time per request. Hedged calls come back fast even when the first attempt hangs; a 1 s timeout shows up as roughly 1 s instead of 10.

## Run

Three terminals:

```bash
dotnet run --project External
dotnet run --project Api
dotnet run --project ConsumerClient
```

Base URLs are in each project's `appsettings.json`. Swagger UI is on for `External` and `Api`.

## Projects

- `External`: the flaky downstream API
- `Api`: typed `HttpClient` with the resilience pipeline, exposed at `GET /weather`
- `ConsumerClient`: worker that calls the API every second and logs the result

.NET 8 · Microsoft.Extensions.Http.Resilience 8.7 · Polly v8
