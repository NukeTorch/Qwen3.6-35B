# NukeTorch - Run Qwen 3.6 35B (4-bit) at 511.1 tok/s on Apple Silicon

511.1 tok/s: NVIDIA SPEED-Bench coding prompts on NukeTorch 1.0.5. The tables below are the four-engine comparison from the benchmark report.

C++ and Metal Tensor Engine serves text and image for inference and training.

**[Video demo](https://youtu.be/tj_VcOJ0VKk)** · **[Detailed benchmark report](benchmark/qwen36_35b_four_engines_M5.pdf)**

### Quickstart

Install with `curl`, not a browser.

```sh
curl -fsSL https://github.com/NukeTorch/nuketorch/releases/download/1.0.5/install.sh | sh
```

Get the model and its draft model from Hugging Face ([model card](https://huggingface.co/NukeTorch/Qwen3.6-35B-A3B-4bit-DFlash2)):

The first line gives you the `hf` command (or: `brew install hf`).

```sh
python3 -m venv ~/NukeTorch/hf-env && ~/NukeTorch/hf-env/bin/pip install -U huggingface_hub
~/NukeTorch/hf-env/bin/hf download NukeTorch/Qwen3.6-35B-A3B-4bit-DFlash2 --local-dir ~/NukeTorch/models/Qwen3.6-35B-A3B-4bit-DFlash2
```

Run it:

```sh
cd ~/NukeTorch/nuketorch-1.0.5-macos-arm64
./nuketorch-serve http \
  --model ~/NukeTorch/models/Qwen3.6-35B-A3B-4bit-DFlash2/Qwen3.6-35B-A3B-4bit \
  --draft-model ~/NukeTorch/models/Qwen3.6-35B-A3B-4bit-DFlash2/Qwen3.6-35B-A3B-DFlash2 \
  --port 8080
```

The server prints `READY` when it can answer. Then:

```sh
curl http://127.0.0.1:8080/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"messages":[{"role":"user","content":"Write a Python function that checks whether a string is a palindrome."}]}'
```

### Benchmark Summary

At one request at a time, NukeTorch decodes 482.7 to 597.5 tok/s on prompts A to D. With 8 users at once NukeTorch generates 821.8 tok/s in total.

**Table 1. Speed, RAM and energy**

| | NukeTorch 1.0.5 | mlx-lm<br>0.32.2 | llama.cpp<br>1.0.0 build 12359 | oMLX<br>0.8.0rc1 Splash + DFlash2 |
|---|---:|---:|---:|---:|
| **Speed (tok/s)** | | | | |
| Prompt A (5 coding prompts) | **517.5** | 126.8 | 107.5 | 387.3 |
| Prompt B (count to 100) | **597.5** | 127.8 | 108.3 | 505.4 |
| Prompt C (short story) | **482.7** | 125.0 | 106.7 | 368.0 |
| Prompt D (coding-15) | **520.4** | 120.0 | 105.7 | 377.7 |
| 2 users at once | **533.6** | 120.2 | 162.4 | 524.0 |
| 4 users at once | **679.3** | 120.0 | 234.4 | 661.5 |
| 8 users at once | **821.8** | 120.4 | \*N/A | \*N/A |
| **RAM (peak increase, GB)** | | | | |
| Prompt A | **24.77** | 20.74 | 31.48 | 24.56 |
| Prompt B | **24.75** | 20.95 | 31.84 | 24.51 |
| Prompt C | **24.70** | 21.26 | 32.01 | 25.60 |
| Prompt D | **24.69** | 21.40 | 32.44 | 25.67 |
| 2 users at once | **28.09** | 20.58 | 31.10 | 23.81 |
| 4 users at once | **27.67** | 20.54 | 31.36 | 24.10 |
| 8 users at once | **27.69** | 21.22 | \*N/A | \*N/A |
| **Power (tok/J, tokens per joule)** | | | | |
| Prompt A | **8.03** | 4.02 | 2.82 | 7.27 |
| Prompt B | **9.42** | 4.19 | 2.91 | 10.04 |
| Prompt C | **7.62** | 4.03 | 2.80 | 7.22 |
| Prompt D | **8.07** | 3.92 | 2.85 | 7.09 |
| 2 users at once | **8.62** | 3.80 | 3.55 | 9.22 |
| 4 users at once | **9.63** | 3.79 | 4.69 | 13.23 |
| 8 users at once | **11.27** | 3.90 | \*N/A | \*N/A |

\*N/A: not supported at the engine's default settings (it serves 4 requests at once).

**Table 2. NukeTorch against the other engines** (the range over every row of Table 1 where both engines have a number)

| NukeTorch 1.0.5 vs | mlx-lm | llama.cpp | oMLX |
|---|---:|---:|---:|
| Inference speed | 3.9× to 6.8× faster | 2.9× to 5.5× faster | 1.0× to 1.4× faster |
| RAM | 15% to 36% more | 10% to 24% less | 4% less to 18% more |
| Power saving (energy per token) | 47% to 65% saving | 51% to 69% saving | 37% more power to 12% saving |

**How it runs.**

- **Prompts.** A: the five NukeTorch demo coding prompts, 3 passes (15 requests). B: count to 100, 3 requests. C: short story, 3 requests. D: one everyday coding prompt (coding-15), 3 requests. Prompts A to D are one request at a time, capped at 8,192 output tokens.
- **Users at once.** The 16 coding prompts in rounds of 2, 4 or 8 requests until every prompt has been sent, 32 requests per level, 1,024 output tokens per request. NukeTorch's users runs used a server started with `--users 8`.
- **Models.** The mlx-community 4-bit model for NukeTorch, mlx-lm and oMLX; the ggml-org Q4_K_M GGUF for llama.cpp. Server default sampling for every engine.
- **Run-to-run spread.** The tables show one run of each test; every run is kept in the report's records. NukeTorch runs below a GPU-clock floor were repeated; the other engines' runs were not checked for it. Details in the report. Let the Mac rest before you measure.

Qwen3.6-35B-A3B 4-bit on one Apple M5 Max (18 cores, 48 GB, macOS 26.7.2). Every engine runs as its maker ships it: latest release, maker's install, maker's start command, default settings, except that NukeTorch's users rows used `--users 8`. mlx-lm and llama.cpp ran without a draft model. oMLX runs with its Splash engine turned on, as oMLX's documentation describes, with the DFlash2 draft model. NukeTorch runs with its DFlash2 draft model and typical acceptance. tok/s is decode speed. RAM is the increase in the whole Mac's memory used. tok/J is tokens per joule of chip power (CPU + GPU + ANE), not wall power.

The full report has power per test, the method, versions and the Mac's conditions: [benchmark/qwen36_35b_four_engines_M5.pdf](benchmark/qwen36_35b_four_engines_M5.pdf).

NukeTorch Enterprise runs one model across machines by pipeline parallelism. It is by invitation. Ask for an evaluation license:

[![Request a NukeTorch Enterprise evaluation license](https://img.shields.io/badge/NukeTorch%20Enterprise-Request%20an%20evaluation%20license-121F47?style=for-the-badge)](mailto:contact@nuketorch.ai?subject=NukeTorch%20Enterprise%20evaluation%20request&body=Name%3A%0AOrganization%3A%0AMacs%20%28how%20many%2C%20which%20chips%2C%20memory%29%3A%0AModel%20or%20workload%3A%0A)

### How we did it

**Unfolding.** A tensor's values are fixed; how they are stored and read is a choice, and changing that choice is unfolding. On one Apple M4 Max in our own runtime, expanding a four-bit model's weights to 16 bits used 2.4× the memory for a 1.14× speedup, while unfolding the same codes in the same bytes gave 1.27×. A control showed that most of that gain came from the form of the kernel's loop, not the order, and that the weight bytes a computation reads set a floor that no order lowers; three large models already decode at 81 to 90% of it in that runtime. Paper: [Unfolding: Theory and Principles](https://nuketorch.ai/papers/unfolding-theory-and-principles.pdf).

**Shaping.** A matrix product has three axes (rows, output width, reduction length) and an order in time, yet devices are mostly judged by one-axis indicators such as busy time. The paper measured the axes one at a time on Apple's Neural Engine, on an M4 Max and an M5 Max. Splitting a long reduction length into slices raised a long-reduction layer pair 3.40× and 3.12× there. Paper: [Understanding the Shapes of Neural Simulation](https://nuketorch.ai/papers/understanding-the-shapes-of-neural-simulation.pdf).

The papers report their own measurements. They do not describe NukeTorch's code and do not explain the speeds in the tables above.

### Minimum hardware requirement

We recommend an M5 Max with 48 GB of RAM. We optimized for the Neural Accelerator of Apple silicon. This release runs on the M5 Max and the M4 Max; support for M3 series chips is coming. On an M4 Max with 128 GB it gave 388.7 tok/s on the SPEED-Bench coding prompts. The model needs about 27 GB of free memory. It needs macOS 27.2 or later.

### More about running it

- `GET /v1/models` shows the model name your client should use.
- Several users: add `--users N` (1 to 8). The default is 1, which is the fastest for one person. Each extra user costs about 1.4 GB (36.5 GB with `--users 8`). A server started for several users is slower for one user than one started with `--users 1`.
- Images: send them inline as an `image_url` part holding a `data:` address (PNG, JPEG or WebP), 1 to 8 per request. Images given by web address are refused.
- Streaming and function tools (`tool_choice` `auto` or `none`) are supported. Forced tool choice, strict schemas and `response_format` are not.
- Options: `--host`, `--port`, `--max-context` (default 32768), `--max-memory GB`, `--prefix-cache-gb`, `--users`.

### Reproduce our numbers

This repeats NukeTorch's speed on our prompts; Table 1 itself, with the other engines, is the `materials` kit.

`bench` runs the model's standard suite; `report` shows your run beside ours.

```sh
cd ~/NukeTorch/nuketorch-1.0.5-macos-arm64
./nuketorch-serve bench ~/NukeTorch/models/Qwen3.6-35B-A3B-4bit-DFlash2
./nuketorch-serve report
```

The SPEED-Bench coding prompts are NVIDIA's and are not in the package. This command downloads NVIDIA's own file from Hugging Face (`nvidia/SPEED-Bench`) at a pinned revision, checks it against hashes stored in the program, then runs that suite:

```sh
./nuketorch-serve bench --fetch speed-bench
./nuketorch-serve bench ~/NukeTorch/models/Qwen3.6-35B-A3B-4bit-DFlash2 --suite speed-bench
```

Other commands: `bench --list` (what each test does), `bench --inputs` (every stored input with its license and source), `bench MODEL --suite quick` (a check in minutes; no golden to compare), `selftest` (the model kernels, about 1 second, no model needed), `materials DIR` (the scripts and report of our comparison with other engines).

Before you measure: turn off Spotlight indexing (`sudo mdutil -a -i off`; turn it back on with `sudo mdutil -a -i on`), turn off Siri, on a MacBook Pro plug in power and choose High Power, close other apps, and let the Mac cool down: rest it 5 minutes before a run and 5 minutes between tests run back to back.

### License

NukeTorch is free for personal use, schools and universities, and businesses and other organizations (not government entities or international organizations) that with their affiliates have less than US$10 million in revenue and fewer than 250 people, on up to 250 machines, each working alone. Everyone else may evaluate it for 30 days, then needs a commercial license: contact@nuketorch.ai. You may not offer its function itself to others: access to its API, or running, serving or fine-tuning models for them. Terms are in [LICENSE.md](https://github.com/NukeTorch/nuketorch/blob/1.0.5/LICENSE.md). The model and its draft model are Apache-2.0, by their makers.

**Powered by NukeTorch** · https://nuketorch.ai · Back to [NukeTorch/nuketorch](https://github.com/NukeTorch/nuketorch)
