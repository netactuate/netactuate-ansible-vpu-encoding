# NetActuate Ansible VPU Encoding

Provision a NetActuate virtual machine with a NETINT Quadra T1U VPU attached, install the full NETINT software stack, and run CPU vs VPU hardware-accelerated encoding benchmarks for H.264, H.265, and AV1.

## What This Deploys

- A NetActuate VM with a NETINT Quadra T1U VPU (Virtual Function attachment)
- NETINT libxcoder (Quadra hardware abstraction library)
- NETINT FFmpeg build with `ni_quadra` hardware encoders and libx264/libx265 CPU encoders in a single binary
- NETINT Bitstreams encoding management software (web dashboard and REST API)
- A systemd service that initializes VPU resources at boot
- Automated CPU vs VPU encoding comparison for H.264, H.265, and AV1
- Full VPU transcode test (H.264 decode + H.265 encode on card)

## Prerequisites

### Control Node

Install Ansible and the NetActuate collection:

**macOS:**
```bash
brew install ansible
ansible-galaxy collection install netactuate.cloud
```

**Linux:**
```bash
pip install ansible
ansible-galaxy collection install netactuate.cloud
```

### NetActuate Account

- API key from **Account → API Access** in the [portal](https://portal.netactuate.com)
- A VPU-enabled location (default: DFW2)

## Configuration

Set your API key:

```bash
export NETACTUATE_API_KEY="your-api-key-here"
```

## Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `vpu_hostname` | VM hostname | `vpu-encoder.example.com` |
| `vpu_plan` | VM plan (needs sufficient resources for encoding) | `VR8x4x100` |
| `vpu_location` | VPU-enabled PoP (IATA code) | `DFW2` |
| `vpu_ssh_key_id` | SSH key ID from **Account → SSH Keys** | Set during initial setup |
| `vpu_firmware_id` | Quadra firmware ID | `1` |
| `test_input_url` | URL to test video (default: Big Buck Bunny 1080p) | Big Buck Bunny 1080p 10s |
| `test_duration` | Synthetic test video duration (when test_input_url is empty) | `10` |
| `skip_build` | Skip NETINT stack compilation (use with golden image) | `false` |
| `h264_bitrate` | H.264 target bitrate | `5M` |
| `h265_bitrate` | H.265 target bitrate | `3M` |
| `av1_bitrate` | AV1 target bitrate | `2M` |
| `vpu_ssh_key_file` | Path to SSH private key | `~/.ssh/id_rsa` |
| `vpu_ssh_user` | SSH login user | `ubuntu` |

## Usage

### Full Deploy and Test (~10 min, compiles from source)

```bash
ansible-playbook netint-vpu-deploy.yml -e vpu_ssh_key_id=YOUR_KEY_ID
```

> **Note:** Set `vpu_ssh_key_id` once in your variables or AI assistant defaults. All examples below assume it is already configured.

### Fast Deploy from Pre-Built Image (~75 sec to encoding-ready)

A NetActuate NETINT certified VM image will be available in the near future, reducing the time to launch a fully configured VPU VM to seconds:

```bash
ansible-playbook netint-vpu-deploy.yml -e vpu_image_id=NETINT_IMAGE_ID -e skip_build=true
```

### Building a Custom Golden Image

Many customers will want to build their own golden image with custom tooling, encoding scripts, or application-specific configuration alongside the NETINT stack. You can snapshot a configured VM into a reusable image using [My Images](https://www.netactuate.com/docs/platform/os-images/my-images).

The golden image builder automates this:

```bash
ansible-playbook netint-vpu-build-image.yml
# Note the image_id from the output
```

Then deploy from your custom image:
```bash
ansible-playbook netint-vpu-deploy.yml -e vpu_image_id=YOUR_IMAGE_ID -e skip_build=true
```

> **Note:** The golden image build must wait for the full image pipeline to complete before deleting the source VM. The playbook handles this automatically.

### Deploy with Custom Video File

```bash
ansible-playbook netint-vpu-deploy.yml \
  -e vpu_ssh_key_id=YOUR_KEY_ID \
  -e test_input_url=https://example.com/my-video.mp4
```

### Deploy with Synthetic Test Pattern

```bash
ansible-playbook netint-vpu-deploy.yml \
  -e vpu_ssh_key_id=YOUR_KEY_ID \
  -e 'test_input_url=' \
  -e test_duration=30
```

### Override Defaults

```bash
ansible-playbook netint-vpu-deploy.yml \
  -e vpu_plan=VR8x8x200 \
  -e vpu_hostname=encoder-01.example.com \
  -e h264_bitrate=8M \
  -e test_duration=30
```

### What Happens

1. **Provision** -- Creates a VM with a Quadra T1U VPU attached via Virtual Function
2. **Wait** -- Polls SSH until the VM is reachable (up to 5 minutes)
3. **Install** -- Clones and builds libxcoder and netint_ffmpeg from source (with libx264/libx265 support)
4. **Initialize** -- Runs `init_rsrc` and creates a systemd service for boot-time initialization
5. **Bitstreams** -- Downloads and installs NETINT Bitstreams encoding management software
6. **Verify** -- Confirms `ni_quadra` encoders are available in FFmpeg
7. **Interlace check** -- Detects and deinterlaces interlaced input (VPU requires progressive content)
8. **CPU Baseline** -- Encodes with libx264, libx265, and libaom-av1
9. **VPU Encode** -- Encodes with h264_ni_quadra_enc, h265_ni_quadra_enc, and av1_ni_quadra_enc
10. **VPU Transcode** -- Full hardware pipeline: H.264 decode + H.265 encode on the card
11. **Report** -- Displays side-by-side comparison of timing and file sizes

### Tested Benchmark Results (Quadra T1U, VR8x4x100, DFW2)

| Codec | CPU Time | VPU Time | Speedup |
|-------|----------|----------|---------|
| H.264 | 28.62s | 1.36s | **21x** |
| H.265 | 50.03s | 1.29s | **39x** |
| AV1 | — | 1.53s | — |
| Full Transcode (H.264→H.265) | — | 1.29s | — |

### Expected Output

```
╔════════════════════════════════════════════════════════════════════╗
║              CPU vs VPU Encoding Comparison                      ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  H.264                                                           ║
║    CPU (libx264):             12.34 sec   0.62 MB                ║
║    VPU (h264_ni_quadra_enc):   0.48 sec   0.61 MB               ║
║                                                                  ║
║  H.265                                                           ║
║    CPU (libx265):             28.91 sec   0.39 MB                ║
║    VPU (h265_ni_quadra_enc):   0.52 sec   0.38 MB               ║
║                                                                  ║
║  AV1                                                             ║
║    CPU (libaom-av1):         142.67 sec   0.26 MB                ║
║    VPU (av1_ni_quadra_enc):    0.71 sec   0.25 MB               ║
║                                                                  ║
║  Full VPU Transcode (H.264 decode → H.265 encode)               ║
║    VPU pipeline:               0.31 sec   0.38 MB               ║
║                                                                  ║
╚════════════════════════════════════════════════════════════════════╝
```

## Idempotency

The playbook is safe to re-run. Build steps use `creates:` guards and `stat` checks to skip work that has already been completed. The encoding tests always run to provide fresh comparison data.

## Bitstreams Encoding Management

After deployment, the VM includes NETINT Bitstreams -- a web-based encoding management tool accessible at `http://<vm-ip>`. Bitstreams replaces raw FFmpeg commands with a dashboard and REST API for managing encoding templates and live streaming sessions.

Key capabilities:
- **Templates** -- reusable encoding configurations (codec, resolution, bitrate, lookahead, overlays)
- **Live sessions** -- SRT and RTMP input with DASH/HLS/CMAF/SRT output
- **Offline sessions** -- file-to-file transcoding
- **Dashboard** -- real-time VPU utilization, temperature, and stream health
- **REST API** -- programmatic control at `http://<vm-ip>/api/v3/`
- **Interlace handling** -- built-in deinterlace filters for interlaced sources

For the full Bitstreams documentation, see [docs.netint.com/bitstreams](https://docs.netint.com/bitstreams/).

## Important: Interlaced Content

The Quadra VPU does not support interlaced content for encoding or decoding. The playbook automatically detects and deinterlaces interlaced input before running VPU encoding tests. When encoding your own content with FFmpeg directly, verify the input is progressive first:

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=field_order -of default=noprint_wrappers=1:nokey=1 input.mp4
```

If the output is `tt`, `bb`, `tb`, or `bt`, deinterlace before VPU encoding. Bitstreams handles interlaced input automatically through its built-in filters.

## AI-Assisted Deployment

Open this repo in Claude Code, Cursor, or GitHub Copilot and use prompts like:

### Bitstreams workflows

```
Deploy a VPU VM, then create a Bitstreams template called "live source"
that encodes to HEVC with resolutions 1080p, 720p, 480p.
Set the lookahead to 15.
```

```
Create a new Bitstreams session called "my-stream" using input
srt://10.1.1.10:8000 with the "live source" template. Provide a DASH output.
```

```
What is the status of all Bitstreams sessions?
```

### FFmpeg benchmarks

```
Deploy a VPU VM and run encoding benchmarks.
Compare CPU vs VPU for H.264, H.265, and AV1.
```

```
Deploy a VPU VM and encode this file with both CPU and VPU:
https://example.com/my-4k-content.mp4
```

```
Deploy a VPU VM and test encoding at broadcast-quality bitrates:
  H.264: 10M, H.265: 6M, AV1: 4M
Use a 30-second test video.
```

See `AGENTS.md` for the full AI context and additional workflow examples.

## NETINT Quadra T1U Specifications

| Capability | Value |
|-----------|-------|
| Encode codecs | H.264, H.265, AV1, JPEG |
| Decode codecs | H.264, H.265, VP9, AV1, JPEG |
| Max streams | 32x 1080p30, 8x 4Kp30, 2x 8Kp30 |
| Resolution range | 32x32 to 8192x5120 |
| Bitrate range | 64 kbit/s to 700 Mbit/s |
| Bit depth | 8-bit and 10-bit |
| HDR | HDR10, HDR10+, HLG |
| AI engine | 18 TOPS |
| Power | 20W typical |
| Interface | PCIe 4.0 x4 |

## Need Help?

- **NETINT documentation:** [docs.netint.com](https://docs.netint.com)
- **NetActuate support:** [support@netactuate.com](mailto:support@netactuate.com) or open a ticket from the [portal](https://portal.netactuate.com)
- **VPU encoding guide:** [netactuate.com/docs/guides/vpu-encoding-with-netint-quadra](https://www.netactuate.com/docs/guides/vpu-encoding-with-netint-quadra)
