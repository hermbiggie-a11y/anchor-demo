# GB10 runbook

## Prerequisites

- Dell Pro Precision GB10 (ARM64, NVIDIA GB10) running NVIDIA DGX OS
- Docker Engine, Compose, and NVIDIA Container Toolkit
- The private Anchor bundle (model weights, CUDA wheelhouse, voice enrollment,
  offline OpenClaw/OpenShell bundle) mounted locally. It is not included in
  this repository; contact Dell support (Logan Lawler) for it.
- `nvcr.io/nvidia/vllm:26.05.post1-py3` available locally or pullable from NGC

## Start and prove the workload

From the root of your copy of this repository:

```bash
export HACKATHON_BUNDLE=/mnt/anchor-bundle   # the folder where you mounted the bundle
./scripts/run-nvidia
```

Expected final line:

```text
Anchor NVIDIA GB10 ready: http://127.0.0.1:8100/
```

The launcher validates the host and local artifacts, starts MongoDB and
Nemotron, performs a cloned-voice warmup, and runs the full clinical call test.
Startup can take several minutes while vLLM loads five NVFP4 shards.

## Inspect

```bash
docker compose -f compose.nvidia.yml ps
docker compose -f compose.nvidia.yml logs --tail 100 nemotron
docker compose -f compose.nvidia.yml logs --tail 100 digital-twin
curl --fail http://127.0.0.1:8100/api/status
nvidia-smi
```

`/api/status` is ready only when MongoDB responds, Nemotron lists the pinned
model, and CSM has generated a valid 24 kHz WAV on CUDA.

Open both working interfaces:

- Clinician console: <http://127.0.0.1:8100/>
- Patient call: <http://127.0.0.1:8100/patient>

The patient flow uses a consented synthetic record. A completed or escalated
call appears in the clinician console without a refresh of the underlying
workload.

## Attach the local agent runtime

The application is complete without an agent hook: alerts are persisted first
and report `not-configured`. To demonstrate the local agent stack, attach
OpenClaw inside NVIDIA OpenShell using
[OPENSHLL.md](OPENSHLL.md). The validated route is:

```text
Anchor container -> private bearer hook -> OpenClaw 2026.5.27
  -> inference.local -> OpenShell 0.0.106 -> loopback Nemotron on GB10
```

The hosted NemoClaw plugin is disabled in this configuration. OpenShell forces
the model route to the local Nemotron provider and denies general model egress.
`/api/status` reports `agent_runtime.configured: true` only when both the hook
URL and its separate bearer token are present.

## Inspect local MongoDB in a browser (optional)

The optional `inspect` profile runs an authenticated mongo-express UI on the
GB10 loopback interface. It connects to the existing `mongodb` service over the
private Compose network; MongoDB port `27017` remains unpublished and the
existing `mongo-data` volume is unchanged.

```bash
export MONGO_EXPRESS_USERNAME=anchor-inspector
export MONGO_EXPRESS_PASSWORD='choose-a-separate-local-password'
docker compose -f compose.nvidia.yml --profile inspect up -d mongo-express
```

Open <http://127.0.0.1:8081/> and sign in with those two values. Override the
loopback port with `MONGO_EXPRESS_PORT` if `8081` is already in use. Keep this
developer tool on localhost. If no inspector credentials are exported, the
local demo defaults are `anchor-inspector` and `anchor-gb10-inspect-local-only`;
set distinct values on any shared workstation.

Stop the inspector without stopping Anchor or deleting its data:

```bash
docker compose -f compose.nvidia.yml --profile inspect stop mongo-express
```

## Prove MongoDB is the system of record

MongoDB is isolated on the Compose network and has no host port. Inspect it
through the service container:

```bash
docker compose -f compose.nvidia.yml exec mongodb sh -lc \
  'mongosh --quiet --username "$MONGO_INITDB_ROOT_USERNAME" \
    --password "$MONGO_INITDB_ROOT_PASSWORD" \
    --authenticationDatabase admin anchor \
    --eval "printjson({patients:db.patients.countDocuments(),calls:db.calls.countDocuments(),alerts:db.alerts.countDocuments(),audit:db.audit_events.countDocuments()})"'
```

The named `mongo-data` volume preserves clinic records across restarts.
Deleting that volume destroys the synthetic demo records and is not part of the
normal stop procedure.

## Stop

```bash
docker compose -f compose.nvidia.yml down
```

`docker compose down` retains `mongo-data`; do not add `--volumes`.
