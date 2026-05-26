# Build Log

Notes from building out this stack on my homelab. I'm using this to record what I've done, so if I have to rebuild this, want to look back at why I made a decision or if anyone needs it theres a place to go.

I'd read through a lot of documentation and watched tutorials on Docker, Traefik, CI/CD, and self-hosting before any of this. I wanted to actually do something with all that theory, so this is where i'm doing it.

The whole thing runs on a Proxmox(on a spare laptop thats always running). One Ubuntu VM does the work, and Docker runs all the services inside that VM.

---

## Stage 0 — Planning and tool choices

Before building anything I had to decide what to use.

**Why a homelab.** I'm trying to move into DevOps, platform engineering or SRE work eventually, and reading about tools only gets me so far and to get my hands dirty. Actually see things break, fix them and undersatnding why.

**Why Proxmox + a VM, not just Docker on the laptop.** Proxmox lets me snapshot the whole VM and roll back if I really break something. I can also run more VMs later (a firewall, a monitoring stack). Treating the laptop as a hypervisor (type 1) instead of a Docker host gives me room to grow.

**Why Traefik.** Traefik reads labels off Docker containers and routes traffic automatically. Add a container with the right labels, Traefik picks it up so no config files to reload. I'd also read that Kubernetes works on similar ideas (ingress controllers), so this would build skills that transfer later. Nginx Proxy Manager was the easier option but it's more of a UI on top of nginx than a different way of thinking about routing.

**Why Gitea.** I wanted to actually understand what a Git server is under the hood instead of treating GitHub as a black box I would just click around in without meaningful understanding. Plus I needed something Woodpecker could integrate with for full self-hosted CI/CD.

**Why Postgres for Gitea.** SQLite is the default for Woodpeckerand it would have worked but I went with Postgres because running a separate database container teaches me the multi-container app pattern, and once Woodpecker starts sending webhooks to Gitea, having a that kind of database matters.

**Why Woodpecker for CI/CD.** Jenkins felt old. Drone (which Woodpecker was forked from) went commercial. GitHub Actions is free and fine but I wouldn't learn anything internals-wise from using it — that was kind of the point. Woodpecker is small enough to actually understand: there's a server, an agent, and a YAML file. I wanted to see and understand all three pieces.

---

## Stage 1 — Repo and project setup

Made a public `homelab-infra` repo on GitHub, cloned it locally, set up folders for each service I was planning to run: `traefik/`, `gitea/`, `woodpecker/`, `docs/`.

Wrote a `.gitignore` from scratch. None of GitHub's templates really fit since this isn't an application — it's infrastructure config. The gitignore covers `.env` files, secrets, Docker volumes, and the usual OS/editor junk. Used the `!.env.example` pattern so I could commit example env files for documentation while ignoring the real ones.

Decided on Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`) from the start. Looking at the log later, it actually does make a difference.

**Things still to do:**

- Add a real architecture diagram to the README. Right now there's an ASCII sketch.

---

## Stage 2 — VM on Proxmox

Made a Ubuntu 24.04 server VM on the Proxmox box. Minimized install, OpenSSH enabled at install time. Made a user account that doesn't share the username with the laptop I was working from so I can tell sessions apart.

Specs: 2 vCPU, 2 GB RAM, 20 GB disk. Bridged networking so the VM has its own IP on the LAN.

Set up SSH key auth from my dev machine with `ssh-copy-id`. After that I could log in without typing a password every time. Ran the usual `apt update && apt upgrade -y` and installed `curl`, `git`, `ca-certificates`.

**Things I picked up:** CPU type `host` in Proxmox passes the host's CPU features through to the VM. Better performance, but it ties the VM to that specific hardware. If I ever move Proxmox to different hardware I'd switch to something portable like `x86-64-v2-AES`. Not a problem right now but i will be doing this in the future.

**Things still to do:**

- The VM has a dynamic IP and I will give it a DHCP reservation in my router so the IP doesn't change on me.

---

## Stage 3 — Docker

Installed Docker on the VM from the official Docker apt repo. There's also a `docker.io` package in Ubuntu's default repos and a snap version, but the documentation and a bunch of reading I did warned about both — the apt version is usually old, and the snap has filesystem isolation that breaks volume mounts in unexpected ways. The official repo is the one to use.

Added my user to the `docker` group so I don't have to `sudo docker` every command. Had to log out and back in for the group change to take effect — group membership only refreshes on a new session.

Being in the `docker` group is basically equivalent to having root on the host machine, since you can mount the root filesystem into a container. That's fine for a homelab where I'm the only user, but it's not something to do on a shared system.

Created the shared Docker network that everything would use `docker network create treafik`

Every service's compose file references this network as `external: true`. That way Traefik and the services it routes to can see each other on the same network, and compose won't try to create a separate one per service.

---

## Stage 4 — Traefik

I used Traefik as the reverse proxy in front of everything else.

Traefik watches the Docker socket for events and when a container with the right labels starts, Traefik sees it and adds routing rules. When the container stops, Traefik drops the rules. No config files to reload manually; different from nginx, where adding a service usually means writing a config block and restarting.

The things to understad were **entrypoints, routers, middlewares and services.** An entrypoint is a port Traefik listens on. A router matches incoming requests (by hostname, path, etc.) and sends them to a service. A service points at backend containers. Middlewares sit between routers and services and can do things like add headers or require auth.

I skipped real domains and Let's Encrypt for and  edited `/etc/hosts` with Nano on my local machine to point `whoami.lab`, `traefik.lab`, `gitea.lab`, `woodpecker.lab` at the VM's IP. Plain HTTP for everything as SSL is its own future project.

I created a `whoami` test container to confirm routing worked. It's a tiny container that echoes HTTP request info back, which is perfect for confirming the request actually came through Traefik (you can see Traefik's forwarded headers in the response).

### What broke

`curl http://whoami.lab` came back with `404 page not found`. I assumed it was a routing or label problem — wrong network, typo in a label, wrong port. Spent a while checking those before reading the logs.

`docker compose logs --tail=50` (this shows the last 50 lines and continues watching new logs as they come in, using just `docker compose log` would show all logs in the containers lifecycle and potentially overload the terminal) on Traefik showed the actual problem which was that Traefik couldn't talk to the Docker daemon at all. The error was about the API version being too old. Ran `docker version` to confirm and the Docker daemon was on a recent release, but the Traefik image I'd pinned to `traefik:v3.1` shipped with an older Docker SDK that couldn't negotiate properly with the newer daemon.

Traefik couldn't see any containers at all and had no routers registered for `whoami.lab`, so any request to that hostname matched nothing and got the default 404.

First fix attempt was setting `DOCKER_API_VERSION` as an env var on the Traefik container but that didn't help because env var isn't honored by Traefik's Docker provider the way it is by the standalone Docker CLI.

The actual fix was changing the image to `traefik:v3` (rolling tag for the latest versions), which ships an updated SDK.

### What I figured out from this

The biggest one was to read the log files first; there was a lot of wasted time guessing before I figured that out. When something doesn't work, `docker compose logs --tail=50` will be the first thing I run for now.

The other thing was that pinning to a specific version isn't always safe. For production I'd pin to a specific version but for my homelab, a rolling tag isnt a worry.

**Things still to do:**

- Right now the Traefik dashboard is exposed on port 8080 without auth on a private LAN but I will route it through Traefik itself with an auth later.

---

## Stage 5 — Gitea

Set up Gitea as a self-hosted Git server with a separate Postgres container as the database.

Gitea needs to listen on two ports. HTTP for the web UI, which Traefik handles on port 80 like any other service and SSH for `git push`, which is trickier because the VM's real `sshd` is already on port 22. So I mapped Gitea's SSH container to port 222 on the host instead, and told Gitea via env vars to advertise port 222 in the clone URLs it shows in the web UI. That way the URL it gives me when I look at a repo is already correct

Gitea needs a database password so I had secrets to deal with. This is the pattern I settled on (and used everywhere after this):

- `gitea/.env.example` I committed with placeholder values, documents what env vars the service needs
- `gitea/.env` I never committed and has the real password which lives only on the VM

Generated the password on the VM with `openssl rand -base64 32` that produces random ASCII characters that I will never have to type because the compose file reads it from `.env`.

I also picked up a useful pattern for Gitea specifically where it reads a config file (`app.ini`) with sections like `[database]` and `[server]`. You can override any value in there with env vars using a `GITEA__section__KEY` convention.

### What broke

Smaller things in this stage, no big disasters.

`git init` defaulted to `master` instead of `main`. Tried to push `main` to Gitea and got `src refspec main does not match any` because the branch I'd committed on was actually `master`. Fixed it globally with `git config --global init.defaultBranch main` so future `git init`s start with `main`. For the repo I'd already created, `git branch -m master main` renamed it in place.

Also learned that branches don't really exist until you commit on them. With no commits, there's nothing for the branch name to point at so I got the same `src refspec` type error. Git treats branches as pointers to commits.

### What I figured out from this

- `depends_on` in compose only waits for a container to start, not for the service inside it to be ready. Postgres takes a couple seconds to actually accept connections after the container starts up and Gitea keeps retrying its connection, but if I ever need stricter ordering I'd add a healthcheck.
- The data folders (`./data` for Gitea, `./db-data` for Postgres)  survive `docker compose down`, but they don't back themselves up.

---

## Stage 6 — Woodpecker CI

Once Woodpecker was up and authenticated with Gitea, every `git push` would trigger a pipeline automatically. A full CI/CD loop

The architecture is three pieces:

- **Server** aka the brain that talks to Gitea via OAuth, receives webhooks, dispatches jobs, hosts the UI.
- **Agent** aka the worker that receives jobs from the server, spawns a fresh Docker container for each pipeline step, captures output.
- **`.woodpecker.yml`**  the pipeline file in each repo that describes what steps to run.

The server and agent communicate over gRPC with a shared secret and they both run on the same VM as everything else.

Same secrets pattern as Gitea. Three secrets needed in `.env`: the Gitea OAuth Client ID and Secret (from a new OAuth app I created in Gitea's admin UI), and a shared agent secret generated with `openssl rand -hex 32`.

This stage took a lot of debugging three separate things broke.

### What broke (1) — bind mount permissions

The server crashed on startup, looping with errors about not being able to open its SQLite database file and the agent couldn't connect to the server (because the server was dead), which made the agent fail too. Two separate errors, one root cause.

The Woodpecker server container runs as a non-root user (UID 1000 inside the container). The `./server-data` folder on the host had been created by Docker on first run, and since the Docker daemon runs as root, the folder was owned by root with permissions that didn't let UID 1000 write to it so the server couldn't create its database file in there, so it kept crashing.

Fixed it with `sudo chown -R <my-user>:<my-user> server-data`, refreshed and the database initialized.

Named Docker volumes don't have this problem because Docker manages ownership automatically. Bind mounts are useful when I want the data visible on the host filesystem (which I do — easier to back up that way).

### What broke (2) — OAuth's two URLs

After the server was healthy and I'd created the OAuth app in Gitea, I tried to log into Woodpecker. The browser bounced me to Gitea, I clicked authorize, got bounced back to Woodpecker — and Woodpecker showed an error: "error while authenticating against OAuth provider."

Server logs showed `oauth2 config exchange failed: dial tcp: lookup gitea.lab on 127.0.0.11:53: no such host`

The Woodpecker container couldn't resolve `gitea.lab` because `/etc/hosts` is on my local machine. It only knows what its DNS resolver can tell it.

I tried changing `WOODPECKER_GITEA_URL` to `http://gitea:3000` (the container name, which Docker's internal DNS resolves automatically within a shared network). The login flow then failed differently. The browser was now trying to redirect to `http://gitea:3000/login/oauth/authorize`. My browser doesn't know what `gitea` is beacuse that name only exists inside Docker.

What I missed was that OAuth involves three HTTP transactions, and they run in different network contexts.

1. Browser to Gitea (you visit the authorize URL); needs to work from the my laptop's network.
2. Gitea to the browser to Woodpecker (the redirect callback).
3. Woodpecker server to Gitea server (the token exchange that fetches the access token). Runs inside Docker on its own network.

Most OAuth tutorials I saw assumed those three contexts have the same DNS view but mine don't and Woodpecker only has one URL env var for "where is Gitea,".That var gets used for both the redirect URLs sent to the browser AND the server-side token exchange so both need to work.

To fix it I changed `WOODPECKER_GITEA_URL` back to `http://gitea.lab` (the external name, what the browser needs), and add `extra_hosts: ["gitea.lab:host-gateway"]` to the Woodpecker server in compose. That makes `gitea.lab` resolvable from inside the container by pointing to the host gateway (the VM), which routes through Traefik to Gitea. The browser sees `gitea.lab` and resolves via `/etc/hosts`, server sees `gitea.lab` and resolves via `extra_hosts`, and both reach Gitea.

The`host-gateway` is a Docker keyword that lets a container communicate with services running directly on the host machine's localhost(docker is isolated by default)and acts like a network bridge. I could have hardcoded the VM's IP but using `host-gateway`nothing breaks if I ever move the VM.

### What broke (3) — step containers had the same problem

The login worked. I created a `.woodpecker.yml` in a test repo, pushed it, and watched the pipeline trigger. It ran the `clone` step (which Woodpecker adds automatically before user steps) and immediately failed with the message `fatal: unable to access 'http://gitea.lab/...': Could not resolve host: gitea.lab`.
Every pipeline step runs in a fresh container the agent spawns, and those containers don't inherit anything from the agent (not `extra_hosts`and no network attachments).

To fix it I set an env var on the agent itself; `WOODPECKER_BACKEND_DOCKER_NETWORK=traefik`.
This parameter instructs the agent to attach every spawned container to the `traefik` Docker network ensuring that in every step that a container is spawned other services like gitea can resolve automatically through Docker's DNS.

I also added `gitea.lab:host-gateway` to the agent's `extra_hosts` as an extra.

After force-recreating the agent so it picked up the new env (`docker compose up -d --force-recreate`), I triggered a pipeline that worked.

### What I figured out from this

This whole stage came back to the same things, the services needs to be reachable by both a browser (which sees the LAN) and another container (which sees Docker's network), `/etc/hosts` on my laptop, `extra_hosts` on a container, agent-level networking for step containers. A DNS server (Pi-hole or AdGuard) would solve this instead of three separate fixes.

**Things still to do:**

- Add `when:` filters to the pipeline steps to clear the warning Woodpecker logs about steps not filtering by event type.
- Pin the Woodpecker image versions instead of `:v3`.

---

## Roadmap

1. **Pi-hole or AdGuard Home for local DNS.**

2. **Tailscale.** Access the homelab from outside my LAN without opening ports on my router.

3. **Real domain + Let's Encrypt SSL.**

4. **Backups.**  The data folders for Gitea, Postgres, and Woodpecker all live on the VM and nowhere else.

5. **Observability (Grafana, Prometheus, Loki.)**

6. **Migrate a real project to Gitea and CI.** `gitea-test` is just a repo with a hello-world pipeline so I will put one of my actual project repos through Gitea + Woodpecker (lint, build, test, deploy)

7. **Pin all the image versions.** Right now I have `:v3`, `:latest`, and other rolling tags scattered around. They were fine for getting unstuck, but they're a risk long-term.

8. **Lock down the Traefik dashboard.** Route it through Traefik with auth middleware instead of leaving it on an open port.

9. **SonarQube.** To Scan for security vulnerabilities

10. **HashiCorp Vault.** For secrets

---
