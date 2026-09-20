# Qwen3.8 Flash Next on Strix Halo 128GB

Complete setup for Ryzen AI Max+ 395, 128 GB unified memory on Linux. Needs ~130 GB free disk space and sudo access.

- Server Runtime: [`halo-box/strix-llama.cpp`](https://github.com/halo-box/strix-llama.cpp) (HIP build). 
  Read more [here](https://pwilkin.github.io/strix-halo/) ("the community fork").
- Model: [`ilintar/qwen3.8-flash-next-gguf-strix-halo`](https://huggingface.co/ilintar/qwen3.8-flash-next-gguf-strix-halo) (93 GiB IQ4_NL, 9 shards + MTP draft).
- Expected Speed: 1k t/s prompt processing and 20-60 t/s token generation.
- Default distro: Ubuntu 26.04. LTS (hints for other distros included)

## 0. Directories & Basic Tools

Chose where to store model and server runtime.

```bash
export MODEL_DIR=~/workspace/models/Qwen3.8-Flash-Next # wherever you store the GGUFs for Qwen3.8 Flash Next
export RUNTIME_DIR=~/workspace/engines                 # parent dir for the server runtime
mkdir -p "$MODEL_DIR" "$RUNTIME_DIR"
```

Install some basic tools.

```bash
sudo apt update && sudo apt install -y vim curl wget
```

Fedora/RHEL: `sudo dnf install -y vim curl wget`. Arch: `sudo pacman -S --needed vim curl wget`


## 1. BIOS: UMA/VRAM Setting

In BIOS, the option is usually called "UMA Frame Buffer Size" or "VRAM
Allocation". Set it to **auto** or the **smallest possible value**
(e.g. 512 MB). The model lives in shared system memory accessed through GTT (step 2). 

If your BIOS has no such option, skip this.

## 2. Kernel Parameters

Set three parameters via the bootloader:

| param | why |
|---|---|
| `amd_iommu=off` | disables IOMMU remapping; avoids DMA translation overhead/quirks of the iGPU with very large buffers under ROCm |
| `amdgpu.gttsize=112640` | grows the GTT (the GPU's window into system RAM) to 110 GB, so 93 GB weights + KV cache + compute buffers fit (formula: `<GTT in GB> * 1024`) |
| `ttm.pages_limit=28835840` | raises TTM's allocation page cap to match 110 GB at 4 KB pages (formula: `<amdgpu.gttsize> * 1024 * 1024 / 4096`)  |

This gives enough space for model weights and context while still leaving room for other tasks so that the halo box may still be usable as main machine (depending on the task) - not only as dedicated inference machine.

Ubuntu: edit `/etc/default/grub` (requires sudo), e.g.

```bash
sudo vi /etc/default/grub
```
uncomment (if need be) and set the default cmdline as follows:

```GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_iommu=off amdgpu.gttsize=112640 ttm.pages_limit=28835840"```

then save, update grub and reboot:

```bash
sudo update-grub && sudo reboot
```

Other distros: edit the same file, then `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`
(Fedora/RHEL) or `sudo grub-mkconfig -o /boot/grub/grub.cfg` (Arch); systemd-boot:
append the three params to your kernel command line entry instead.

After reboot, verify:

```bash
cat /proc/cmdline
sudo cat /sys/module/amdgpu/parameters/gttsize
```

## 3. User Permissions for GPU Access

```bash
sudo usermod -a -G render,video $(whoami) && exec sudo -u "$(whoami)" -i 
```

## 4. ROCm

Install the current ROCm SDK per the official docs <https://rocm.docs.amd.com/en/latest/>, select

- Device Family: AMD Ryzen
- Use Case: Compute
- Ryzen APU: AMD Ryzen AI Max+ PRO 395

The single SDK package `amdrocm-core-sdk<version>-gfx1151` covers everything needed. Adjust the version number to whatever the latest version is, e.g.

```bash
sudo apt update && sudo apt install -y amdrocm-core-sdk10.0-gfx1151
```

Ubuntu 26.04. LTS is a supported ROCm target. But the same package family exists for RHEL/Fedora.

Verify:

```bash
rocminfo | grep -q gfx1151 && echo gfx1151-ok
hipconfig --version
```

## 5. Build Tools

Ubuntu:

```bash
sudo apt install -y build-essential cmake ninja-build git pkg-config
```

Fedora/RHEL: `sudo dnf install -y gcc gcc-c++ make cmake ninja-build git pkgconf-pkg-config`
Arch: `sudo pacman -S --needed base-devel cmake ninja git`

## 6. GPU monitor (optional)

Useful live insights into GPU / VRAM usage, see <https://github.com/Umio-Yasuno/amdgpu_top>.

### 1. Prerequisite `cargo` (Rust)

Make sure `cargo` is installed.

Ubuntu:
```bash
sudo apt update && sudo apt install -y cargo pkg-config libdrm-dev
```

Other distros:
```bash
# any distro
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Make sure `~/.cargo/bin` is in `PATH`.

```bash
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc
```

### 2. Install `amdgpu_top`

```bash
cargo install amdgpu_top --locked
```

### 3. Run

```bash
amdgpu_top
```

Dark mode:
```bash
amdgpu_top --dark
```
During model load, you'll see GTT usage climb.

## 7. Build Runtime

```bash
cd "${RUNTIME_DIR}"
git clone https://github.com/halo-box/strix-llama.cpp.git
cd strix-llama.cpp
export HIPCXX="$(hipconfig -l)/clang"
export HIP_PATH="$(hipconfig -R)"
cmake -B build -DGGML_HIP=ON -DGPU_TARGETS=gfx1151 -DCMAKE_BUILD_TYPE=Release
cmake --build build -j --target llama-server llama-bench
```

## 8. Model

Repo: <https://huggingface.co/ilintar/qwen3.8-flash-next-gguf-strix-halo>

Download manually into $MODEL_DIR or use hugginface `hf` client.

## 9. Runtime Configuration

Writes a tailored models-preset `llama.ini` into the model dir:

```bash
cat > "$MODEL_DIR/llama.ini" <<EOF
[Qwen3.8-Flash-Next]
load-on-startup = true
model = ${MODEL_DIR}/Qwen3.8-Flash-Next-IQ4_NL-PROJFIX-00001-of-00009.gguf
device = ROCm0
n-gpu-layers = 999
flash-attn = on
fit = off
# the next two lines are crucial for prompt processing
load-mode = none
lazy-mode = on-direct
cache-type-k = f16
cache-type-v = f16
ctx-size = 262144
batch-size = 16384
ubatch-size = 16384
cache-prompt = on
ctx-checkpoints = 128
# agentic coding
reasoning = on
reasoning-format = deepseek
reasoning-effort = xhigh
temp = 0.6
top-p = 0.95
top-k = 20
min-p = 0.05
presence-penalty = 0.0
repeat-penalty = 1.0
metrics = on
parallel = 1
jinja = true
# draft model
spec-type = draft-mtp
spec-draft-model = ${MODEL_DIR}/mtp-Qwen3.8-Flash-Next-shared-Q8_0.gguf
spec-draft-device = ROCm0
gpu-layers-draft = 999
# increase speculation dynamically when predicted acceptance probability >= 0.75   
spec-draft-adaptive = on
spec-draft-n-min = 0
spec-draft-n-max = 7
spec-draft-p-min = 0.75
spec-draft-type-k = f16
spec-draft-type-v = f16

# vision (optional)
# mmproj = <path to mmproj model>

EOF
```

## 10. Run Server

```bash
cd ${RUNTIME_DIR}/strix-llama.cpp
export GGML_HIP_ENABLE_UNIFIED_MEMORY=1
./build/bin/llama-server --models-preset "${MODEL_DIR}/llama.ini"
```

**Parallel requests:** if you raise `parallel` above 1 in the models-preset (or
pass `-np`), also set `HIP_LAUNCH_BLOCKING=1`. At `parallel = 1` it is not needed.

Model load takes a few minutes. Runtime is ready when it prints the endpoint on the console.

## Troubleshooting

- `/dev/kfd: Permission denied` — make sure user is in groups `render` and `video` 
(for docker: GIDs must match host GIDs and devices `/dev/kfd` and `/dev/dri` must be mapped into the container.)
- **Model loading fails with out-of-memory** (despite 128 GB) — set kernel params (step 2), check `/proc/cmdline` and `/sys/module/amdgpu/parameters/gttsize`.
- **Garbled output only with `parallel > 1`** — set `HIP_LAUNCH_BLOCKING=1` before running the server.
- **Error loading shared libraries** — set `LD_LIBRARY_PATH=/opt/rocm/lib:$LD_LIBRARY_PATH` before running the server.

