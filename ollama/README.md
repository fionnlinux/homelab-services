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

Clone the repo and move into this service's folder:

    git clone https://github.com/fionnlinux/homelab-services.git
    cd homelab-services/ollama

Confirm the NVIDIA driver is installed and working first:

    nvidia-smi

If this fails, see "GPU passthrough" below before continuing.

Install the NVIDIA Container Toolkit:

    curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
    sudo dnf install nvidia-container-toolkit

Generate the CDI spec so Podman knows how to expose the GPU to
containers:

    sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
    nvidia-ctk cdi list

Confirm your GPU is listed in the output above, then enable the
SELinux boolean that allows containers to access host devices:

    sudo setsebool -P container_use_devices on

Now set up the Quadlet itself:

    mkdir -p ~/.config/containers/systemd
    cp ollama.container ~/.config/containers/systemd/
    mkdir -p ~/.ollama
    systemctl --user daemon-reload
    systemctl --user enable --now ollama.service
    systemctl --user status ollama.service

Confirm it shows active (running), then pull a model to test:

    podman exec -it systemd-ollama ollama pull phi4-mini
    podman exec -it systemd-ollama ollama run phi4-mini "hello"

Watch nvidia-smi in another terminal while it responds — GPU memory
and utilisation should rise if passthrough is working correctly.

## GPU passthrough

This Quadlet passes through the host GPU via CDI
(AddDevice=nvidia.com/gpu=all), letting Ollama run inference on the
GPU instead of falling back entirely to CPU.

Requires a working NVIDIA driver on the host first (confirm with
nvidia-smi) — installation varies significantly by GPU generation and
distro, see your distro's NVIDIA/RPM Fusion documentation.

One non-obvious gotcha worth knowing: for older GPUs (Pascal
generation and earlier), the current default "open" NVIDIA kernel
module doesn't support the hardware — the driver installs but
nvidia-smi fails with a GSP-related error. Check lspci -k to confirm
which driver is bound, and use the correct legacy driver branch (on
Fedora/RPM Fusion, akmod-nvidia-580xx) if this applies to you.

## What I learned

- Quadlet-managed containers are internally named systemd-<filename>,
  not just <filename> — worth knowing when using podman exec directly
  against them rather than through systemd.
- Volume mount directories must exist on the host before the container
  starts — Podman won't create them automatically.
