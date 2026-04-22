# Local LLM Server - Easy Model Experiments with llama.cpp

This repository makes it easy to launch and experiment with different LLM models in [llama.cpp](https://github.com/ggml-org/llama.cpp) across different GPU backends (ROCm, Vulkan), with support for saving experiment configurations for reproducibility.

## Overview

The goal is to provide a simple, reproducible way to:
- Launch llama.cpp models with pre-configured settings
- Experiment with different models, context sizes, batch sizes, and optimization flags
- Save working configurations for future reference
- Support multiple GPU backends via containerized environments

## Prerequisites

### Ubuntu + Distrobox

This setup uses [distrobox](https://distrobox.it/) to run containerized GPU toolchains. This is necessary on Ubuntu because the containerized environments from [kyuz0/amd-strix-halo-toolboxes](https://github.com/kyuz0/amd-strix-halo-toolboxes) don't work natively with Ubuntu's default container setup. See [this issue](https://github.com/kyuz0/amd-strix-halo-toolboxes/issues/16) for details.

**Install distrobox:**
```bash
curl -s https://raw.githubusercontent.com/89luca89/distrobox/main/install | sudo sh
```

### Container Images

This project uses the amazing [kyuz0/amd-strix-halo-toolboxes](https://github.com/kyuz0/amd-strix-halo-toolboxes) which provides pre-built containerized environments for different GPU backends:

- **ROCm 7.2**: `docker.io/kyuz0/amd-strix-halo-toolboxes:rocm-7.2`
- **Vulkan RADV**: `docker.io/kyuz0/amd-strix-halo-toolboxes:vulkan-radv`

These containers handle all GPU driver setup automatically.

## Quick Start

### Launch a Model

```bash
# ROCm backend
rocm-llama qwen3-coder-30b --port 8000

# Vulkan backend
vulkan-llama devstral-small-24b --port 8080

# View available models
rocm-llama
```

### Access the Server

Once running, the server is available at:
- Localhost: `http://localhost:8000`
- Network: `http://<your-machine-ip>:8000`

## Configuration

Configurations are stored in `./configs/` as simple shell scripts. Each config defines:

```bash
MODEL_NAME="Qwen3-Coder-30B"
MODEL_PATH="/path/to/model.gguf"
CONTEXT_SIZE=262144
BATCH_SIZE=2048
UBATCH_SIZE=2048
THREADS=8
GPU_LAYERS=999
FLASH_ATTN=1
JINJA=1
```

### Environment Variables

Override the config directory location:

```bash
# Use custom config directory
LLAMA_CONFIG_DIR=~/.config/model-configs rocm-llama qwen3-coder-30b --port 8000

# Or set permanently
export LLAMA_CONFIG_DIR=~/.config/model-configs
```

## Script Architecture

The project uses a clean separation of concerns:

### Host-side (Entry Points)

- **`rocm-llama`** - Simple wrapper → `llama-server rocm`
- **`vulkan-llama`** - Simple wrapper → `llama-server vulkan`
- **`llama-server`** - Master orchestrator that:
  - Creates/manages containers if needed
  - Copies configs and wrapper scripts into containers
  - Handles backend selection and configuration loading

### Container-side (Server Startup)

- **`llama-server-container`** - Universal server launcher that:
  - Sources the model config
  - Builds the llama-server command with proper flags
  - Executes the server with all configured parameters

## Adding New Models

1. Create a new config file in `./configs/`:

```bash
cat > ./configs/my-model.conf << 'EOF'
MODEL_NAME="My Model"
MODEL_PATH="/path/to/model.gguf"
CONTEXT_SIZE=4096
BATCH_SIZE=2048
UBATCH_SIZE=2048
THREADS=8
GPU_LAYERS=999
FLASH_ATTN=1
JINJA=1
EOF
```

2. Launch it:

```bash
rocm-llama my-model --port 8000
```

## Available Configs

Current models configured and tested:

- **qwen3.6-35b-a3b** - Qwen3.6-35B-A3B (38.5GB, UD-Q8_K_XL, MoE 3B active, default)
- **qwen3-coder-next** - Qwen3-Coder-Next (86GB, UD-Q8_K_XL, MoE 3B active)
- **qwen3-coder-30b** - Qwen3-Coder-30B (34GB, Q8_K_XL, OpenCode-compatible)
- **devstral-small-24b** - Devstral-Small-2-24B (28GB, Q8_K_XL)
- **gpt-oss-120b** - GPT-OSS-120B (61GB, F16)
- **nomic-embed-v1.5** - nomic-embed-text-v1.5 (140MB, Q8_0, 768-dim embeddings, port 8081; disabled by default)
- **default** - Fallback configuration

## Embeddings

Set `EMBEDDINGS=1` in any config to enable the `/v1/embeddings` endpoint alongside `/v1/chat/completions` on the same server. Optionally set `POOLING` (e.g. `mean`, `last`, `cls`) to override llama.cpp's default. `PORT` picks a non-default port for dedicated embedding-only models.

## Configuration Options

| Option | Example | Description |
|--------|---------|-------------|
| `MODEL_NAME` | "Qwen3-Coder-30B" | Display name for logging |
| `MODEL_PATH` | "/path/to/model.gguf" | Path to GGUF model file |
| `CONTEXT_SIZE` | 262144 | Token context window |
| `BATCH_SIZE` | 2048 | Batch size for prompt processing |
| `UBATCH_SIZE` | 2048 | Unbatch size for KV processing |
| `THREADS` | 8 | CPU threads for offloaded layers |
| `GPU_LAYERS` | 999 | Layers to keep in VRAM (999 = all) |
| `FLASH_ATTN` | 1 | Enable flash attention |
| `JINJA` | 1 | Enable jinja template processing for tools |
| `NO_MMAP` | 1 | Disable memory mapping (recommended for Strix Halo) |
| `CACHE_TYPE_K` | q8_0 | KV cache key quantization type |
| `CACHE_TYPE_V` | q8_0 | KV cache value quantization type |
| `CACHE_REUSE` | 4096 | Prompt cache reuse size |
| `KV_UNIFIED` | 1 | Unified KV cache (for unified memory systems) |
| `SPEC_TYPE` | ngram-mod | Speculative decoding type |
| `SPEC_NGRAM_SIZE_N` | 10 | N-gram size for speculative decoding |
| `DRAFT_MIN` | 12 | Minimum draft tokens |
| `DRAFT_MAX` | 24 | Maximum draft tokens |
| `CHAT_TEMPLATE_FILE` | (optional) | Custom jinja template for tools |

## Systemd Service

A parameterized systemd user service is provided for auto-starting models on boot:

```bash
# Start a model
systemctl --user start llama-server@qwen3.6-35b-a3b

# Enable on boot
systemctl --user enable llama-server@qwen3.6-35b-a3b

# Switch default model
systemctl --user disable llama-server@qwen3.6-35b-a3b
systemctl --user enable llama-server@qwen3-coder-next

# Check status
systemctl --user status llama-server@qwen3.6-35b-a3b
```

The instance name after `@` matches the config filename (without `.conf`).

## Additional Arguments

Pass extra arguments directly to llama-server:

```bash
rocm-llama qwen3-coder-30b --port 8000 --n-predict 500 --threads-batch 16
```

The following are automatically set:
- `--host 0.0.0.0` (exposes on network by default)

## Troubleshooting

### Container not found

The first run will automatically create the container with the correct image. Subsequent runs will use the existing container.

### Model file not found

Ensure the `MODEL_PATH` in your config file exists and is accessible to the container. Paths should be absolute.

### Config directory issues

Check the config directory path:

```bash
# Show current config directory in use
rocm-llama

# Use custom directory
LLAMA_CONFIG_DIR=/path/to/configs rocm-llama my-model --port 8000
```

### Tool calls not working

Ensure `JINJA=1` is set in the model config. This enables the jinja template engine required for tool/function calling in OpenCode and other tools.

## References

- [llama.cpp](https://github.com/ggml-org/llama.cpp) - The amazing inference engine
- [kyuz0/amd-strix-halo-toolboxes](https://github.com/kyuz0/amd-strix-halo-toolboxes) - Containerized GPU toolchains
- [distrobox](https://distrobox.it/) - Container wrapper for easy integration
- [OpenCode](https://opencode.ai/) - AI-powered coding assistant

## License

This project is a configuration and tooling layer. Use in accordance with the licenses of the underlying projects (llama.cpp, amd-strix-halo-toolboxes, etc.).
