# IFM models

Source: https://docs.ifm.ai/#/model-catalog

## Catalog

| Series | Model | Context | Languages | Hosted API |
| --- | --- | --- | --- | --- |
| K2 Horizon | `IFM/K2-Horizon-375B-A23B` | 512K | English, Arabic, 30+ more | Yes |
| K2 Horizon | `IFM/K2-Horizon-MoVA-36B-A4B` | 512K | English, Arabic, 30+ more | No |
| K2 Horizon | `IFM/K2-Horizon-32B` | 512K | English, Arabic, 30+ more | No |
| K2 Horizon | `IFM/K2-Horizon-7B` | 512K | English, Arabic, 30+ more | No |
| K2 Horizon | `IFM/K2-Horizon-3.7B` | 512K | English, Arabic, 30+ more | No |
| K2 Horizon | `IFM/K2-Horizon-0.9B` | 128K | English, Arabic, 30+ more | No |
| K2 V2 | `IFM/K2-Think-V2` | 128K | English | Deprecating |
| K2 V2 | `IFM/K2-V2-Instruct` | 128K | English | No |
| K2 V2 | `IFM/K2-V2` | 128K | English | No |
| Jais 2 | `Jais-2-70B-Chat` | 8K | Arabic, English | No |
| Jais 2 | `Jais-2-8B-Chat` | 8K | Arabic, English | No |
| ASR | `IFM/Jais-ASR` | — | Arabic (incl. dialects), English, auto-detect | Yes (`/v1/audio/transcriptions`) |

## K2 Horizon

IFM's frontier reasoning series, built for agentic tool use and long-horizon tasks. Six open-weight sizes; the 375B sparse MoE is served on the hosted API. Collection: https://huggingface.co/collections/IFM/k2-horizon

| Model | Architecture | Context | Best for | API |
| --- | --- | --- | --- | --- |
| IFM/K2-Horizon-375B-A23B | Sparse MoE | 512K | Frontier reasoning and long-horizon agents | ✓ |
| IFM/K2-Horizon-MoVA-36B-A4B | MoVA + MoE | 512K | Production-style serving experiments | — |
| IFM/K2-Horizon-32B | Dense decoder-only | 512K | Long-context and reasoning experiments | — |
| IFM/K2-Horizon-7B | Dense decoder-only | 512K | Fine-tuning and cost-conscious deployment | — |
| IFM/K2-Horizon-3.7B | Dense decoder-only | 512K | Efficient research and single-node serving | — |
| IFM/K2-Horizon-0.9B | Dense decoder-only | 128K | Local development, evaluation dry runs | — |

Weights: `https://huggingface.co/<model id>`.

## K2 V2

Built for research, fine-tuning, and long-context work. Collection: https://huggingface.co/collections/IFM/k2-v2

| Model | Type | Params | Context | API |
| --- | --- | --- | --- | --- |
| K2-Think-V2 | Reasoning | 70B | 128K | Deprecating |
| K2-V2-Instruct | SFT / aligned | 70B | 128K | — |
| K2-V2 | Base (pretrained) | 70B | 128K | — |

Intended use: LLM/reasoning research, downstream fine-tuning (instruction following, agents, domain models), long-context architecture experiments, reproducible scaling benchmarks. For aligned conversational use and any production traffic, use K2 Horizon (see [migration.md](migration.md)).

## Jais 2

Second-generation Jais — Arabic-centric bilingual chat models by MBZUAI, Cerebras, and Inception, trained from scratch on 600B+ curated Arabic tokens. Custom 150K-token Arabic-centric vocabulary, RoPE, ReLU² activations. Collection: https://huggingface.co/collections/inceptionai/jais-2-family

| Model | Architecture | Active params | Context | Weights |
| --- | --- | --- | --- | --- |
| Jais-2-70B-Chat | Dense decoder-only | 70B | 8K | https://huggingface.co/inceptionai/Jais-2-70B-Chat |
| Jais-2-8B-Chat | Dense decoder-only | 8B | 8K | https://huggingface.co/inceptionai/Jais-2-8B-Chat |

## Choosing

- Hosted production chat/agents → `IFM/K2-Horizon-375B-A23B`.
- Self-hosting / fine-tuning with long context → a smaller K2 Horizon size (7B for cost-conscious, 0.9B for local dev).
- Arabic-centric chat, self-hosted, short context → Jais 2.
- Speech-to-text → `IFM/Jais-ASR`.
