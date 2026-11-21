# Guests

Every virtual machine and container on `hv-01`, plus the one CI lane that lives off the host: what it is for,
what runs inside, what it may reach, how it is managed and whether it is backed up. Sizes are deliberately left
out; each guest's memory *policy* is described instead, in [HOST.md](HOST.md#memory-policy).

| Guest | Type | Purpose in one line |
| --- | --- | --- |
| [`edge-node`](#edge-node) | VM | The only machine the router forwards to: TLS, routing, rate limits, bans |
| [`infra-node`](#infra-node) | VM | The control plane that deploys and watches web apps, and the observability stack |
| [`git-node`](#git-node) | LXC | Git hosting and the server side of the Pushlane CI system |
| [`ci-runner`](#ci-runner) | VM | Runs CI jobs, sandboxed |
| [`ws-runner`](#ws-runner) | VM on a workstation | An optional CI lane that borrows a desktop's spare capacity |
| [`game-node`](#game-node) | VM | An on-demand game server |
| [`sandbox-ct`](#sandbox-ct) | LXC | Scratch space for experiments |

---

## edge-node

| | |
| --- | --- |
| **Type** | VM, ballooned |
| **Runs** | nginx (TLS termination, one site per public name, a catch-all `444`), certbot, fail2ban, the security alert and digest scripts |
| **Reachable from** | The internet on `:443` (the router's only forward); SSH from the LAN |
| **Reaches** | `git-node` (CI site, API, webhook service) and the deployed apps' sockets |
| **Managed by** | The control panel over SSH with a scoped `sudo` allowlist, and the admin CLI for the CI site |
| **Backups** | Nightly, plus the weekly second copy |

It is also the **data plane** for apps deployed by the control panel: each app gets its own gunicorn, Celery
worker and Celery beat units, a unix socket and an nginx site. Detail:
[Homelab-Control-Plane](https://github.com/tobioj/Homelab-Control-Plane).

Security on this box is described in [SECURITY.md](SECURITY.md); its routes in [NETWORK.md](NETWORK.md).

---

## infra-node

| | |
| --- | --- |
| **Type** | VM, ballooned |
| **Runs** | The deploy control panel (FastAPI, SQLite state, encrypted secrets), a shared Redis, the observability stack (Grafana, Loki behind an authenticating proxy, GlitchTip, ntfy), the tunnel client that publishes it, a rack and inventory app, and development services, plus timers for DNS, uptime probes and certificate renewal |
| **Reachable from** | The LAN; the observability services also through the outbound tunnel. The panel binds to a LAN address and requires a login |
| **Reaches** | `edge-node` over SSH (a single dedicated key); the tunnel provider, outbound |
| **Managed by** | systemd (`Restart=always`) for the panel, Docker Compose for the services, one stack per folder |
| **Backups** | Nightly, plus the weekly second copy |

The control plane is deliberately a separate machine from the apps it deploys: a bad deploy can never take
down the tool needed to fix it. Its design, safety model and incident log are in
[Homelab-Control-Plane](https://github.com/tobioj/Homelab-Control-Plane).

**Development services.** Apps that are still being built get their supporting services here too, for example
a PostgreSQL database for an app's development and test runs. They bind to the machine's own loopback address,
so nothing on the LAN can reach them directly; a developer's laptop and its coding agents reach them through an
SSH tunnel with key-based login. Each lives in its own Compose project and folder, with its own memory cap, so it
can be started, stopped or deleted without touching the observability stack next to it.

**The observability path.** Dashboards, error tracking, log ingest and push alerts are published through a
Cloudflare Tunnel: an outbound connection from this VM, so an app can report errors and ship logs from wherever
it runs, and a phone can receive alerts away from home, without any port being opened on the router. Each
service keeps its own authentication. See [NETWORK.md](NETWORK.md#the-second-way-in-an-outbound-tunnel).

---

## git-node

| | |
| --- | --- |
| **Type** | LXC container, hard memory cap |
| **Runs** | Forgejo v15 LTS (SQLite), the Pushlane API and results service (job logs, run results with reasons, re-run and cancel, setup progress), the GitHub sync service |
| **Reachable from** | `edge-node` (all public traffic), the runners (job fetches), the host (administration) |
| **Reaches** | GitHub (sync, commit statuses) |
| **Managed by** | The admin CLI on the host |
| **Backups** | Nightly, plus the weekly second copy |

This is the server half of **Pushlane**, a self-hosted CI system that sits in front of GitHub. People push to
Pushlane; tests run on the runners; only commits that pass are forwarded to GitHub. Highlights:

- **Accounts and plans.** Private repositories, 2FA for everyone, storage quotas, and two plans: standard and
  priority.
- **GitHub stays in step.** A GitHub App reports `pushlane/ci` as a commit status on GitHub, and copies merges
  and other GitHub-side commits back to Pushlane (fast-forward only, never forced, never deleted). A test
  exercises the whole round trip against a real GitHub repository.
- **Every outcome explained.** Forgejo records *that* a job was cancelled, not *why*; Pushlane works out the
  reason and reports it through its API and in the GitHub status.
- **Agent friendly.** A documented API, a Get started page whose checklist is read from the server's real state,
  and an agent guide whose every command is executed by a test suite.

Forgejo stays on the long-term-support line on purpose; features that only exist in newer releases wait for
the next LTS.

---

## ci-runner

| | |
| --- | --- |
| **Type** | VM, **fixed** memory, compressed swap (zram) inside |
| **Runs** | Two `forgejo-runner` processes (the **shared lane** and the **priority lane**), a shared Actions cache server, Docker |
| **Reachable from** | The host only (SSH) |
| **Reaches** | The internet, `git-node` and the edge proxy. Every other private range is dropped by its firewall |
| **Managed by** | Provisioning scripts and the admin CLI; rebuilt from scratch in minutes |
| **Backups** | None, by design: nothing on it is worth restoring |

How it is organised:

| Piece | Behaviour |
| --- | --- |
| **Shared lane** | A small number of jobs at once for every account and organisation on the standard plan. |
| **Priority lane** | A few jobs at once, shared by the accounts on the priority plan. They run only on priority lanes: when those are full, their jobs queue rather than spill onto the shared lane. |
| **Per-account runner identities** | Forgejo cannot keep an instance-wide runner away from some accounts, so each lane registers one identity per account it serves. A timer adds new accounts and drops deleted ones. |
| **Actions cache** | One cache server for both lanes, bound to localhost, entries isolated per repository. A cache saved on one lane is found on the other. |
| **Per-lane action folders** | Each lane keeps the actions it downloads in its own folder; two processes writing one folder collided. |
| **Job sandbox** | Unprivileged containers, Docker user-namespace remapping, no Docker socket in jobs, no host bind mounts, per-job CPU, memory and process limits, a 60-minute timeout. |
| **Graceful drains** | Changing a lane's configuration stops it taking new jobs and lets the running ones finish before it restarts. |

---

## ws-runner

| | |
| --- | --- |
| **Type** | A VM under a desktop hypervisor on a workstation, bridged onto the home network; not on `hv-01` |
| **Runs** | A third `forgejo-runner` lane for priority accounts only, its own Actions cache and forwarder image, Docker |
| **Reachable from** | The host only (SSH) |
| **Reaches** | The internet, `git-node` and the edge proxy's `:443`, enforced by a guard inside the VM |
| **Managed by** | The admin CLI on the host (build, hand over, connect, disconnect) and a Start-menu folder on the workstation |
| **Backups** | None: it is rebuilt from the same scripts as `ci-runner` |

**Why it exists.** The host's spinning disk caps CI throughput, and its memory can't be upgraded. A workstation
that is idle much of the day has a solid-state disk and spare cores. Rather than buy a second server, it lends
them to CI when it is switched on; measured, the same test suite ran two to three times faster there.

**How it stays trustworthy for the workstation's owner:**

- **Contained.** Everything lives in one folder, a Start-menu folder and a startup shortcut. Nothing changes the
  workstation's own settings, drivers or features.
- **Guarded from inside.** An nftables guard is loaded before Docker on every boot, and Docker will not start
  without it. A desktop firewall can't filter bridged VM traffic, which is why the guard lives in the VM.
- **Polite.** The desktop hypervisor gives the VM low priority whenever its window isn't in front.
- **In the owner's hands.** Pause CI (no new jobs, running ones finish, then the VM powers off), Resume CI,
  Stop CI now, CI status, and Remove, which deletes the VM and its shortcuts and nothing else.

**How the host works with it.** The VM takes its address from the router; the host finds it by its network
card's MAC address and checks it over SSH, because the workstation drops pings. By default the host's priority
lane keeps running alongside it, so priority accounts get both lanes' slots; a standby mode instead lets the host's
lane take over within a couple of minutes of the workstation going away. Shared jobs never run here, so the
shared lane never depends on whether a desktop is on.

---

## game-node

| | |
| --- | --- |
| **Type** | VM, large |
| **Runs** | A game server |
| **State** | Stopped by default; started by hand for play sessions |
| **Constraint** | Does not fit in memory alongside `ci-runner`. Stop the runner first; `ci-admin status` warns if both are up |
| **Backups** | None: the game state is replaceable |

---

## sandbox-ct

| | |
| --- | --- |
| **Type** | LXC container, small |
| **Runs** | Nothing permanent: a place to try packages and configurations before they reach a real guest |
| **State** | Stopped |
| **Backups** | None |
