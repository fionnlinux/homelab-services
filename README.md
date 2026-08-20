# homelab-services

Self-hosted services running on my homelab, each using whichever tool
actually fits that specific service — rootless Podman Quadlets for
now, possibly Docker Compose for others later. Part of my ongoing
homelab and Linux studies.

## Services

- [ollama](./ollama) — local LLM inference server, with GPU passthrough
- [adguard-home](./adguard-home) — network-wide DNS filtering, currently
  scoped to localhost only

More services will be added here as the homelab grows — see each
service's own README for why a particular tool was chosen for it, and
for real troubleshooting notes from getting each one working.
