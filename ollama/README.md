# Ollama

Runs Ollama as a rootless Podman container, managed via a systemd
Quadlet, with GPU passthrough for local LLM inference.

## Why Podman/Quadlet

Rootless by default, and Quadlet gives proper systemctl integration —
automatic restarts, clean startup ordering — without needing a separate
orchestration layer or Docker daemon running as root.

## What it does

- Serves a local LLM API on 127.0.0.1:11434
- Models persist in ~/.ollama on the host, mounted into the container

## Why bound to localhost only

Ollama's API has no built-in authentication — anything that can reach
the port can query the models and use the GPU. Binding to 127.0.0.1
specifically means only this machine can reach it, not anything else
on the home network.

## Setup

    mkdir -p ~/.config/containers/systemd
    cp ollama.container ~/.config/containers/systemd/
    mkdir -p ~/.ollama
    systemctl --user daemon-reload
    systemctl --user enable --now ollama.service

## GPU passthrough

This Quadlet passes through the host GPU via CDI
(AddDevice=nvidia.com/gpu=all), letting Ollama run inference on the
GPU instead of falling back entirely to CPU.

Requirements on the host:

- NVIDIA driver installed and working (nvidia-smi succeeds)
- nvidia-container-toolkit installed
- CDI spec generated: nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
- SELinux boolean enabled: sudo setsebool -P container_use_devices on

Note: for GPUs below a certain generation, the mainline driver's "open"
kernel module may not support the hardware — check lspci -k to confirm
which driver is actually bound, and use the correct legacy driver
branch if needed for older cards.

## What I learned

- Quadlet-managed containers are internally named systemd-<filename>,
  not just <filename> — worth knowing when using podman exec directly
  against them rather than through systemd.
- Volume mount directories must exist on the host before the container
  starts — Podman won't create them automatically.
