Game Server Infrastructure

Self-hosted Minecraft server running Kubernetes, exposed to the internet via a reverse tunnel to an Oracle Cloud VPS (no port forwarding or exposed home IP required)

File Layout
├── terraform/          Oracle Cloud VPS — fully automated with cloud-init
│   ├── TODO: Fully automate deployment of Oracle Cloud VPS
│
├── vps/               Config files on the host for reverse tunnel connectivity
|   ├── frps.service
|   ├── frps.toml
│
└── k3s/                Kubernetes manifests
    ├── backup-pvc.yaml            PVC for saving backups              
    ├── debug.yaml                 pod attached to world save PVC for debugging server issues 
    ├── frpc-config.yaml           frpc tunnel client config
    ├── frpc-deployment.yaml       reverse tunnel deployment on host  
    ├── pvc.yaml                   current running server files
    ├── server-deployment.yaml     game server deployment     
    ├── service.yaml               ClusterIP service creation exposing 25565
    ├── sidecar.yaml               Pod running fileserver and tailscale for remote file access
    └── TODO: Add automated backups for the ability to rollback to previous version if need be


Requirements
A machine running k3s
Oracle Cloud Account
Tailscale account + auth key (for fileserver pod)
kubectl installed


Order of Deployment

1. Provision VPS (Ideally in the future this can all be done with one terraform apply)
   - Spin up an Oracle VM (Always Free will do for a VM used as a VPS)
   - Open ports 25565 and 7000 on Oracle VM to enable connectivity between the server and VM
     sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 7000 -j ACCEPT
     sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 25565 -j ACCEPT
     sudo netfilter-persistent save
   - Create the frp tunnel
   - run on VPS:
     wget https://github.com/fatedier/frp/releases/download/v<VERSION>/frp_<VERSION>_linux_arm64.tar.gz
     tar -xzf frp_<VERSION>_linux_arm64.tar.gz
     cd frp_<VERSION>_linux_arm64
   - Create frps.toml and add:
     bindPort = 7000
     auth.method = "token"
     auth.token = "INSERT A RANDOM LONG STRING" # This is a token that enables the connectivity between the server and VPS
     allowPorts = [
     { start = 25565, end = 25565 }
     ]

   - Run it as a systemd service so it survives reboots — create /etc/systemd/system/frps.service and add:
    [Unit]
    Description=frp server
    After=network.target
    
    [Service]
    Type=simple
    ExecStart=/home/ubuntu/frp_<VERSION>_linux_arm64/frps -c /home/ubuntu/frp_<VERSION>_linux_arm64/frps.toml
    Restart=always
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target

   2. Make necessary changes to kubernetes manifests
   - In frpc-config.yaml, add the public ip corresponding with the Oracle VM in the serverAddr spot and add the created           token in the auth.token
  
  Run kubectl apply -f frpc-config.yaml -f frpc-deployment.yaml to verify successful connection between the VPS and server
  Run kubectl apply on the k3s files to create all the other necessary resources
