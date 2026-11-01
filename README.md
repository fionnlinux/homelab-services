# homelab-services

Self-hosted services running on my homelab, each using whichever tool
actually fits that specific service — rootless Podman Quadlets for
now, possibly Docker Compose for others later. Part of my ongoing
homelab and Linux studies.

## Services

| Service | What it does |
|---------|--------------|
| [ollama](./ollama) | Local LLM inference server, with GPU passthrough |
| [adguard-home](./adguard-home) | Network-wide DNS filtering, currently scoped to localhost only |

More services will be added here as the homelab grows — see each
service's own README for why a particular tool was chosen for it, and
for real troubleshooting notes from getting each one working.

## How these run

- **Rootless Podman**, with each service defined as a Quadlet and
  managed natively by systemd
- **Localhost only by default**, until there is a genuine reason to
  expose a service to the wider network
- **Real troubleshooting documented** in each service's README, such
  as rootless port binding under Podman's pasta networking and SELinux
  container labelling

## Mirror

This repository is mirrored to
[Codeberg](https://codeberg.org/fionnlinux/homelab-services).
