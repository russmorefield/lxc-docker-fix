# Fixing `net.ipv4.ip_unprivileged_port_start` & AppArmor Docker Errors in a Proxmox LXC

This guide documents the exact steps taken to fix the following Docker errors when running **inside an unprivileged Proxmox LXC** on **Ubuntu 24.04.3 LTS**:

- `open sysctl net.ipv4.ip_unprivileged_port_start file: reopen fd 8: permission denied: unknown`
- `Could not check if docker-default AppArmor profile was loaded: open /sys/kernel/security/apparmor/profiles: permission denied`

Environment:

- **Hypervisor:** Proxmox VE (LXC container)
- **CT OS:** Ubuntu 24.04.3 LTS (Noble Numbat)
- **LXC:** Unprivileged container
- **Docker:** 28.5.2
- **Initial containerd:** 1.7.29-1~ubuntu.24.04~noble
- **runc:** 1.3.3

> Goal: Get Docker working reliably inside the LXC (including `docker run hello-world` and `docker compose up`) by downgrading containerd and disabling Docker’s AppArmor integration inside the container.

---

## 1. Symptoms

From inside the LXC container, any attempt to start a container failed with:

```bash
docker run hello-world
```

Error:

```text
Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: open sysctl net.ipv4.ip_unprivileged_port_start file: reopen fd 8: permission denied: unknown
```

After some changes, this evolved into:

```text
Could not check if docker-default AppArmor profile was loaded: open /sys/kernel/security/apparmor/profiles: permission denied
```

This indicated two separate but related issues:

1. A **containerd / runc bug** around `net.ipv4.ip_unprivileged_port_start` in nested/containerized environments.
2. **AppArmor access problems** inside an LXC when Docker tries to check or use AppArmor profiles.

---

## 2. Fix on the Proxmox Host (LXC Configuration)

On the **Proxmox host**, edit the LXC configuration for the container running Docker.

Example CTID: `117`

```bash
nano /etc/pve/lxc/117.conf
```

Add (or ensure) the following lines exist:

```ini
features: keyctl=1,nesting=1
lxc.apparmor.profile: unconfined
```

- `nesting=1` and `keyctl=1` are required for Docker-in-LXC to behave properly.
- `lxc.apparmor.profile: unconfined` relaxes AppArmor constraints for this particular container (expected decision for a “Docker host” CT).

After saving the file, restart the LXC from the Proxmox host:

```bash
pct restart 117
```

Then log back into the container.

---

## 3. Downgrade `containerd.io` Inside the LXC

Inside the container, we observed the available versions of `containerd.io`:

```bash
apt list -a containerd.io
```

Output snippet (Ubuntu 24.04 / noble):

```text
containerd.io/noble,now 1.7.29-1~ubuntu.24.04~noble amd64 [installed]
containerd.io/noble 1.7.28-2~ubuntu.24.04~noble amd64
containerd.io/noble 1.7.28-1~ubuntu.24.04~noble amd64
containerd.io/noble 1.7.28-0~ubuntu.24.04~noble amd64
...
```

We downgraded to the known-good version `1.7.28-1~ubuntu.24.04~noble`:

```bash
apt-get update

apt-get install -y   containerd.io=1.7.28-1~ubuntu.24.04~noble

apt-mark hold containerd.io
```

> `apt-mark hold` prevents automatic upgrades back to the problematic version during future `apt upgrade` runs.

Restart Docker to pick up the downgraded containerd:

```bash
systemctl restart docker
```

At this point, the original `net.ipv4.ip_unprivileged_port_start` error was resolved, but Docker still failed due to AppArmor being partially visible but unusable inside the LXC.

---

## 4. Disable Docker’s AppArmor Integration Inside the LXC (systemd override)

Even with `lxc.apparmor.profile: unconfined`, Docker still tried to interact with AppArmor and failed to read `/sys/kernel/security/apparmor/profiles`. To fix that, we told Docker **inside the container** to treat itself as running inside another container and skip AppArmor integration entirely.

Create a systemd drop-in for the Docker service:

```bash
systemctl edit docker
```

This opens an override file (e.g. `/etc/systemd/system/docker.service.d/override.conf`). Put the following content in it:

```ini
[Service]
Environment=container="setmeandforgetme"
```

Save and exit.

Then reload systemd and restart Docker:

```bash
systemctl daemon-reload
systemctl restart docker
```

Confirm that Docker no longer lists AppArmor as a security option:

```bash
docker info | grep -i apparmor || echo "no apparmor entry"
```

Expected output:

```text
no apparmor entry
```

---

## 5. Verification

Now test Docker again inside the LXC:

```bash
docker run hello-world
```

Expected output (successful run):

```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

If using Docker Compose (v2 plugin), you can now run your stack normally, for example:

```bash
cd ~/beszel
docker compose up -d
```

Check running containers:

```bash
docker ps
```

Example output:

```text
CONTAINER ID   IMAGE                                COMMAND                  CREATED          STATUS          PORTS                                         NAMES
9897c2825b2a   henrygd/beszel                       "/beszel serve --htt…"   X minutes ago    Up X minutes    0.0.0.0:8090->8090/tcp, [::]:8090->8090/tcp   beszel
51f064b0af18   ghcr.io/mbecker20/periphery:latest   "periphery"              4 months ago     Up X minutes    0.0.0.0:8120->8120/tcp, [::]:8120->8120/tcp   komodo-periphery-periphery-1
```

At this point, Docker is fully functional again inside the Proxmox LXC.

---

## 6. Optional: Pin Docker Packages Too

If Docker itself was recently upgraded and you want to stabilize the environment further, you can also hold the Docker Engine packages (if installed from Docker’s official repo):

```bash
apt-mark hold docker-ce docker-ce-cli docker-ce-rootless-extras
```

This ensures that containerd and Docker remain on known-good versions until you explicitly decide to upgrade and re-test.

---

## 7. Security Considerations

In this setup:

- The LXC is **unprivileged**, which already provides a boundary.
- `lxc.apparmor.profile: unconfined` relaxes AppArmor at the LXC level so Docker can operate normally in a nested environment.
- Docker’s AppArmor integration is **disabled** inside this LXC via the `Environment=container="setmeandforgetme"` override.

This combination is a reasonable trade-off for a **dedicated “Docker host” LXC in a homelab environment**, but you should:

- Avoid running highly untrusted containers in this CT.
- Keep it segregated from more sensitive services if possible.
- Periodically review Docker and containerd updates and re-evaluate whether upstream has fixed the underlying issues so you can remove workarounds in the future.

---

## 8. Quick Summary (Copy/Paste Checklist)

On **Proxmox host** (`/etc/pve/lxc/117.conf`):

```ini
features: keyctl=1,nesting=1
lxc.apparmor.profile: unconfined
```

Then:

```bash
pct restart 117
```

Inside the **LXC**:

```bash
# Downgrade and pin containerd
apt-get update
apt-get install -y containerd.io=1.7.28-1~ubuntu.24.04~noble
apt-mark hold containerd.io
systemctl restart docker

# Disable Docker AppArmor integration via systemd override
systemctl edit docker
# (add this)
# [Service]
# Environment=container="setmeandforgetme"

systemctl daemon-reload
systemctl restart docker

# Verify
docker info | grep -i apparmor || echo "no apparmor entry"
docker run hello-world
```

If those commands succeed, Docker is healthy again and `docker compose up -d` should work for your stacks.

---

> This file is intended for GitHub documentation under a homelab / Proxmox / Docker so future you (or others) can quickly reproduce the fix when Docker breaks in nested LXC environments on Ubuntu 24.04.
