# homelab-infra
Self-hosted infrastructure running on a Proxmox server. A learning project for DevOps: reverse proxying, self-hosted Git, CI/CD pipelines, and infrastructure-as-code.

# Stack
Proxmox VE — hypervisor, running on a dedicated laptop
Ubuntu 24.04 VM — Docker host
Traefik — reverse proxy, routes traffic to services
Gitea — self-hosted Git server
Woodpecker CI — CI/CD pipelines triggered by Gitea pushes

# Status
In Progress (see commit history for build log.)

# Architecture



## 🏠 Homelab Architecture

```text
╔════════════════════════════════════════════════════╗
║                 🏠 HOMELAB STACK                  ║
║        Proxmox → Ubuntu VM → Docker Services      ║
╚════════════════════════════════════════════════════╝

🖥️  Proxmox Host
│
└── 🐧 Ubuntu 24.04 VM
    │
    └── 🐳 Docker
        │
        ├── 🌐 Traefik              Reverse Proxy :80 / :8080
        ├── 🦊 Gitea                Self-hosted Git, SSH :222
        ├── 🐘 Postgres             Database backend
        ├── 🪵 Woodpecker Server    CI/CD server
        ├── 🛠️ Woodpecker Agent     Pipeline runner
        └── 🧪 whoami               Routing test service

| Service              | Purpose                   | Notes                          |
| -------------------- | ------------------------- | ------------------------------ |
| 🖥️ Proxmox          | Bare-metal virtualization | Running on spare laptop        |
| 🐧 Ubuntu 24.04 VM   | Docker host               | Main server VM                 |
| 🐳 Docker            | Container runtime         | Runs all homelab services      |
| 🌐 Traefik           | Reverse proxy             | Ports `80` and `8080`          |
| 🦊 Gitea             | Self-hosted Git           | SSH exposed on port `222`      |
| 🐘 Postgres          | Database                  | Backend for Gitea              |
| 🪵 Woodpecker Server | CI/CD server              | Connects to Gitea              |
| 🛠️ Woodpecker Agent | CI/CD runner              | Executes pipelines             |
| 🧪 whoami            | Test container            | Used to verify Traefik routing |
