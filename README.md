# Game Server Infrastructure

Self-hosted Minecraft server running on Kubernetes, exposed to the internet via a reverse tunnel to an Oracle Cloud VPS — no port forwarding or exposed home IP required.

---

## File Layout

```
├── terraform/                  # Oracle Cloud VPS — fully automated with cloud-init
│   ├── TODO: Fully automate deployment of Oracle Cloud VPS
│
├── vps/                        # Config files on the VPS host for reverse tunnel connectivity
│   ├── frps.service
│   └── frps.toml
│
└── k3s/                        # Kubernetes manifests
    ├── backup-pvc.yaml         # PVC for saving backups
    ├── debug.yaml              # Pod attached to world save PVC for debugging server issues
    ├── frpc-config.yaml        # frpc tunnel client config
    ├── frpc-deployment.yaml    # Reverse tunnel deployment on host
    ├── pvc.yaml                # Current running server files
    ├── server-deployment.yaml  # Game server deployment
    ├── service.yaml            # ClusterIP service exposing port 25565
    ├── sidecar.yaml            # Pod running fileserver and Tailscale for remote file access
    └── TODO: Add automated backups for rollback capability
```

---

## Requirements

- A machine running [k3s](https://k3s.io/)
- [Oracle Cloud](https://www.oracle.com/cloud/) account
- [Tailscale](https://tailscale.com/) account + auth key (for the fileserver pod)
- `kubectl` installed

---

## Deployment Order

### 1. Provision the VPS

> Goal: Automate this entirely with a single `terraform apply` in the future.

Spin up an Oracle VM — the Always Free tier is sufficient for a VPS.

---

### 2. Open Required Ports on the Oracle VM

```bash
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 7000 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 25565 -j ACCEPT
sudo netfilter-persistent save
```

---

### 3. Set Up the frp Reverse Tunnel

#### Download and extract frp on the VPS

```bash
wget https://github.com/fatedier/frp/releases/download/v<version>/frp_<version>_linux_arm64.tar.gz
tar -xzf frp_<version>_linux_arm64.tar.gz
cd frp_<version>_linux_arm64
```

#### Create `frps.toml`

```toml
bindPort = 7000

auth.method = "token"
auth.token = "INSERT_A_RANDOM_LONG_STRING"  # Shared secret between the server and VPS

allowPorts = [
  { start = 25565, end = 25565 }
]
```

#### Create `/etc/systemd/system/frps.service`

Run frps as a systemd service so it survives reboots:

```ini
[Unit]
Description=frp server
After=network.target

[Service]
Type=simple
ExecStart=/home/ubuntu/frp_<version>_linux_arm64/frps -c /home/ubuntu/frp_<version>_linux_arm64/frps.toml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

#### Enable and start the service

```bash
sudo systemctl enable frps
sudo systemctl start frps
```

---

### 4. Configure the Kubernetes Manifests

In `frpc-config.yaml`, fill in:

- `serverAddr` — the public IP of your Oracle VM
- `auth.token` — the token you set in `frps.toml`

#### Verify tunnel connectivity

```bash
kubectl apply -f frpc-config.yaml -f frpc-deployment.yaml
```

Check logs to confirm a successful connection between the VPS and server.

#### Apply remaining manifests

```bash
kubectl apply -f k3s/
```

This creates the PVC, game server deployment, ClusterIP service, sidecar pod, and all other resources.

---

## TODOs

- [ ] Fully automate VPS provisioning with Terraform + cloud-init (single `terraform apply`)
- [ ] Add automated backups with rollback support
