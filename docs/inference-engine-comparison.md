# Comparing Inference Engines

An inference engine is the software layer responsible for loading a Large Language Model's (LLM) weights and managing the iterative process of generating text. It is the beating heart of this guide and powers/aids all other tooling (chat platform, model proxy, MCP server proxy, etc.) documented in the software stack.

## Conceptual Overview

To choose the right engine, it's important to understand a few core concepts:

- **VRAM (Video RAM)**: The high-speed memory located on your GPU. LLMs are massive; the more of the model that fits in VRAM, the faster the text generation. If a model is too large for VRAM, it must either be quantized or "offloaded" to slower system RAM.
- **Quantization**: The process of reducing the precision of the model's weights (e.g., from 16-bit floating point to 4-bit integers). This significantly reduces the memory footprint and increases speed with a minimal loss in accuracy. Higher bit quants yield better precision and less deviation from the unquantized model. Not all models respond to quantization in the same way: a lower bit quant for one model may outperform a higher bit quant for a different but similarly-sized model.
- **GGUF, AWQ, GPTQ, EXL2/3 etc.**: These are different formats for quantized models. `GGUF` is designed for flexibility and CPU/GPU hybrid use, while `AWQ`, `GPTQ`, and `EXL2`/`EXL3` are optimized for high-performance GPU-only inference. There are others not mentioned here.
- **Hybrid Inference (Offloading)**: The ability to split a model between the GPU and the CPU. This allows you to run models that are larger than your available VRAM, albeit at a slower speed.
- **Latency vs. Throughput**: Latency is the time it takes to generate a single token (how "fast" it feels to one user), while Throughput is the total number of tokens generated per second across all concurrent users.
- **Tensor Parallelism**: A technique used in high-end setups to increase throughput by splitting a single model's computations across multiple GPUs simultaneously.
- **KV Cache (Key-Value Cache)**: As the model generates text, it stores previous tokens' computations in VRAM to avoid re-calculating them. The larger the context window (the amount of text the model "remembers"), the more VRAM the KV cache consumes. Some models are more efficient with KV caching than others - this impacts overall VRAM usage.

As a rule of thumb, if a model (quantized or unquantized) + its KV cache fits within your system's VRAM, it will run significantly faster than if it requires offloading. This is particularly true of dense models, where all parameters are used for inference. Similarly, offloading is greatly benefitted by higher RAM speed (DDR5 > DDR4 > DDR3), as the main bottleneck is how fast the CPU and GPU can communicate with each other. Systems like the Nvidia DGX Spark, Apple Mac Mini/Studio, and AMD Strix Halo have unified memory, where the CPU and GPU are either integrated or sit on the same chip, enabling fast speeds for a large pool of RAM that is also considered the VRAM. This is a boon for users of Mixture-of-Expert (MoE) models, where only some model parameters are active during inference. Simply put,

| **Setup**          | **Fits in VRAM** | **Recommended Model Type**        | **Performance Bottleneck**  |
| :----------------- | :--------------- | :-------------------------------- | :-------------------------- |
| GPU Only           | Yes              | Dense (Full/AWQ/GPTQ) / Small MoE | GPU Compute / VRAM Capacity |
| Hybrid (GPU + CPU) | No               | Quantized Dense / MoE (GGUF)      | System RAM Bandwidth        |
| Unified Memory     | Yes (Unified)    | Large MoE / High-precision Dense  | Unified Memory Bandwidth    |

You can learn more about Mixture-of-Expert models through [this helpful HuggingFace blog post](https://huggingface.co/blog/moe).

## Comparison

### llama.cpp vs. Ollama

> [!NOTE]
> As of mid-2025, Ollama has transitioned from using the `llama.cpp` server as its direct backend to a specialized runtime built on the GGML tensor library. While it maintains compatibility with the GGUF format pioneered by `llama.cpp`, Ollama posits that this transition allows for tighter integration of multimodal capabilities and more optimized model management. See [their blog post on multimodal models](https://ollama.com/blog/multimodal-models) for details.

| **Aspect**             | **llama.cpp**                                                                                                                                                                                                        | **Ollama**                                                    |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Installation/Setup** | Requires manual setup/configuration                                                                                                                                                                                  | One-click install & CLI model management                      |
| **Model Switching**    | Native model-switching via [`--models-preset`](https://github.com/ggml-org/llama.cpp/tree/master/examples/server) flag (router mode with INI-file configs) or [llama-swap](https://github.com/mostlygeek/llama-swap) | Native model-switching via server                             |
| **Customizability**    | Full control over parameters via CLI                                                                                                                                                                                 | Limited: Modelfiles are cumbersome                            |
| **Transparency**       | Full transparency in parameter settings                                                                                                                                                                              | Defaults may override model parameters (e.g., context length) |
| **GGUF Support**       | Best GGUF implementation                                                                                                                                                                                             | GGUF support via own engine + GGML library                    |
| **Hybrid Inference**   | Supports hybrid CPU-GPU inference                                                                                                                                                                                    | Same as llama.cpp                                             |

---

### vLLM vs. Ollama/llama.cpp
| **Feature**                  | **vLLM**                                         | **Ollama/llama.cpp**                                                            |
| ---------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------- |
| **Vision Models**            | Supports vision models natively                  | Same as vLLM                                                                    |
| **Quantization**             | Supports AWQ, GPTQ, BnB, etc.                    | Only supports GGUF                                                              |
| **Multi-GPU Inference**      | Yes                                              | Yes                                                                             |
| **Tensor Parallelism**       | Yes                                              | No                                                                              |
| **Hybrid Inference**         | No                                               | Yes                                                                             |
| **New Architecture Support** | Almost immediately: vLLM is an industry-standard | Takes 1-3 months for new architectures to be integrated into existing codebases |

In summary,

- **Ollama**: Best for those who want an "it just works" experience.
- **llama.cpp**: Best for those who want total control over their inference servers and are familiar with engine arguments. The native `--models-preset` flag provides model switching without extra tools, while [llama-swap](https://github.com/mostlygeek/llama-swap) adds a web UI and more advanced routing.
- **vLLM**: Best for those who want to
  - run non-GGUF quantizations (or native safetensors weights) of models, or
  - enable efficient batched inference using tensor parallelism for multiple concurrent users, or
  - run the latest architectures as quickly as possible.

Since vLLM does not support hybrid inference, it is generally not recommended for VRAM-constrained environments. Users with less than 24GB of VRAM may find vLLM's requirements restrictive. Switching to llama.cpp enables the use of quantized models and hybrid inference, making inference possible in resource-limited environments.

Using Ollama as a service is efficient as unused models are automatically offloaded from VRAM after a period of inactivity. Similarly, llama.cpp's router mode (via `--models-preset`) allows models to be dynamically loaded and unloaded. In contrast, vLLM typically keeps models resident in memory; therefore, it is not recommended to run vLLM alongside other engines in an automated, always-on configuration unless it serves as the primary inference engine. Essentially, the optimal setups are:

1. llama.cpp with `--models-preset`, or
2. llama.cpp/vLLM with llama-swap, or
3. Ollama (standalone)

> [!IMPORTANT]
> While Ollama provides a good user experience and ease of setup, it introduces an abstraction layer that can limit transparency and fine-grained control over inference parameters. For production environments or robust solutions where reliability, maximum performance, and immediate support for new model architectures are the priority, **llama.cpp and vLLM are recommended over Ollama**. There has also been [some controversy](https://www.reddit.com/r/LocalLLaMA/comments/1qvq0xe/bashing_ollama_isnt_just_a_pleasure_its_a_duty/) as far as Ollama's use of llama.cpp's backend without proper attribution: something worth knowing if you also consider a company's historical decision-making when choosing a product.