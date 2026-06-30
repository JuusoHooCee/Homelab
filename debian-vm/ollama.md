# Ollama Setup

Ollama runs natively on the Debian VM (not in Docker) — see `debian-vm/setup.md` for the prerequisite NVIDIA driver setup. Running Ollama natively means it uses the host's CUDA libraries and driver directly, with no need for the NVIDIA Container Toolkit.

## Installation

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

## Getting a newer version if needed

The version installed by `install.sh` may lack certain newer features (e.g. `ollama search`, `QUANTIZE` support in Modelfiles — see the kvantisointi section below). Direct "latest release" binary download links can return "Not Found" depending on network/redirect handling. A more reliable method is querying the GitHub API directly for the actual asset URL:

```bash
curl -L $(curl -s https://api.github.com/repos/ollama/ollama/releases/latest | grep browser_download_url | grep linux-amd64 | cut -d '"' -f 4) -o ollama
```

Caution: if a release has multiple `linux-amd64` assets (binary, `.tgz`, `.deb`), this can grab the wrong one. Check the file size before using it — a valid binary should not be hundreds of megabytes larger than expected for no reason, and running a wrong-format file will fail with `Exec format error`.

The more reliable approach is downloading the explicit `.tgz` archive by version tag:

```bash
wget https://github.com/ollama/ollama/releases/download/v0.3.14/ollama-linux-amd64.tgz
tar -xvf ollama-linux-amd64.tgz
```

This extracts directly into the current directory (not into a subfolder):

```
./bin/ollama          # the binary
./lib/ollama/*.so     # bundled CUDA libraries (libcublas, libcudart, libcublasLt)
```

### Manual installation from the extracted archive

```bash
mkdir -p ~/ollama-install
mv bin lib ~/ollama-install/
cd ~/ollama-install
sudo cp bin/ollama /usr/local/bin/
sudo chmod +x /usr/local/bin/ollama
sudo mkdir -p /usr/local/lib/ollama
sudo cp lib/ollama/* /usr/local/lib/ollama/
```

Create the systemd unit manually at `/etc/systemd/system/ollama.service`:

```ini
[Unit]
Description=Ollama Service
After=network.target

[Service]
ExecStart=/usr/local/bin/ollama serve
Restart=always
Environment=OLLAMA_HOME=/usr/local/lib/ollama

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable ollama
sudo systemctl start ollama
```

Verify: `ollama --version`

## Making Ollama reachable from other machines/containers

By default, Ollama binds to `127.0.0.1:11434` only — fine for local use, but it blocks anything external (e.g. Open WebUI running in Docker, even on the same machine) from reaching it.

**Does not work:** passing `--host 0.0.0.0` as a flag to `ollama serve` in the systemd unit's `ExecStart`. This causes Ollama to fail to start (exits with status 1) — `--host` is not a valid flag in this context.

**Correct fix:** set the `OLLAMA_HOST` environment variable via a systemd override:

```bash
sudo systemctl edit ollama
```

In the override file that opens:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

Verify:

```bash
ss -tulpn | grep 11434
```

Should show the wildcard address or the VM's actual IP, not just `127.0.0.1`.

If the override doesn't seem to take effect even after reloading and restarting, remove any previous override attempts cleanly first (`rm -r /etc/systemd/system/ollama.service.d`), reload, restart, confirm Ollama is back to its normal localhost-only behavior, then recreate the override from scratch with `systemctl edit ollama`. A leftover broken override (e.g. from an earlier failed attempt at the `--host` flag approach) can prevent a later, correct override from being applied cleanly.

## Quantization status (unresolved)

The `QUANTIZE` directive in a Modelfile (used to create a quantized variant of a model, e.g. `QUANTIZE q4_K_M`) does not work on the Ollama Linux builds used in this project, even after upgrading to a newer release via the method above. The error is:

```
Error: command must be one of "from", "license", "template", "system", "adapter", "parameter", or "message"
```

This means only models that Ollama provides already pre-quantized (like the default `llama3:8b`, which ships at Q4_0) could actually be run. Plans to run larger models (Phi-3 Medium, Gemma2 9B, Qwen2.5 7B at a different quantization level) via custom Modelfiles were not completed.

A possible path forward, not yet implemented or tested: using an `llama.cpp` backend directly (either standalone or as an alternate backend for Open WebUI) to load arbitrary GGUF files, bypassing Ollama's Modelfile quantization limitation entirely.

### Models confirmed to exist / not exist in the Ollama library (as of this project)

Exist: `llama3` (8B), `llama3:70b`, `qwen2.5:7b`, `qwen2.5:72b`, `mistral` (7B), `phi3:medium` (14B), `gemma2:9b`.

Do not exist (pull fails with "pull model manifest: file does not exist"): `llama3:13b`, `qwen2.5:14b`, any `qwen3:*` tag. Always check the official library at ollama.com/library rather than assuming a model size variant exists.

## GPU compute backend and performance notes

Ollama's logs on this GPU show:

```
level=WARN source=cuda_compat.go:38 msg="NVIDIA driver too old" device="NVIDIA GeForce GTX 1080" compute=6.1
level=INFO source=types.go:32 msg="inference compute" id=0 filter_id=0 library=Vulkan compute=0.0 ...
```

Ollama falls back to a Vulkan compute backend rather than CUDA, due to its own internal compatibility check considering the driver "too old" for CUDA — despite `nvidia-smi` itself reporting CUDA 12.2 working fine. This is Ollama's own gate, not an actual CUDA failure, and doesn't prevent GPU acceleration from working.

Note also: Ollama on Linux never prints an explicit "using CUDA backend" message regardless of which backend it's actually using (unlike Windows/macOS) — its absence in `--verbose` output is not a sign that the GPU isn't being used.

### Interpreting `nvidia-smi` output during inference

Observed while running `llama3:8b`:
- VRAM usage peaked at ~4.8GB / 8GB and did not increase under sustained load
- GPU utilization reached 100%
- Power draw reached up to ~200W (near the card's cap)

This is normal, not a problem. VRAM usage is determined by model size, quantization, and context length — not by how hard the GPU is working. A small model won't use more VRAM just because it's under heavy load. The GTX 1080 (Pascal architecture) has no Tensor Cores and no native acceleration suited to modern LLM inference kernels, so it has to work harder (higher utilization, higher power draw) per token than a newer GPU architecture would for the same model — this is expected for this hardware generation, not a misconfiguration.

Performance figures from `ollama run llama3:8b --verbose`:
- eval rate: ~43 tokens/s
- prompt eval rate: ~191 tokens/s

This eval rate is consistent with the GPU genuinely contributing to inference (a CPU-only fallback would be noticeably slower).

### Practical VRAM guidance for this GPU (8GB)

- Llama3 8B at Ollama's default Q4_0 quantization: ~4.8GB — directly observed and confirmed working
- Realistic usable headroom on this card after driver/runtime overhead: ~7.0–7.2GB
- Models in the 9B–14B range (Gemma2 9B, Phi-3 Medium 14B) were discussed as plausible if quantization worked, but were never actually tested successfully due to the Modelfile/QUANTIZE limitation above — treat these as estimates only, not verified results.

## Useful commands

```bash
ollama list                          # list downloaded models
ollama show <model>                  # show model details
ollama rm <model>                    # remove a model
curl http://localhost:11434/api/tags # test the API locally
```
