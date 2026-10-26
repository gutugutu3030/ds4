# Serving Models

[README](../README.md) | [Client setup](CLIENTS.md)

## Start locally

```sh
./ds4-server --ctx 32768
```

The default address is `http://127.0.0.1:8000`. Use `--host 0.0.0.0` to listen
on other interfaces. Restrict access to trusted clients; for an Internet-facing
deployment, put authentication and TLS in front of the server.

`--cors` enables browser cross-origin headers. It does not change the listening
address or provide access control.

## Which peers can connect

`--host` can be repeated, or take a comma-separated list, so one process can
hold loopback and a second interface without binding the wildcard:

```sh
./ds4-server --host 127.0.0.1 --host 100.115.220.98
```

`--allow-host` filters the *peer* address instead of the local address. Repeat it
or separate entries with commas; each entry is an IPv4 address, a CIDR prefix, or
`localhost` for the whole `127.0.0.0/8` block:

```sh
./ds4-server --host 0.0.0.0 \
  --allow-host 127.0.0.0/8,192.168.11.42,100.64.0.0/10
```

The check happens when the connection is accepted, before the request is read, so
a peer outside the list gets no response and never reaches a client thread or the
request queue. Each refused address is logged once with the address it used, which
is how you find the value to add:

```
ds4-server: refused peer 192.168.11.77: not in --allow-host
```

An entry with host bits set (`192.168.11.42/24`) is a startup error rather than a
silent widening to the whole `/24`, and any unparsable value stops startup. With
no `--allow-host`, every peer the bind address reaches is accepted, which is the
behaviour earlier versions had; binding anything but `127.0.0.1` then logs a
warning.

The listener is IPv4 only. A Tailscale peer connecting over the tailnet appears
with its `100.64.0.0/10` address, so allow that range or the node's address. Two
things follow from that: an IPv6-only peer cannot reach the server at all, and
traffic proxied by Tailscale Serve arrives from the proxy rather than from the
client, so per-client allowlisting needs a direct tailnet connection.

`--allow-host` decides who can talk to the server; it does not encrypt or
authenticate anything. For an Internet-facing deployment, keep authentication and
TLS in front of it, and remember `--ui-auth` covers only the chat page while
`/v1/*` stays unauthenticated to any peer that passes this filter.

The selected build chooses the backend. Pass `-m FILE` for an explicit model.
Use `--chdir /path/to/ds4` when starting outside the project directory so
relative runtime files such as Metal kernels can be found.

## Chat page login

```sh
./ds4-server --ui-auth-file /usr/local/etc/ds4/ui-auth
```

`--ui-auth USER:PASSWORD` makes the chat page at `/` answer `401` with a Basic
challenge until the browser submits those credentials. Nothing else is gated:
the `/v1/*` endpoints stay unauthenticated so existing clients keep working, and
anyone who can reach the port can still call them directly.

`--ui-auth-file FILE` reads the same `USER:PASSWORD` from the first line of a
file, and is the form to use wherever the server is started for you. Arguments
are readable to every local user through `ps`, and a launcher that keeps its
whole command line in a unit file stores the password there too.

`./setup_ui_auth.sh` does that for you: it prompts for the password twice with
echoing off, writes the file `600`, and hands it to the account the server runs
as. The password never appears in a command argument or in `ps`.

```sh
./setup_ui_auth.sh                      # defaults: login $USER, service user llm
./setup_ui_auth.sh -u ada -s ds4 -f /usr/local/etc/ds4/ui-auth
```

The server refuses to start when the file is readable or writable by other
users, since a credential anyone can read is not one; group access is allowed and
gets noted, so a service account reading through its own group works as
`root:service 0640`. One `USER:PASSWORD` per file, `:` splitting at the first
occurrence so the password may contain more. Each part has to fit in 256 bytes,
which is what an HTTP Basic header can carry: longer input stops startup rather
than producing a login that can never succeed.

Basic sends the password base64-encoded, not encrypted, so use it on a trusted
network or behind a TLS-terminating proxy. The browser keeps the credentials
until the tab closes; there is no logout.

## APIs

| Endpoint | Use |
| --- | --- |
| `GET /v1/models` | Loaded model information |
| `POST /v1/chat/completions` | OpenAI-style chat |
| `POST /v1/responses` | Responses-style requests and continuations |
| `POST /v1/completions` | Text completions |
| `POST /v1/messages` | Anthropic-style messages |

The Flash and PRO names accepted by the model endpoints are compatibility
aliases, not separate loaded models. The GGUF passed at startup selects the model.

```sh
curl http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"Explain Redis streams."}],"stream":true}'
```

Chat, Responses, and Anthropic support tools and SSE streaming. Reasoning is
returned separately from visible text in each API's native form. Standard
sampling and output-budget fields are supported; explicit request parameters
take precedence over defaults.

The default sampling settings are temperature 1, top-p 1, and min-p 0.05.
For DeepSeek, thinking is on by default. `reasoning_effort=max` selects Think
Max only with sufficient context; otherwise it falls back to normal thinking.
`xhigh` maps to normal thinking, not Think Max. Use `think:false`, a disabled
thinking object, or a non-thinking model alias for direct answers.

## Multiple sessions

```sh
./ds4-server --ctx 4096 --batched-session 4
```

Without `--batched-session`, there is one resident session. With it, the server
preallocates independent KV states and queues requests when all slots are busy.
Choose context and slot count together: a context that fits once may not fit
four times. Idle slots can be cached before reuse; active requests are not evicted.

Where the model supports it, the slots share one prefill workspace instead of
each keeping its own, so an extra slot costs only its caches. That matters most
for Qwen3.8 Flash Next, whose transients are sized by the prefill chunk rather
than by the context: at the default chunk they run to several GiB per session.
The startup line reports both figures.

| Backend/model | Decode execution |
| --- | --- |
| Metal, resident Flash | Native shared-expert/QKV batching where supported |
| Metal, resident V4.1 Flash | Native decoding for 2-8 sessions |
| Metal RDMA TP, V4.1 Flash | Native decoding for 3-8 sessions; ordered fallback for two |
| Metal SSD streaming, V4.1 Flash | Ordered fallback |
| Metal, GLM 5.2 | Ordered fallback |
| Metal, GLM 5.3 | Native batching through 2051 visible tokens; ordered fallback afterward |
| Metal, Qwen3.8 Flash Next | Native batching of the shared work; recurrent state, caches and PLE history stay per session |
| CUDA, supported multi-GPU Flash TP layout | Native grouped decode and mixed prefill/decode |
| Single-GPU CUDA, including Spark | Ordered fallback |

Fallback executes the rows separately. It provides concurrency and scheduling
fairness, not the aggregate speedup of native batching. Native grouping may
change floating-point reduction order slightly. V4.1 sessions containing images
use the ordered fallback.

Long prefills yield to active decoders in bounded intervals, normally 128
tokens. `--mixed-prefill-quantum N` changes that interval for testing.
Session-batched serving uses ordinary target decoding, except Qwen3.8 on
Metal, where `--mtp` also batches speculative decoding. Its
`--mtp-exact-sampling` mode uses ordinary batches for nonzero-temperature
requests. Other models do not use MTP/DSpark while session batching is active.
For the eight-L40S example, see [CUDA GPUs](CUDA_MULTI_GPU.md#serve-multiple-users).

## Images

Start with the matching language GGUF and `--vision FILE`; see
[model-specific instructions](MODELS.md#vision).

OpenAI chat and Responses accept inline PNG/JPEG data URIs. Anthropic accepts
base64 image sources. Remote URLs and server-side file paths are rejected.
Image blocks preserve their order in the request. The limit is 16 images and
a 64 MiB HTTP body.

## Disk KV cache

Disk caching saves useful prefixes across slot reuse and server restarts:

```sh
./ds4-server --ctx 100000 \
  --kv-disk-dir /tmp/ds4-kv --kv-disk-space-mb 8192
```

Clients can resend complete conversation histories. The server first tries
the live token prefix, then compatible rendered-text prefixes from disk, and
prefills the new suffix. With multiple slots, each slot has its own live state;
disk is the additional persistence layer, not the only way to retain sessions.

TP cache loading rebuilds the saved token prefix on both ranks. It is not an
instantaneous restoration of both GPUs. Pipeline loading redistributes the
saved layer state over its route.

Defaults are intended to avoid saving fragile token boundaries. For unusual
workloads, the controls are `--kv-cache-min-tokens`,
`--kv-cache-cold-max-tokens`, `--kv-cache-continued-interval-tokens`,
`--kv-cache-boundary-trim-tokens`, and `--kv-cache-boundary-align-tokens`.
Check `./ds4-server --help` for their defaults.

Quantization variants may share compatible prefixes. Add
`--kv-cache-reject-different-quant` for same-quant reuse only.
Cache files contain prompt text and model state: treat the directory as
private. It is disposable; stop the server before clearing it.

## Tool history and debugging

For DeepSeek, the server preserves sampled DSML tool blocks and assigns
unguessable tool IDs. Replaying those IDs avoids retokenizing a differently
formatted JSON history. The bounded replay map can be stored in cache files.
When exact replay is unavailable, canonical rendering may require rebuilding
part of the prefix.

`--tool-memory-max-ids` bounds this map.
`--disable-exact-dsml-tool-replay` disables it for diagnostic comparisons.
Use `--trace /tmp/ds4-trace.txt` to record prompt rendering, cache decisions,
generated text, and tool-parser events. Traces can contain sensitive content.

Cache formats are implementation details. The current header and extension
definitions are in [ds4_kvstore.h](../ds4_kvstore.h) and
[ds4_kvstore.c](../ds4_kvstore.c); model-specific payload handling is in
[ds4.c](../ds4.c).
