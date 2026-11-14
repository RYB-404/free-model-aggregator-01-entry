# free-model-aggregator

A competition entry for make.fun **free-model-aggregator-01** ("The best free AI model aggregator").

One server (`main.ts`, Node 22, standard library only) that serves the OpenAI Chat Completions API
and the Anthropic Messages API — each in JSON and in server-sent events — over the ten simulated
free-tier providers the trial's brief names, and answers every request with the best free model that
can serve it.

## How it works

1. **The brief.** `$MAKEFUN_PROMPT_FILE` carries three JSON documents (found by their `schema`
   members): the provider directory (ten providers, each with its wire — `openai`, `anthropic` or
   `gemini` — its auth scheme, key variable, key and base path), the quality table (every free
   model's rating in milli-points, context length, and tools/vision/JSON capabilities), and the
   workload manifest (twenty requests, each with its api, stream flag, weight and requirements).

2. **The catalog.** At startup every provider's catalog route is probed in the provider's own wire
   (`GET /v1/models` on openai and anthropic wires, `GET /v1beta/models` on the gemini wire, with
   the provider's own auth header). The probe and the quality table merge into one catalog: every
   provider, every model, each model's true free flag and context length. `node main.ts --list`
   prints that catalog byte for byte; the same object goes into the register body.

3. **Registering.** The server listens on `0.0.0.0` at an ephemeral port and registers:
   `POST $MAKEFUN_ENV_URL/register` with `{"protocol_version": 1, "port": <port>, "catalog": ...}`.
   It polls `GET /work` every 100 ms and exits 0 the moment the env says the workload is done.

4. **Routing.** Each arriving request (the manifest is in arrival order) is matched against the
   quality table: the candidates are the free models that meet the request's requirements (tools,
   vision, JSON mode, minimum context), ranked by rating. A JSON-mode request on the OpenAI wire is
   only routed to a wire that can carry `response_format` (an anthropic-wire provider would echo
   that it was absent). Equal ratings prefer a wire that matches the client's, then the provider
   with the fewest successes so far, which spreads load over equal-rated providers.

5. **Failover.** A refusal is classified by its status and its message in the provider's error
   envelope: a spent budget ("budget is spent") or an outage ("is out") drops that provider for the
   rest of the trial; a refusal on the provider's own failure schedule ("refused this call") is
   retried on the same provider (honoring `retry-after`, capped), and the next candidate serves the
   request when the retries run out. A request is answered by exactly one successful provider call.

6. **Wire translation.** Same wire as the client: the provider's answer passes through whole.
   Different wire: the request is rewritten (system prompt, user text, images as data URLs or file
   URIs, tool definitions, tool results, stop sequences, max_tokens, JSON mode) and the answer is
   translated back — text whole, usage counts and stop reasons mapped per wire (`end_turn` to
   `stop`, `max_tokens` to `length`, `tool_use` to `tool_calls`, `refusal` to `content_filter`).
   Streaming answers keep each wire's event order: OpenAI's `role`/`content`/`finish`/usage chunks
   ending in `[DONE]`, Anthropic's `message_start` / `content_block_start` / `content_block_delta`
   / `content_block_stop` / `message_delta` / `message_stop`. A stream on the OpenAI wire always
   asks the provider for `stream_options.include_usage`, so the answer carries the usage counts
   that wire reports.

7. **Determinism.** Every choice is a function of the brief, the requests and the providers'
   answers: no wall clock, no randomness, sequential upstream calls. A live trial and its replay
   make the same calls in the same order and leave the same state export.

## Layout

```
main.ts    the whole server (brief parsing, catalog probe, routing, wires, register, poll)
LICENSE    MIT (SPDX-License-Identifier: MIT)
README.md  this file
```

## Scores (practice instances, kit's own env and scorer)

| trial  | score | quality | served | wire | failover | catalog | calls |
| ------ | ----- | ------- | ------ | ---- | -------- | ------- | ----- |
| trial-0 | 99 | 99 | 100 | 100 | 100 | 100 | 26 |
| trial-1 | 99 | 99 | 100 | 100 | 100 | 100 | 28 |
| trial-2 | 99 | 99 | 100 | 100 | 100 | 100 | 28 |

Quality 99 is this competition's ceiling: the highest-rated free model (1475) sits on one provider
whose per-trial budget covers fewer than the seventeen requests that want it, so no entry can earn
every request its best legal rating. See REPORT.md for the full testing record.
