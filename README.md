# Anchor: a local recovery check-in agent on Dell Pro Precision GB10

Anchor is a local AI recovery check-in agent for an outpatient recovery clinic. Clinicians author the care plan
and review every alert, call, transcript and audit event, while patients take a consented check-in call in a
catalog voice, a personalized voice clone or text only. The language model, speech and database all run on one
Dell Pro Precision GB10, and the clinician stays in charge: the agent never contacts a patient or changes a care plan
on its own.

All clinic and patient data in this demo is synthetic. Voice cloning needs the informed consent of the person whose
voice is cloned, and the caller is always disclosed as a digital twin.

## Watch it first

The narrated overview video, which describes what Anchor does, is attached to the v1.0 release under
[Releases](../../releases).

## The stages of a check-in

| Stage | What happens | Check or human gate |
| --- | --- | --- |
| 1. Plan | A clinician authors and publishes a bounded recovery plan in the care console (`/`). | Human gate: only a clinician publishes a plan. Each publish creates an immutable revision; the model cannot publish or change a plan. |
| 2. Consent | The patient opens the call page (`/patient`) and picks one of four reference-free catalog voices, a consented personalized voice or text only. | Check: a call without voice consent is rejected. Catalog voices never use the personalized reference audio. |
| 3. Call | Anchor simulates the outbound call in the browser and loads the active plan, current signals and facts from earlier calls from MongoDB. | Check: the greeting and allowed options come only from the clinician's plan and stored records. |
| 4. Conversation | The patient speaks (microphone audio is transcribed on the GB10 by Whisper tiny.en) or types. Nemotron, served locally by NVIDIA vLLM, writes a short reply and Sesame CSM-1B speaks it. | Check: no remote language or speech API is configured. Raw microphone audio is processed in memory and not stored. |
| 5. Safety | Every turn is scored by deterministic rules. A concerning turn is written to MongoDB as an alert before anything else happens, and high-risk replies use fixed, clinician-authored wording. Crisis language tells the patient to contact emergency or crisis services. | Human gate: the alert waits for clinician review. The patient is never told a clinician was contacted. |
| 6. Agent handoff (optional) | If configured, Anchor wakes an OpenClaw agent inside NVIDIA OpenShell with a bounded review task, routed only to the local Nemotron model. | Check: the delivery result (`delivered`, `failed` or `not-configured`) is stored on the same alert. OpenClaw cannot contact a patient or edit a plan. |
| 7. Hang up | Nemotron extracts facts and a one-line summary; the call, transcript, facts and audit events are saved for the next check-in. | Human gate: the clinician reviews calls, transcripts, alerts and the audit trail in the console and resolves alerts. |
| 8. Prove it | `scripts/verify-nvidia.py` runs the whole path end to end; `/goal` shows live GB10 telemetry and the architecture. | Check: the launcher reports ready only after MongoDB, Nemotron and a real CSM voice sample are healthy and the full workload passes. |

## Hardware

Built and run on:

| Component | Built and run on |
| --- | --- |
| System | Dell Pro Precision GB10 |
| GPU | NVIDIA GB10 Grace Blackwell superchip, 128 GB unified LPDDR5x (shared by CPU and GPU) |
| CPU | 20-core NVIDIA Grace Arm CPU |
| Memory | 128 GB unified LPDDR5x (shared by CPU and GPU) |
| Storage | 4 TB NVMe SSD |
| OS | NVIDIA DGX OS |
| Software | NVIDIA vLLM serving nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-NVFP4; Sesame CSM-1B (Transformers, CUDA BF16); Whisper tiny.en; MongoDB 8; OpenClaw 2026.5.27 in NVIDIA OpenShell 0.0.106 |

## What's in this repo

| Path | What |
| --- | --- |
| `app/careline/` | FastAPI application: call flow, deterministic safety scoring, MongoDB store, synthetic fixtures, Nemotron, CSM and Whisper clients, OpenClaw wake adapter, `/goal` telemetry |
| `app/careline/voice-catalog.json` | The four reference-free catalog voices |
| `app/web/` | Browser interfaces: clinician console (`/`), patient call (`/patient`), live operations view (`/goal`) |
| `app/Dockerfile.csm-cuda` | Application image on the NVIDIA PyTorch container with CUDA Sesame CSM |
| `compose.nvidia.yml` | Compose stack: MongoDB 8, NVIDIA vLLM with Nemotron, the Anchor app, optional mongo-express inspector |
| `scripts/run-nvidia` | Launcher: checks the host and bundle, starts the stack, waits for readiness, runs the full verifier |
| `scripts/verify-nvidia.py` | End-to-end check of the consent, voice, safety, alert and memory path |
| `scripts/verify-websocket.py` | Live check of text check-ins over WebSocket |
| `scripts/check-csm-assets.py`, `scripts/validate-fixtures.py` | Static checks of the CSM inputs and the synthetic roster |
| `scripts/private-tcp-proxy.py` | Binds the local model to one private bridge for the OpenShell route |
| `config/openshell-openai-local.yaml` | OpenShell provider profile for the local Nemotron route |
| `tests/test_websocket.py` | WebSocket transport tests |
| `GB10-RUNBOOK.md` | Full start, inspect, verify and stop checklist |
| `OPENSHLL.md` | Optional OpenShell and OpenClaw handoff setup and its private network boundary |
| `SYSTEM-ARCHITECTURE.md`, `IMPLEMENTATION.md`, `DATA-MODEL.md` | Architecture, trust boundaries, call path and persisted data model |
| `docs/README.md`, `docs/websocket-messaging.md` | Documentation index and text messaging through the app over WebSocket |
| `docs/ux-audit/` | Screenshots of the patient call and clinician roster |
| `blueprint.yaml`, `CHANGELOG.md`, `.env.example` | Project summary, change history and example environment settings |

## Setup

Requirements:

- Dell Pro Precision GB10 with NVIDIA DGX OS, Docker Engine with Compose, and the NVIDIA Container Toolkit.
- The NVIDIA vLLM container `nvcr.io/nvidia/vllm:26.05.post1-py3`, available locally or pulled from NGC.
- The private Anchor bundle (model weights, CUDA wheelhouse, personalized voice recording and the offline
  OpenClaw/OpenShell bundle): not included, contact Dell support (Logan Lawler).

Steps (Linux on the GB10):

1. Download or clone this repository onto the GB10 and open a terminal in its root folder.
2. Mount the private bundle and point Anchor at it (use the folder where you mounted it):

   ```bash
   export HACKATHON_BUNDLE=/mnt/anchor-bundle
   ```

   You can keep this and the other settings in a `.env` file based on `.env.example`. Change `MONGO_PASSWORD` from the
   local default on any shared machine.
3. Start the stack:

   ```bash
   ./scripts/run-nvidia
   ```

   The launcher checks the host and the bundle, starts MongoDB and Nemotron, generates a real voice sample and runs
   the full call test before it prints `Anchor NVIDIA GB10 ready: http://127.0.0.1:8100/`.
4. Open the clinician console at <http://127.0.0.1:8100/>, the patient call at <http://127.0.0.1:8100/patient> and the
   live GB10 operations view at <http://127.0.0.1:8100/goal>. On the patient page, choose one of the four catalog
   voices or **Text only** (the default path), answer the call and hang up; the call then appears in the clinician
   console.
5. Repeat the verification at any time without rebuilding:

   ```bash
   CARELINE_BASE_URL=http://127.0.0.1:8100 python3 scripts/verify-nvidia.py
   ```

6. Optional: attach the local agent runtime by following [OPENSHLL.md](OPENSHLL.md).
7. Stop with `docker compose -f compose.nvidia.yml down`; the clinic records stay in the `mongo-data` volume.

The full checklist (start, inspect, verify, MongoDB inspector, agent handoff and stop) is
[GB10-RUNBOOK.md](GB10-RUNBOOK.md).

## Known limits

- Anchor is a proof of concept and a wellness workflow demo, not medical care. It does not diagnose, prescribe or
  change a care plan on its own, and it claims no clinical efficacy.
- It simulates the outbound phone call in the browser and does not dial phones.
- It is not for real patient data or protected health information. All clinic and patient data is synthetic.
- It cannot run without the private bundle (model weights, voice recording and the offline OpenClaw/OpenShell
  bundle), which is not in this repository. Contact Dell support for it.
- Voice cloning needs the informed consent of the person whose voice is used. Never commit a voice reference or its
  transcript; `.gitignore` excludes environment files, databases, recordings, checkpoints and model weights.
- Without the optional OpenClaw hook, alerts are stored and marked `not-configured`; no one is notified automatically.
- Post-call summaries are model-written and need clinician review: in one synthetic test call the summary described
  planned actions as completed.
- No run time is stated because none was measured.
- Production use would need identity and role controls, encryption, tenant isolation, retention and deletion policy,
  backups, consent revocation, access audit and clinical validation.

## Credits

Created by the Anchor team: Jai Bhagat. Anchor's source is
[ChaiWithJai/anchor-nvidia-gb10](https://github.com/ChaiWithJai/anchor-nvidia-gb10), which extends the CareLine
blueprint in [ChaiWithJai/hybrid-ai-blueprints](https://github.com/ChaiWithJai/hybrid-ai-blueprints) (branch
`blueprint/careline-wellness-checkin`). If you find Anchor useful, please star both repositories. Dell Technologies
shares this copy with the team's permission; Dell's changes are rebranding, documentation, removal of the
temporary public-tunnel sharing option and removal of the team's later planning documents and tools, which describe
work that is not part of what runs here.

- **Partner:** NVIDIA (GB10 Grace Blackwell superchip, DGX OS, Nemotron, vLLM container, OpenShell).
- **Dependencies:** no dependency repositories are bundled or cloned. The model weights, voice recording and
  offline OpenClaw/OpenShell bundle come from the private bundle; containers and Python packages are pulled by name
  as listed in `compose.nvidia.yml` and `app/Dockerfile.csm-cuda`.
- **Upstream licences:** the source repository has no licence file, so the team's code is not relicensed here (see
  [LICENSE](LICENSE)). No licence is claimed here for the CareLine blueprint; see the notice in
  [ChaiWithJai/hybrid-ai-blueprints](https://github.com/ChaiWithJai/hybrid-ai-blueprints) itself. Nemotron model weights, Sesame CSM-1B, Whisper, MongoDB 8, mongo-express, OpenClaw, NVIDIA
  OpenShell and the NVIDIA containers keep their own terms; see [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
- **AI-generated assets:** all spoken audio is generated at run time by Sesame CSM-1B. The four catalog voices are
  reference-free generations that represent no human voice; the personalized voice is a clone made only with its
  owner's consent. Replies and call summaries are generated by Nemotron.
- **Trademarks:** NVIDIA, Grace Blackwell, DGX, Nemotron and OpenShell are trademarks of NVIDIA Corporation. MongoDB
  is a trademark of MongoDB, Inc. Sesame CSM, Whisper, OpenClaw, Docker and other names are the property of their
  respective owners. Their use here does not imply
  endorsement of Dell Technologies, and Dell does not endorse their products by naming them.

## Support

Logan Lawler, Dell Technologies: logan.lawler@dell.com

## License

Copyright (c) 2026 the Anchor team: Jai Bhagat. The source repository has no licence file, so the Anchor team's code
is not offered under MIT or any open-source licence. Dell Inc. shares it with the authors' permission so that you can
download, run and evaluate it as a demo; no other rights are granted. Dell's changes are Copyright (c) 2026 Dell Inc.
and carry the same terms. Third-party components, models and data keep their own licences. See [LICENSE](LICENSE)
and [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
