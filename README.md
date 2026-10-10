# vLLM SDK for Workshop

This SDK provides vLLM, a high-throughput inference and serving engine for
large language models, with its OpenAI-compatible API server. A Python 3.12
runtime with vLLM and PyTorch is prebuilt into the SDK; each accelerator is
published on its own track: `cuda` and `cpu`. Downloaded models,
compilation caches and server configuration are persisted on the host across
workshop updates, and an opt-in `vllm serve` service is exposed on port 8000.

---

## Reference workshop

A minimal workshop:

```yaml
# workshop.yaml
name: dev
base: ubuntu@24.04
sdks:
  - name: system
    plugs:
      vllm-server:
        interface: tunnel
        endpoint: localhost:8000
  - name: vllm
    channel: cuda/edge
```

The `system` plug makes the server reachable from the host at
`http://localhost:8000`.

### Tracks

| Channel | Backend | Platforms |
|---|---|---|
| `cuda/<risk>` | CUDA 13; NVIDIA driver R580 or newer on the host | ubuntu@24.04 and ubuntu@26.04, amd64 and arm64 |
| `cpu/<risk>` | x86 with AVX2/AVX512, or Arm | ubuntu@24.04 and ubuntu@26.04, amd64 and arm64 |

There is deliberately no `latest` track: always give a `channel:`.

---

## Using the SDK

### Serve a model

Inside the workshop, set the model in `~/.config/vllm/server.env`:

```bash
VLLM_MODEL="Qwen/Qwen3-0.6B"
```

Then, on the host, apply it:

```bash
workshop refresh
```

The `vllm` user service starts and downloads the model on first use. Once it
is up, query it from the host:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-0.6B","messages":[{"role":"user","content":"Hello"}]}'
```

Follow startup with `journalctl --user -u vllm -f` inside the workshop.
`VLLM_SERVE_ARGS`, `HF_TOKEN` (gated models) and `VLLM_API_KEY` are set in the
same file. Clear `VLLM_MODEL` and refresh to stop the service.

On the `cpu` track, vLLM reserves most of the workshop's memory by default; set
the KV cache size in GiB instead, for example `VLLM_CPU_KVCACHE_SPACE=4` in
`server.env` or in the environment of `vllm-python`.

### Offline inference from Python

`vllm-python` runs the SDK's Python interpreter:

```bash
vllm-python -c 'from vllm import LLM; print(LLM("facebook/opt-125m").generate("Hello")[0].outputs[0].text)'
```

The runtime is read-only. For extra Python packages, create a separate virtual
environment, for example with the `uv` SDK.

### Verify

```bash
# On the host:
workshop info
# Inside the workshop:
vllm --version
```

---

## Plugs (resources this SDK consumes)

### `gpu`

- Interface: `gpu`
- Purpose: access to the host's NVIDIA GPU.

### `models`

- Interface: `mount`
- Workshop target: `/home/workshop/.cache/huggingface`
- Purpose: persists downloaded Hugging Face models between workshop updates.

### `cache`

- Interface: `mount`
- Workshop target: `/home/workshop/.cache/vllm`
- Purpose: persists vLLM compilation caches between workshop updates.

### `config`

- Interface: `mount`
- Workshop target: `/home/workshop/.config/vllm`
- Purpose: persists the `vllm` service configuration (`server.env`) between
  workshop updates.

## Slots (resources this SDK provides)

### `vllm-server`

- Interface: `tunnel`
- Workshop source: port `8000`
- Purpose: exposes the OpenAI-compatible `vllm serve` API.

---

## Documentation and guidance

- [vLLM documentation](https://docs.vllm.ai)

---

## Community and support

- [vLLM forum](https://discuss.vllm.ai)
- Please review our [Code of Conduct](https://ubuntu.com/community/ethos/code-of-conduct)
  before participating.

---

## License and copyright

Copyright 2026 Canonical Ltd. The SDK packaging is distributed under the
Apache License 2.0. The packaged vLLM is Copyright the vLLM project
contributors, distributed under the Apache License 2.0.
