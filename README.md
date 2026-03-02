# vast-opencode

A single command that rents a GPU on [Vast.ai](https://vast.ai), serves [Qwen3.5 35B-A3B](https://huggingface.co/Qwen/Qwen3.5-35B-A3B) via [vLLM](https://docs.vllm.ai), launches [OpenCode](https://opencode.ai), and tears everything down when you quit. Billing stops automatically.

## How It Works

```
vast-opencode
```

### 1. Preflight

Checks for and installs any missing dependencies:
- `vastai` CLI (via pipx)
- `opencode` (via curl installer)
- `jq`

Loads credentials from `~/.vast-opencode.env` if it exists, or prompts for:
- **Vast.ai API key** (from https://cloud.vast.ai/cli/)
- **HuggingFace token** (optional — the default model is public/Apache 2.0)

On first run, offers to save keys to `~/.vast-opencode.env` (chmod 600) so you're never prompted again.

### 2. GPU Selection

Searches Vast.ai for verified datacenter GPUs with 60GB+ VRAM, sorted by price. Presents an interactive menu:

```
Available GPU offers (sorted by price):

  #     GPU                     VRAM    Price       DL Mbps   Rel.    Location
  ──────────────────────────────────────────────────────────────────────────────
  1     1x RTX PRO 6000 S       95GB    $0.73/hr    850       99%     Michigan, US
  2     1x RTX PRO 6000 WS      95GB    $0.77/hr    920       98%     Nevada, US
  3     1x A100 SXM4            80GB    $0.87/hr    1200      99%     Massachusetts, US
  ...

Select an offer [1-10] (or 'q' to quit):
```

Context length is auto-sized to the GPU:
| VRAM | Context |
|------|---------|
| 80GB+ | 256K tokens (full native) |
| 60-79GB | 128K tokens |

### 3. Thinking Mode

```
Thinking mode:
  The model can "think" before responding — reasoning step-by-step
  internally before giving its answer.

  on   Better quality, uses more context, slower responses
  off  Faster responses, uses less context, still capable

Enable thinking mode? [Y/n]
```

When enabled, the model reasons in `<think>...</think>` blocks before answering. When disabled, it responds directly.

### 4. Provision

Rents the selected GPU and boots a vLLM container with:
- Qwen3.5 35B-A3B at FP8 quantization (~27GB weights)
- OpenAI-compatible API on port 8000
- HuggingFace token if needed (default model is public)

### 5. Wait for Ready

- Polls until the instance status is "running"
- Establishes an SSH tunnel (`localhost:8000` -> instance)
- Polls the vLLM `/v1/models` endpoint until the model is loaded (typically 2-5 minutes)

### 6. Code

OpenCode launches connected to the model through the SSH tunnel. You get a full terminal-based AI coding assistant backed by a private GPU. Nothing leaves the instance — your prompts travel through the encrypted SSH tunnel to vLLM running on the same machine.

```
═══════════════════════════════════════════════════════
  GPU session active — 1x RTX PRO 6000 S (95GB) @ $0.73/hr
  Model: Qwen/Qwen3.5-35B-A3B
  Context: 262144 tokens
  Thinking: ON
  Quit OpenCode to automatically destroy the instance
═══════════════════════════════════════════════════════
```

### 7. Teardown

When you quit OpenCode (or press Ctrl+C at any point):
- SSH tunnel is closed
- Vast.ai instance is **destroyed** (not just stopped — billing ends immediately)
- Temp config is cleaned up
- Session summary is printed:

```
>>> Session duration: 47m 12s
>>> Estimated cost: ~$0.4564
```

## Installation

```bash
# Clone
git clone https://github.com/1beb/vast-opencode.git

# Add to PATH
echo 'export PATH="$HOME/projects/vast-opencode:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Run
vast-opencode
```

Or just copy the script anywhere in your PATH:

```bash
curl -o ~/bin/vast-opencode https://raw.githubusercontent.com/1beb/vast-opencode/main/vast-opencode
chmod +x ~/bin/vast-opencode
```

## Options

```
vast-opencode                          # interactive (default)
vast-opencode -y                       # auto-pick cheapest, thinking on, no prompts
vast-opencode -v 80                    # only GPUs with 80GB+ VRAM
vast-opencode -c 131072               # force 128K context
vast-opencode -m Qwen/Qwen3-30B-A3B   # different model
vast-opencode --no-thinking            # skip thinking mode
vast-opencode -p 9000                  # use different local port
```

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `VAST_MODEL` | Model to serve | `Qwen/Qwen3.5-35B-A3B` |
| `VAST_CTX_LEN` | Context length (overrides auto) | auto by VRAM |
| `VAST_MIN_VRAM` | Minimum GPU VRAM in GB | `60` |
| `VAST_MIN_DISK` | Minimum disk space in GB | `80` |
| `VAST_MIN_RELIABILITY` | Minimum host reliability | `0.95` |
| `VAST_MIN_INET` | Minimum download bandwidth Mbps | `1000` |
| `VAST_THINKING` | `on` or `off` | prompted |
| `VAST_VLLM_IMAGE` | Docker image | `vllm/vllm-openai:nightly` |
| `VAST_READY_TIMEOUT` | Seconds to wait for model | `900` |
| `VAST_LOCAL_PORT` | Local tunnel port | `8000` |
| `HF_TOKEN` | HuggingFace token (optional) | none |

## Privacy

- vLLM runs on the rented instance — prompts never leave the machine
- Connection is an encrypted SSH tunnel
- Vast.ai verified datacenters have ISO 27001 / SOC 2 certification
- Instance is destroyed on exit — all data deleted
- No telemetry, no third-party API calls

## Requirements

- Linux or macOS
- `bash`, `ssh`, `curl`
- A [Vast.ai](https://vast.ai) account with credit
- A [HuggingFace](https://huggingface.co) token (only if using a gated model — the default model is public)

## License

MIT
