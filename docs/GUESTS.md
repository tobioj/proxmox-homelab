# Guests

Every virtual machine and container on `hv-01`: what it is for, what runs inside, what it may reach, how it
is managed and whether it is backed up. Sizes are illustrative.

| Guest | Type | Purpose in one line |
| --- | --- | --- |
| [`edge-node`](#edge-node) | VM | The only internet-facing machine: TLS, routing, rate limits, bans |
| [`infra-node`](#infra-node) | VM | The control plane that deploys and watches web apps |
| [`git-node`](#git-node) | LXC | Git hosting and the server side of the Pushlane CI system |
| [`ci-runner`](#ci-runner) | VM | Runs CI jobs, sandboxed |
| [`game-node`](#game-node) | VM | An on-demand game server |
| [`sandbox-ct`](#sandbox-ct) | LXC | Scratch space for experiments |

---

## edge-node

| | |
| --- | --- |
| **Type / size** | VM · 2 vCPU · 4 GB, balloon floor 2 GB |
| **Runs** | nginx (TLS termination, one site per public name, catch-all `444`), certbot, fail2ban, the security alert and digest scripts |
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
| **Type / size** | VM · 4 vCPU · 8 GB, balloon floor 4 GB |
| **Runs** | The deploy control panel (FastAPI, SQLite state, encrypted secrets), a shared Redis, Grafana, Loki, GlitchTip and ntfy, plus timers for DNS, uptime probes and certificate renewal |
| **Reachable from** | LAN only; the panel binds to a LAN address and requires a login |
| **Reaches** | `edge-node` over SSH (a single dedicated key) |
| **Managed by** | systemd (`Restart=always`) and its own timers |
| **Backups** | Nightly, plus the weekly second copy |

The control plane is deliberately a separate machine from the apps it deploys: a bad deploy can never take
down the tool needed to fix it. Its design, safety model and incident log are in
[Homelab-Control-Plane](https://github.com/tobioj/Homelab-Control-Plane).

---

## git-node

| | |
| --- | --- |
| **Type / size** | LXC container · 2 vCPU · 2 GB hard cap |
| **Runs** | Forgejo v15 LTS (SQLite), the Pushlane API (job logs, re-run and cancel for people and their AI agents), the GitHub sync service |
| **Reachable from** | `edge-node` (all public traffic), the runner (job fetches), the host (administration) |
| **Reaches** | GitHub (sync, commit statuses) |
| **Managed by** | The admin CLI on the host |
| **Backups** | Nightly, plus the weekly second copy |

This is the server half of **Pushlane**, a self-hosted CI system that sits in front of GitHub. People push to
Pushlane; tests run on `ci-runner`; only commits that pass are forwarded to GitHub. Highlights:

- **Accounts and plans.** Private repositories, 2FA for everyone, storage quotas, and two plans: standard and
  priority.
- **GitHub stays in step.** A GitHub App reports `pushlane/ci` as a commit status on GitHub, and copies merges
  and other GitHub-side commits back to Pushlane (fast-forward only, never forced, never deleted). It is the
  newest piece: verified at the webhook level, with the end-to-end run against a live repository next.
- **Agent friendly.** A documented API and an agent guide whose every command is executed by a test suite.

Forgejo stays on the long-term-support line on purpose; features that only exist in newer releases wait for
the next LTS.

---

## ci-runner

| | |
| --- | --- |
| **Type / size** | VM · 8 vCPU · 12 GB **fixed** · 4 GB zram |
| **Runs** | Two `forgejo-runner` processes (the **priority lane** and the **shared lane**), a shared Actions cache server, Docker |
| **Reachable from** | The host only (SSH) |
| **Reaches** | The internet, `git-node` and the edge proxy. Every other private range is dropped by its firewall |
| **Managed by** | Provisioning scripts and the admin CLI; rebuilt from scratch in minutes |
| **Backups** | None, by design: nothing on it is worth restoring |

How it is organised:

| Piece | Behaviour |
| --- | --- |
| **Priority lane** | 3 jobs at once, shared by the accounts on the priority plan. They run only here: when both slots are full, they queue rather than spill onto the shared lane. |
| **Shared lane** | 2 jobs at once for every other account and organisation. |
| **Per-account runner identities** | Forgejo cannot keep an instance-wide runner away from some accounts, so the shared lane registers one identity per account. A timer adds new accounts and drops deleted ones. |
| **Actions cache** | One cache server for both lanes, bound to localhost, entries isolated per repository. A cache saved on one lane is found on the other. |
| **Per-lane action folders** | Each lane keeps the actions it downloads in its own folder; two processes writing one folder collided. |
| **Job sandbox** | Unprivileged containers, Docker user-namespace remapping, no Docker socket in jobs, no host bind mounts, per-job CPU, memory and process limits, a 60-minute timeout. |
| **Graceful drains** | Changing a lane's configuration stops it taking new jobs and lets the running ones finish before it restarts. |

Per job (illustrative): 4 vCPU, 4 GB memory plus 1 GB of zram swap, 60 minutes.

---

## game-node

| | |
| --- | --- |
| **Type / size** | VM · 6 vCPU · 20 GB |
| **Runs** | A game server |
| **State** | Stopped by default; started by hand for play sessions |
| **Constraint** | Does not fit in RAM alongside `ci-runner`. Stop the runner first; `ci-admin status` warns if both are up |
| **Backups** | None: the game state is replaceable |

---

## sandbox-ct

| | |
| --- | --- |
| **Type / size** | LXC container · 1 vCPU · 1 GB |
| **Runs** | Nothing permanent: a place to try packages and configurations before they reach a real guest |
| **State** | Stopped |
| **Backups** | None |
