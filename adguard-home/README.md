# AdGuard Home

Runs AdGuard Home as a rootless Podman container for local DNS
filtering and ad/tracker blocking, currently scoped to this laptop
only.

## Why Podman/Quadlet

Rootless containers matter here specifically since this service
touches all DNS traffic on the machine, and I wanted it managed
cleanly by systemd rather than a manually-run container.

## What it does

- Serves DNS on 127.0.0.1:53 and a web admin UI on 127.0.0.1:3000
- Blocklists and config persist in ~/.adguardhome

## Why bound to localhost only

Currently only used for DNS filtering on this laptop, so published
ports are restricted to 127.0.0.1 rather than all interfaces — no
service in this repo is exposed to the rest of the home network until
there's a genuine reason for it to be. When this eventually runs
network-wide from dedicated homelab infrastructure, that will be a
deliberate change, not a default.

## Setup

Clone the repo and move into this service's folder:

    git clone https://github.com/fionnlinux/homelab-services.git
    cd homelab-services/adguard-home

Fedora's systemd-resolved occupies port 53 by default, which conflicts
with AdGuard Home binding the same port. Disable its stub listener
first:

    sudo mkdir -p /etc/systemd/resolved.conf.d
    sudo tee /etc/systemd/resolved.conf.d/adguardhome.conf << 'RESOLVED'
    [Resolve]
    DNS=127.0.0.1
    DNSStubListener=no
    RESOLVED
    sudo mv /etc/resolv.conf /etc/resolv.conf.backup
    sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
    sudo systemctl reload-or-restart systemd-resolved

Rootless Podman can't bind to port 53 by default either (see
"Privileged port binding" below) — allow it:

    sudo sysctl -w net.ipv4.ip_unprivileged_port_start=53
    echo "net.ipv4.ip_unprivileged_port_start=53" | sudo tee /etc/sysctl.d/99-rootless-ports.conf

Then set up the Quadlet itself:

    mkdir -p ~/.config/containers/systemd
    cp adguardhome.container ~/.config/containers/systemd/
    mkdir -p ~/.adguardhome/work ~/.adguardhome/conf
    systemctl --user daemon-reload
    systemctl --user enable --now adguardhome.service

Complete setup at http://localhost:3000, then point your system's DNS
at 127.0.0.1:

    nmcli connection show
    nmcli connection modify "<connection-name>" ipv4.dns "127.0.0.1"
    nmcli connection modify "<connection-name>" ipv4.ignore-auto-dns yes
    nmcli connection up "<connection-name>"

Verify with:

    dig google.com

and check the query log at http://localhost:3000.

## What I learned

- Fedora's systemd-resolved occupies port 53 by default via a stub
  listener, which conflicts with any container trying to bind that
  port. Resolved by disabling DNSStubListener in a systemd-resolved
  drop-in config.
- Volume mount directories must exist on the host before the container
  starts.
- On Fedora with SELinux enforcing, volume mounts need the :Z flag so
  the container gets the correct SELinux label to actually read and
  write those directories.

## Privileged port binding (port 53)

Rootless containers can't bind to ports below 1024 by default. I
initially assumed AddCapability=NET_BIND_SERVICE in the Quadlet would
be a targeted fix for this. Testing showed that's incorrect, and it's
a documented, known limitation, not specific to this setup.

As of Podman 5.0, pasta is the default rootless networking backend.
The pasta process runs inside Podman's own user namespace to perform
port binding on the host's behalf — capabilities granted to the
containerized process do not extend to it, since namespace-scoped
capabilities don't cross back into the host's real namespace this
way.

Confirmed against Red Hat's own documentation and direct discussion
from the Podman maintainers: net.ipv4.ip_unprivileged_port_start is
the correct, supported fix for this limitation.

    sudo sysctl -w net.ipv4.ip_unprivileged_port_start=53
    echo "net.ipv4.ip_unprivileged_port_start=53" | sudo tee /etc/sysctl.d/99-rootless-ports.conf

AddCapability is kept in the Quadlet as it may still be relevant in
other contexts, but it does not solve privileged port binding under
rootless Podman with pasta.
