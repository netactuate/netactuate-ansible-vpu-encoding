# NetActuate Ansible VPU Encoding

Provision a VM with a NETINT Quadra T1U VPU, install the NETINT software stack, and run CPU vs VPU encoding benchmarks for H.264, H.265, and AV1.

## Required Inputs

| Input | Source | Example |
|-------|--------|---------|
| `NETACTUATE_API_KEY` | Environment variable | `export NETACTUATE_API_KEY="your-key"` |
| `vpu_ssh_key_id` | Portal → Account → SSH Keys | Set once during initial setup |

## Optional Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `vpu_plan` | VM plan | `VR8x4x100` |
| `vpu_hostname` | VM hostname | `vpu-encoder.example.com` |
| `vpu_location_id` | Location ID (281=DFW2) | `281` |
| `vpu_image_id` | OS image ID (5787=Ubuntu 24.04) | `5787` |
| `vpu_accelerator_type_id` | Accelerator type from API | `1` |
| `vpu_firmware_id` | Firmware ID from API | `1` |
| `vpu_attachment_type` | `vf` or `pf` | `vf` |
| `vpu_contract_id` | Billing contract ID | Set via environment |
| `vpu_ssh_key_file` | Path to SSH private key | `~/.ssh/id_rsa` |
| `vpu_ssh_user` | SSH login user | `ubuntu` |
| `test_input_url` | URL to test video (default: Big Buck Bunny 1080p 10s) | Big Buck Bunny |
| `test_duration` | Synthetic test duration when test_input_url is empty | `10` |
| `skip_build` | Skip NETINT compile (use with golden image) | `false` |
| `h264_bitrate` | H.264 target bitrate | `5M` |
| `h265_bitrate` | H.265 target bitrate | `3M` |
| `av1_bitrate` | AV1 target bitrate | `2M` |

## What to Do

1. Set the API key:
   ```bash
   export NETACTUATE_API_KEY="your-api-key"
   ```

2. Full deploy (compiles NETINT stack from source, installs Bitstreams, ~10 min):
   ```bash
   ansible-playbook netint-vpu-deploy.yml
   ```

3. Fast deploy from pre-built image (~75 sec):
   ```bash
   ansible-playbook netint-vpu-deploy.yml -e vpu_image_id=IMAGE_ID -e skip_build=true
   ```

4. Run with a custom test file:
   ```bash
   ansible-playbook netint-vpu-deploy.yml -e test_input_url=https://example.com/my-video.mp4
   ```

5. Watch the results -- the playbook downloads Big Buck Bunny 1080p as the default test file and outputs a CPU vs VPU comparison table.

## Important: Interlaced Content

The Quadra VPU does not support interlaced content for encoding or decoding. Before running VPU encoding on user-provided content, check if the input is interlaced:

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=field_order -of default=noprint_wrappers=1:nokey=1 input.mp4
```

If the output is `tt`, `bb`, `tb`, or `bt`, the content is interlaced and must be deinterlaced first:

```bash
ffmpeg -i input.mp4 -vf yadif -c:v libx264 -preset ultrafast output_progressive.mp4
```

The playbook handles this automatically, but when running ad-hoc FFmpeg commands, always verify first. Bitstreams handles interlaced input automatically through its built-in deinterlace filters.

## What the Playbook Does

1. **Provisions** a VM via the NetActuate v2 API with a Quadra T1U VPU (Virtual Function)
2. **Waits** for SSH (up to 5 minutes)
3. **Builds** libxcoder and netint_ffmpeg from source (idempotent — skips on re-run)
4. **Initializes** VPU and creates a systemd boot service
5. **Verifies** ni_quadra encoders are available in FFmpeg
6. **Prepares** test input (downloads from URL or generates 1080p test pattern)
7. **Encodes with CPU**: H.264 (libx264), H.265 (libx265), AV1 (libaom-av1)
8. **Encodes with VPU**: H.264, H.265, AV1 using ni_quadra encoders
9. **Runs full VPU transcode**: H.264 decode → H.265 encode (entire pipeline on card)
10. **Displays** comparison table with timing and file sizes
11. **Shows** VPU utilization and temperature via ni_rsrc_mon

## Bitstreams Workflows

After deployment, the VM includes Bitstreams -- NETINT's encoding management software accessible via web dashboard (`http://<vm-ip>`) and REST API (`http://<vm-ip>/api/v3/`).

### Authentication

Bitstreams API uses Basic Auth with access tokens (not login credentials).

The playbook auto-creates a token during deployment. Default credentials:
- **Token ID:** `netactuate-api`
- **Token Secret:** `netactuate-automation`

Override via playbook variables `bitstreams_token_id` and `bitstreams_token_secret`.

To create a token manually via MySQL on the VM:
```bash
TOKEN_ID="my-token"
TOKEN_SECRET="my-secret"
TOKEN_MD5=$(echo -n "$TOKEN_SECRET" | md5sum | awk '{print $1}')
docker exec configs-mysql-1 mysql -u root -p"$BITSTREAMS_DB_PWD" ni_user -e "
  INSERT IGNORE INTO livetran_user_token (userid, user_name_id, name, description, token_id, token_secret_md5, environment_id, permissions, status, expires_at, create_time, update_time)
  SELECT u.userid, un.id, 'api', 'API token', '$TOKEN_ID', '$TOKEN_MD5', 0, '0', 1, 0, NOW(), NOW()
  FROM livetran_user u JOIN livetran_user_name un ON u.userid = un.userid LIMIT 1;
"
```

Use in API calls:
```bash
AUTH=$(echo -n "netactuate-api:netactuate-automation" | base64)
curl -sL -H "Authorization: Basic $AUTH" http://<vm-ip>/api/v3/template/
```

> **Note:** All API paths require a trailing slash (e.g., `/api/v3/template/` not `/api/v3/template`).

### API Reference (Key Endpoints)

**Templates:**
- `GET /api/v3/template/` -- list all templates
- `POST /api/v3/template/` -- create template
- `GET /api/v3/template/{id}` -- get template
- `PUT /api/v3/template/{id}` -- update template
- `DELETE /api/v3/template/{id}` -- delete template

**Live Sessions (Streams):**
- `GET /api/v3/streams/` -- list all sessions
- `POST /api/v3/streams/` -- create session
- `GET /api/v3/streams/{id}` -- get session details
- `GET /api/v3/streams/name/{name}` -- get session by name
- `PUT /api/v3/streams/{id}/start` -- start session
- `PUT /api/v3/streams/{id}/stop` -- stop session
- `DELETE /api/v3/streams/{id}` -- delete session
- `GET /api/v3/streams/{id}/playback_status` -- playback status
- `GET /api/v3/streams/health_report` -- health report for all sessions

**Offline Transcoding:**
- `GET /api/v3/transcoding/` -- list transcoding jobs
- `POST /api/v3/transcoding/` -- create transcoding job
- `PUT /api/v3/transcoding/{id}/start` -- start job
- `PUT /api/v3/transcoding/{id}/stop` -- stop job

**System:**
- `GET /api/v3/regions` -- get region info (required for stream creation)
- `GET /api/v3/other/information` -- system info and VPU status
- `GET /api/v3/images` -- list uploaded overlay images
- `POST /api/v3/images/upload` -- upload overlay image

### Create a Template

```bash
curl -sL -X POST http://<vm-ip>/api/v3/template/ \
  -H "Authorization: Basic $AUTH" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "live source",
    "description": "HEVC 1080p/720p/480p with lookahead 15",
    "output": {
      "video": [
        {"codec": "h265", "width": 1920, "height": 1080, "bitrate": 5000},
        {"codec": "h265", "width": 1280, "height": 720, "bitrate": 3000},
        {"codec": "h265", "width": 854, "height": 480, "bitrate": 1500}
      ]
    },
    "advanced_params_quadra": {
      "encoder_params": {
        "enable_lookahead": true,
        "lookahead_depth": 15
      }
    }
  }'
```

Video codec options: `h264`, `h265`, `av1`. Bitrate is in kbps (100-100000). Resolution width/height in pixels (128-8192, or 0 for source resolution). FPS: 1.0-60.0.

### Create a Live Session

First get the region:
```bash
REGION=$(curl -sL -H "Authorization: Basic $AUTH" http://<vm-ip>/api/v3/regions/ | python3 -c "import sys,json; print(json.load(sys.stdin)['data'][0]['region'])")
```

Then create the session:
```bash
curl -sL -X POST http://<vm-ip>/api/v3/streams/ \
  -H "Authorization: Basic $AUTH" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "nab2026",
    "description": "NAB 2026 demo stream",
    "input_type": "srt_push",
    "template_id": <TEMPLATE_ID>,
    "region": "'$REGION'",
    "output_types": ["dash"]
  }'
```

Input types: `srt_push`, `rtmp_push`, `srt_pull`, `hls_pull`, `dash_pull`, `multicast_pull`, `rtsp_pull`, `rtp_pull`, `capture_card`.
Output types: `dash`, `hls`, `cmaf`, `srt`, `rtmp`, `multicast`.

### Monitor Session Status

```bash
# Get all sessions
curl -sL -H "Authorization: Basic $AUTH" http://<vm-ip>/api/v3/streams/

# Get session by name
curl -sL -H "Authorization: Basic $AUTH" http://<vm-ip>/api/v3/streams/name/nab2026

# Get health report
curl -sL -H "Authorization: Basic $AUTH" http://<vm-ip>/api/v3/streams/health_report
```

### Start / Stop / Delete Session

```bash
# Start
curl -sL -X PUT -H "Authorization: Basic $AUTH" http://<vm-ip>/api/v3/streams/<id>/start

# Stop
curl -sL -X PUT -H "Authorization: Basic $AUTH" http://<vm-ip>/api/v3/streams/<id>/stop

# Delete
curl -sL -X DELETE -H "Authorization: Basic $AUTH" http://<vm-ip>/api/v3/streams/<id>
```

### Update a Template

```bash
curl -sL -X PUT http://<vm-ip>/api/v3/template/<id> \
  -H "Authorization: Basic $AUTH" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "live source",
    "output": {
      "video": [
        {"codec": "h265", "width": 1920, "height": 1080, "bitrate": 5000},
        {"codec": "h265", "width": 1920, "height": 1080, "bitrate": 5000, "fps": 59.94},
        {"codec": "h265", "width": 1280, "height": 720, "bitrate": 3000},
        {"codec": "h265", "width": 854, "height": 480, "bitrate": 1500}
      ]
    }
  }'
```

### Upload an Overlay Image

```bash
curl -sL -X POST http://<vm-ip>/api/v3/images/upload \
  -H "Authorization: Basic $AUTH" \
  -F "file=@logo.png"
```

Then reference the image_id in the template's `output.logo` array with position settings.

### AI-Assisted Bitstreams Prompts

Users can ask an AI assistant to drive these API calls with natural language:

```
Create a new Bitstreams template called "live source" that encodes
to HEVC with resolutions 1080p, 720p, 480p and set the lookahead to 15.
```

```
Create a new Bitstreams session called "nab2026" using input
srt://10.1.1.10:8000 with the "live source" template. Provide a DASH output.
```

```
What is the status of the "nab2026" session?
```

```
Update the "live source" template to add a second 1080p output at 59.94 fps.
```

```
Stop "nab2026" and delete the session.
```

The AI should authenticate against the Bitstreams API at `http://<vm-ip>/api/v3/`, create tokens if needed, and execute the appropriate REST calls. The Swagger spec is bundled at `/opt/bitstreams/Bitstreams_Edge_v2.12.0_swagger.json` on the VM for full API reference.

## FFmpeg Encoding Workflows

Users can also ask an AI assistant (Claude Code, Cursor, Copilot) to run FFmpeg encoding scenarios directly:

**Basic deployment and test:**
```
Deploy a VPU VM and run encoding benchmarks.
Compare CPU vs VPU for H.264, H.265, and AV1.
```

**Test with a specific video file:**
```
Deploy a VPU VM and encode this file with both CPU and VPU:
https://example.com/my-4k-content.mp4
```

**Custom bitrate targets:**
```
Deploy a VPU VM and test encoding at these bitrates:
  H.264: 8M, H.265: 5M, AV1: 3M
Use a 30-second test video.
```

**Re-run tests on existing VM:**
```
Re-run the encoding comparison on the VPU VM with this test file:
https://example.com/new-content.mp4
```

The AI should clone this repo, read this AGENTS.md, set variables, and run the playbook. For re-runs on existing VMs, it can run the encoding commands directly via SSH.

## VPU-Enabled Locations

VPU availability varies by location. Common VPU-enabled PoPs:

You can discover VPU-enabled locations via the API:

```bash
# Get accelerator categories
curl "https://vapi2.netactuate.com/api/cloud/accelerator-types/category?key=$NETACTUATE_API_KEY"

# Check a specific location for VPU availability
curl "https://vapi2.netactuate.com/api/cloud/accelerator-types?category=VPU&location_id=281&key=$NETACTUATE_API_KEY"
```

Currently available:

| IATA Code | Location | Cards |
|-----------|----------|-------|
| `DFW2` | Dallas, TX (ID 281) | 4x Quadra T1U |

Check the portal under **Infrastructure → Virtual Machines → + Add** with accelerator filter enabled, or query the API above for current availability.

## Key Behavior

- The VM is provisioned with `accelerators` specifying a Quadra T1U with Virtual Function attachment
- NETINT FFmpeg is built with `--enable-libx264 --enable-libx265 --enable-gpl` so a single binary (`/usr/local/bin/ffmpeg`) handles both CPU and VPU encoding
- CPU encoding uses standard FFmpeg encoders (libx264, libx265, libaom-av1) for fair comparison
- VPU encoding uses ni_quadra hardware encoders
- Full VPU transcode tests the complete hardware pipeline (decode + encode on card)
- Timing uses `/usr/bin/time` for accurate wall-clock measurement
- Build steps are idempotent — safe to re-run
- The play fails if `ni_quadra` encoders are not found in FFmpeg

## Common Errors

| Error | Fix |
|-------|-----|
| `NETACTUATE_API_KEY not set` | `export NETACTUATE_API_KEY="your-key"` |
| API returns 422 on buy_build | Check plan name exists and location_id is valid for VPU |
| SSH timeout | VM may still be building — increase timeout or retry |
| `ni_quadra` encoders not found | VPU may not be attached — check portal for accelerator status |
| Encoding test fails | Check `ni_rsrc_mon` output for VPU health |
| `libaom-av1` not available | CPU AV1 encoder may not be compiled in — AV1 CPU baseline will be skipped |
| Location does not support VPU | Use a VPU-enabled location -- query the accelerator-types API or use DFW2 |
| Bitstreams license error (err_code 2000) | Bitstreams requires a valid license for write operations. Contact NetActuate support or NETINT. |
| Bitstreams 401 "basic auth error" | API requires token-based auth, not login credentials. Generate tokens via `POST /api/v3/secrets/` first. |
| Bitstreams 301 redirect | Add a trailing slash to the API path (e.g., `/api/v3/template/` not `/api/v3/template`). |
