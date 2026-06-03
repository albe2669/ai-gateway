# First-token timeout (TTFT) with automatic backend fallback

This example demonstrates the `FirstTokenTimeout` field on `AIGatewayRouteRule`, a per-rule deadline on how long Envoy will wait for the **first response byte** from an upstream model backend on a streaming response.

The rule in `base.yaml` is configured with:

- `Timeouts.Request: 30s` - the overall response deadline.
- `FirstTokenTimeout: 5s` - Envoy will reset the upstream stream if no byte has arrived within 5s of the request being forwarded.

Two backends are wired up in priority order:

- `first-token-timeout-silent` (priority 0): accepts the TCP connection and then holds it open without sending anything, always triggering the TTFT deadline.
- `first-token-timeout-healthy` (priority 1): the standard `testupstream` configured to return a streaming OpenAI response.

`fallback.yaml` attaches a `BackendTrafficPolicy` whose retry triggers include `reset`, which turns the reset into a fall-over to the healthy backend.

## Prerequisites

A Kubernetes cluster with Envoy Gateway and the Envoy AI Gateway installed. The quickest way to get there locally is the [Getting Started guide](https://aigateway.envoyproxy.io/docs/getting-started/).

## Apply

```shell
kubectl apply -f examples/first_token_timeout/base.yaml
kubectl apply -f examples/first_token_timeout/fallback.yaml

# Wait for the Gateway to become ready.
kubectl wait --for=condition=Programmed gateway/first-token-timeout --timeout=120s
```

## Verify

Port-forward the Gateway and send a streaming chat completion. The first attempt should hit the stalling backend, stall for ~5s, then fall over to the healthy backend, which returns a response.

```shell
# The Service that fronts the Gateway is named envoy-<ns>-<gateway>-<hash>.
GATEWAY_SVC=$(kubectl get svc -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=first-token-timeout \
  -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n envoy-gateway-system "svc/$GATEWAY_SVC" 8080:80 &

curl -w '\nttft=%{time_starttransfer}s total=%{time_total}s\n' \
  -X POST http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
        "model": "ttft-demo",
        "stream": true,
        "messages": [{"role":"user","content":"hi"}]
      }'
```

You should see something along the lines of:

```json
{"choices": [{"index": 0,"message": {"role": "assistant","content": "The quick brown fox jumps over the lazy dog."},"finish_reason": "stop"}],"usage": {"prompt_tokens": 1,"completion_tokens": 100,"total_tokens": 300}}
ttft=5.084152s total=5.084261s
```

`ttft` should be just over `5s`, as that means the stalling backend held the stream until the per-try idle timer fired, after which Envoy retried to the healthy backend.
