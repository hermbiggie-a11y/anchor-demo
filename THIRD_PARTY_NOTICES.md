# Third-party notices

## Upstream repository

- **ChaiWithJai/anchor-nvidia-gb10** (Anchor, by the Anchor team): no licence file and no notice file upstream, so
  there is no notice text to reproduce. The team's code is not relicensed; see [LICENSE](LICENSE).
- **ChaiWithJai/hybrid-ai-blueprints** (CareLine blueprint, branch `blueprint/careline-wellness-checkin`), which
  Anchor extends: credited as upstream. No licence is claimed for it here and no licence text from it is included in
  this repository; see that repository's own notice.

No dependency repository is vendored into this copy, so there are no vendored LICENSE files.

## Components used at run time (not included in this repository)

Each of these keeps its own licence terms, which ship with the component. This repository does not include their
code or weights.

| Component | How Anchor uses it | Where it comes from |
| --- | --- | --- |
| nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-NVFP4 model weights | Local conversation, fact and summary model | Private bundle |
| Sesame CSM-1B | Local voice synthesis; the repository's `app/careline/voice-catalog.json` records its model licence as Apache-2.0 | Private bundle |
| Whisper tiny.en | Local microphone transcription | Private bundle |
| OpenClaw 2026.5.27 | Optional bounded clinician-review agent | Offline bundle (private) |
| NVIDIA OpenShell 0.0.106 | Sandbox and forced local inference route for OpenClaw | Offline bundle (private) |
| NVIDIA vLLM container `nvcr.io/nvidia/vllm:26.05.post1-py3` | Serves Nemotron | NVIDIA NGC |
| NVIDIA PyTorch container `nvcr.io/nvidia/pytorch:26.05-py3` | Base image of the Anchor app | NVIDIA NGC |
| MongoDB 8 (`mongo:8.0`) | Clinical system of record | Docker Hub |
| mongo-express 1.0.2 | Optional localhost database inspector | Docker Hub |
| Python packages: fastapi, uvicorn, httpx, websockets, pymongo, dnspython, transformers, accelerate, safetensors, sentencepiece, soundfile | Application runtime | Private wheelhouse and PyPI |
