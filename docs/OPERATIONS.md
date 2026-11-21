# Operations

Runbooks for the things that actually come up. Almost everything goes through one admin CLI on the host,
`ci-admin`, so the same command works whether I run it or a script does.

---

## Contents

- [Daily health check](#daily-health-check)
- [Changing CI concurrency without killing jobs](#changing-ci-concurrency-without-killing-jobs)
- [Sizing the lanes from data](#sizing-the-lanes-from-data)
- [The workstation lane](#the-workstation-lane)
- [Accounts and plans](#accounts-and-plans)
- [The lane sync timer](#the-lane-sync-timer)
- [The network card hung](#the-network-card-hung)
- [After an unclean power-off](#after-an-unclean-power-off)
- [Rebuilding the runner](#rebuilding-the-runner)
- [Certificates](#certificates)
- [Security](#security)
- [Test suites](#test-suites)

---

## Daily health check

```bash
ci-admin status
```

One line per moving part, `[ok]` or `[!!]` with the command that fixes it (sample output):

```
[ok] Forgejo is up
[ok] lane sync runs every minute (new accounts get the shared lane)
[ok] shared lane up (standard accounts and organisations)
[ok] priority lane up (priority accounts), running alongside the workstation lane
[ok] workstation lane up (priority accounts)
[ok] actions cache servers up
[ok] GitHub sync and pushlane/ci statuses up
[ok] https://example-ci.duckdns.org answers through edge-node
[ok] certificate valid for 74 more days (auto-renews)
[ok] runner disk 22% used
[ok] host memory available
[ok] last backup: last night
```

It also warns when `game-node` is running, because it and the CI runner do not fit in memory together.
`ci-admin jobs` lists what is running and what is waiting, per lane.

---

## Changing CI concurrency without killing jobs

```bash
# edit the slot counts in the config, then:
ci-admin apply-lanes            # or: apply-lanes shared | apply-lanes priority
```

Each lane is restarted **gracefully**: it stops taking new jobs, lets the running ones finish (up to the job
time limit) and then comes back with the new setting. Jobs that arrive meanwhile wait in the queue. The status
line says `applies its new config when its running jobs finish` until the switch has happened.

---

## Sizing the lanes from data

Slot counts are set from measurements, not guesses:

```bash
job-stats start 24              # record every job on every lane for the next 24 hours
job-stats report OWNER[/REPO]   # per job: which runner, how long, real and peak memory, CPU time, throttling
```

It never reads job logs, only the containers' resource accounting. The question it answers is "would one more
slot have fitted?", and on spinning disks the answer usually comes from I/O pressure rather than from memory.

---

## The workstation lane

An optional third lane, on a VM on a desktop workstation (see [GUESTS.md](GUESTS.md#ws-runner)).

| Task | Where | How |
| --- | --- | --- |
| Build the VM image and the setup script | Host | `ci-admin pc build` |
| Offer the setup files to the workstation for a short window | Host | `ci-admin pc serve`, then one command on the workstation |
| Take the VM over once setup has started it | Host | `ci-admin pc connect` (installs the runner, the guard, the cache and the forwarder image, then starts the lane) |
| Pause, resume, stop, check | Workstation | The Pushlane CI folder in the Start menu |
| Retire it | Host, then workstation | `ci-admin pc disconnect`, then **Remove Pushlane CI** |

**Pause** means no new jobs, let the running ones finish, then power the VM off; nothing is lost. While the
workstation is off, the host's priority lane carries priority jobs on its own, so nothing has to be done on the
host when the desktop sleeps.

If the VM boots but the host can't find it, check the desktop hypervisor's bridged network first: it must be
bridged to the workstation's physical network card, not left on "automatic"
([EDGE-CASES.md](EDGE-CASES.md#16-a-bridged-vm-on-the-wrong-network)).

---

## Accounts and plans

| Task | Command |
| --- | --- |
| Invite someone (prints a welcome message to send privately) | `ci-admin invite USER EMAIL [--priority]` |
| Move an account between plans | `ci-admin plan USER standard\|priority` |
| See everyone's plan, storage and repositories | `ci-admin plans` |
| After deleting or renaming someone in the web UI | `ci-admin plan-cleanup` |
| Lock someone out without deleting anything | `ci-admin disable USER` |

A plan change moves the account to its new lanes **before** removing it from the old one, so it is never left
without somewhere to run.

---

## The lane sync timer

Every minute a host timer runs `ci-admin sync-lanes`:

1. Lists every account and organisation from the Git server.
2. Registers a shared-lane identity for anything new; retires the identity of anything deleted or moved to
   the priority plan.
3. Rewrites a lane's configuration and restarts it **only if something changed**. An unchanged run takes a few
   seconds and touches nothing.
4. Deletes a retired identity only once its lane is running a configuration without it, so a job in flight on
   that identity is never cut off.
5. Checks whether the workstation lane is up, and, in standby mode, switches the host's priority lane on or off
   to match.

Tests that create temporary accounts call the same sync directly instead of waiting for the timer.

---

## The network card hung

Symptoms: the host and every guest are unreachable, but the console is alive and the kernel log repeats
`Detected Hardware Unit Hang`.

```bash
ip link set eno1 down; ip link set eno1 up        # from the console: usually enough
ethtool -k eno1 | grep -E 'segmentation|receive-offload'   # confirm the offloads are still off
```

The offloads are disabled permanently by a `post-up` line on the interface, so this should not recur. If the
console is unresponsive too, one short press of the power button starts a clean shutdown; give the guests a
couple of minutes before cutting power.

---

## After an unclean power-off

| Check | How | Healthy result |
| --- | --- | --- |
| What happened | `journalctl -b -1 -n 200` | The last lines before the cut: a hang, a power-key press, or nothing |
| Guests came back | `qm list`, `pct list` | Everything that should run is running; the game server stays stopped |
| Disks | `smartctl -H` on both disks, reallocated and pending sectors | `PASSED`, zero bad sectors |
| Filesystems | `journalctl -b -k \| grep -i ext4` | Orphan cleanup and `recovery complete`, nothing worse |
| Git database | `sqlite3 forgejo.db "pragma quick_check;"` | `ok` |
| The platform | `ci-admin status` | All `[ok]` |
| Interrupted CI jobs | Nothing to do | The Git server fails them once their runner has been silent too long, the results service records why, and people re-run them |

---

## Rebuilding the runner

```bash
ci-admin rebuild-runner
```

Destroys the CI VM and builds a fresh one: OS, Docker with user-namespace remapping, zram, the runner, the
cache server, both lanes, the firewall and the daily image and cache cleanup. Runner identities are reused from
the host, so nothing has to be re-registered. Job images are pulled in the background afterwards.

For a configuration change on a running runner, `ci-admin provision-runner` updates it in place instead.

---

## Certificates

Renewal is automatic (a daily timer, with a randomised delay). By hand:

```bash
ci-admin cert          # issue or renew the CI site's certificate over DNS-01
```

The status check flags any certificate within two weeks of expiry.

---

## Security

| Task | Command |
| --- | --- |
| See who is blocked, per jail | `ci-admin blocked` |
| Block an address on all ports for a week | `ci-admin block IP` |
| Remove an address from every list | `ci-admin unblock IP` |
| Reinstall the jails, alerts and digest | `ci-admin security` |

---

## Test suites

Each runs against temporary accounts and cleans up after itself.

| Test | Proves |
| --- | --- |
| `lane-test` | Priority accounts run only on their lanes, everyone else only on the shared lane, each within its limit |
| `cache-test` | A cache saved on one lane is restored on the other; other repositories never see it |
| `action-fetch-test` | Jobs starting together on both lanes download their actions cleanly |
| `logs-api-test` | Logs, re-run and cancel through the API, and every refusal case |
| `reasons-test` | Cancel reasons are recorded and reported; runs that can never finish are cancelled with a reason |
| `agents-doc-test` | Every command in the agent guide works exactly as documented |
| `github-test` | GitHub sync and the `pushlane/ci` status, against a real GitHub test repository |
| `smoke-test` | Push, pass and fail, forward to GitHub, end to end |
| Fresh-agent run | A new AI agent, given only what a user would give it, sets up a project in a sandbox; its report shows where the docs fall short |
