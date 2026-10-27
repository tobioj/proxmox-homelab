# Edge cases

Real failures, and what changed because of them. Each is written as **symptom, cause, fix, lesson**. Counts and
sizes are illustrative.

| # | Area | One line |
| --- | --- | --- |
| [1](#1-concurrent-ci-jobs-saturating-a-spinning-disk) | Capacity | Too many CI jobs at once made Docker stop answering |
| [2](#2-two-runner-processes-sharing-one-action-cache) | CI | Two lanes corrupted each other's downloaded actions |
| [3](#3-a-runner-that-takes-every-accounts-jobs) | CI | Forgejo cannot keep a runner away from some accounts |
| [4](#4-a-lane-that-refuses-to-start) | CI | A deleted account stopped a whole lane from starting |
| [5](#5-reconfiguring-a-lane-without-killing-jobs) | Operations | Restarting a lane used to cancel its jobs |
| [6](#6-the-onboard-network-card-hanging-under-load) | Hardware | The NIC locked up and took every guest offline |
| [7](#7-an-unclean-power-off) | Hardware | Recovering from a hard power cut mid-job |
| [8](#8-a-router-that-cannot-hairpin) | Network | The public name was unreachable from inside |
| [9](#9-an-isp-that-blocks-port-80) | Network | HTTP certificate validation was impossible |
| [10](#10-fail2ban-versus-webhooks) | Security | A wrong secret could have banned GitHub |
| [11](#11-logs-that-survive-a-rename) | API | Renaming an account hid its CI logs |
| [12](#12-a-job-name-with-an-em-dash) | API | One character turned a log download into a 502 |
| [13](#13-a-run-marked-failed-while-jobs-still-run) | API | Re-run and cancel disagreed with reality |
| [14](#14-two-guests-that-cannot-share-the-ram) | Capacity | The game server and CI do not fit together |

---

## 1. Concurrent CI jobs saturating a spinning disk

**Symptom.** Jobs failed with `context deadline exceeded` and database services that "never became healthy".
The failures moved between jobs and had nothing to do with the code under test.

**Cause.** Ten jobs ran at once, several with Postgres service containers. On a spinning disk that is ten-plus
concurrent random writers. I/O pressure inside the runner reached about 70%; Docker answered its API so slowly
that health checks and the runner's own calls timed out.

**Fix.** Concurrency came down in measured steps, from 5 + 5 to 3 + 3 to 3 + 2. Test databases now keep their
data on `tmpfs`, so they start in seconds and never touch the disk; the agent guide shows the exact service
definition. Restarts that apply new limits never kill running jobs (see [5](#5-reconfiguring-a-lane-without-killing-jobs)).

**Lesson.** Find the resource that actually saturates. Here it was never CPU or RAM.

---

## 2. Two runner processes sharing one action cache

**Symptom.** Jobs failed before their first step while downloading `actions/checkout`:
`shallow file has changed since we read it`, or `shallow.lock: File exists`. Only when several jobs started in
the same second.

**Cause.** Both lanes are separate runner processes running as the same Unix user, so both used that user's
default action folder. Each process serialises fetches between *its own* jobs; nothing serialised them across
processes.

**Fix.** Each lane keeps its actions in its own folder. Git's automatic cleanup after a fetch now runs in the
foreground, inside the runner's lock, instead of detaching into the background. A test starts two temporary
runners and 4 jobs at once, several rounds: with one shared folder it reproduces the failure, with a folder
each every job passes.

**Lesson.** A lock that lives in a process protects nothing from a second process. Look for every piece of
per-user state when you add a second instance.

---

## 3. A runner that takes every account's jobs

**Symptom.** The priority accounts were meant to run only on their own lane, but their jobs also landed on the
shared lane whenever it polled first.

**Cause.** Forgejo hands a job to any runner whose labels match. An instance-wide runner matches every account,
and there is no way to exclude some.

**Fix.** The shared lane no longer registers once for everyone. It holds one owner-scoped identity per account
and organisation not on the priority plan. A timer keeps that list current: new accounts and organisations get
an identity within two minutes, deleted ones are dropped, and a lane is restarted only when its list changed.
A test confirms a priority job waits for a priority slot while the shared lane sits idle.

**Lesson.** When the platform cannot express a rule, change what you register rather than what you ask it.

---

## 4. A lane that refuses to start

**Symptom.** After an account was deleted, the next restart of a lane failed in a loop.

**Cause.** A runner process announces each of its identities to the server at start-up, and exits if any one
of them is rejected. Deleting an account deletes its runner identities, so the lane's own config now named one
that no longer existed.

**Fix.** Before a lane config is written, its identities are checked against the server and dead ones are left
out. Identities that a lane stops using are only deleted once that lane runs a config without them, so a job
in flight on one is never cut off.

**Lesson.** Ordering matters in both directions: never reference what might be gone, never delete what might
be in use.

---

## 5. Reconfiguring a lane without killing jobs

**Symptom.** Applying a new concurrency setting cancelled whatever was running.

**Cause.** The runner's default shutdown grace period was short, and systemd's stop timeout shorter still.

**Fix.** The runner's shutdown timeout equals the job time limit, systemd waits longer than that, and restarts
are queued without blocking. A lane stops taking new jobs, finishes its running ones, then comes back with the
new config. The status check reports `applies its new config when its running jobs finish` meanwhile.

**Lesson.** "Restart" should mean "drain, then restart" anywhere work is in flight.

---

## 6. The onboard network card hanging under load

**Symptom.** The host, every guest and every site went unreachable at once. The console still worked.

**Cause.** The onboard Intel NIC (e1000e driver) logged `Detected Hardware Unit Hang` about 400 times in ten
minutes while two browser-test jobs moved a lot of data. This driver is known to stall its transmit queue under
heavy traffic with segmentation offload enabled.

**Fix.** TCP segmentation, generic segmentation and generic receive offload are off: applied live without a
link drop, persisted as a `post-up` line on the interface, recorded in the docs with the backup of the old
config. The runbook now says how to bring the link back from the console without a reboot.

**Lesson.** "The server crashed" was the wrong diagnosis: the OS ran the whole time. Read the last log lines
before concluding anything.

---

## 7. An unclean power-off

**Symptom.** After the NIC hang, the machine was powered off hard, mid-job.

**Fix.** Nothing was lost because the design expects it: guests start on boot, filesystems replay their
journals, the Git server's SQLite database passes `quick_check`, and interrupted CI jobs are marked failed by
the server's zombie-task timeout so people simply re-run them. A checklist (boot log, SMART, filesystem
recovery, database check, platform status) turns "I hope it's fine" into evidence.

**Lesson.** Short-press the power button and wait: it starts a clean shutdown. Holding it is the last resort.

---

## 8. A router that cannot hairpin

**Symptom.** CI jobs could not check code out from the CI site, though it worked from outside.

**Cause.** The router does not route traffic for its own public address back into the LAN.

**Fix.** Job containers start with `--add-host` pinning the public name to the edge proxy's LAN address; admin
scripts use `curl --resolve` and Git's `http.curloptResolve`. The name, and so the certificate, are unchanged.

---

## 9. An ISP that blocks port 80

**Symptom.** HTTP-01 certificate validation can never succeed.

**Fix.** DNS-01 through the dynamic DNS provider's API, with a hook that waits until two independent public
resolvers both see the TXT record. One resolver was not enough: the certificate authority validates from
several vantage points, and one of them once saw a stale record.

---

## 10. fail2ban versus webhooks

**Symptom.** None yet, by design. The risk: a misconfigured webhook secret would make every GitHub delivery fail
verification.

**Cause.** The sign-in jail bans an address after repeated `401`s on the API paths. GitHub's webhook senders are
a shared pool of addresses; banning them would silently break every repository's sync.

**Fix.** The webhook service answers a bad or missing signature with `403`, which no jail counts.

**Lesson.** When you add an endpoint behind existing ban rules, check which status codes those rules read.

---

## 11. Logs that survive a rename

**Symptom.** After an account was renamed, its older CI logs returned `404` through the API.

**Cause.** The server files each job log under the account name *at the time the job ran*.

**Fix.** When the expected path is missing, the API looks the log up by its unique job ID across all accounts.
The account cleanup command treats a rename as a rename: it keeps the plan and never deletes a log folder that
any job still points to.

---

## 12. A job name with an em dash

**Symptom.** Downloading the log of a job named `Unit tests — fast` failed with `502`.

**Cause.** The API returns the job name as an HTTP header, and header values must be Latin-1. The `—` raised an
encoding error mid-response.

**Fix.** Header values are normalised to plain ASCII (dashes mapped, accents stripped). The body is untouched.

---

## 13. A run marked failed while jobs still run

**Symptom.** Re-run returned `502` and cancel returned `409` on a run whose page showed jobs still running.

**Cause.** The server marks a run `failure` as soon as one job fails, while its other jobs carry on. The API
judged by the run's status alone.

**Fix.** Re-run and cancel look at the run's jobs: re-run waits until every job has finished, cancel works while
any job is still active, and both say why when they refuse.

---

## 14. Two guests that cannot share the RAM

**Symptom.** Starting the game server while CI was busy would push the host into swap on a spinning disk.

**Fix.** The CI runner has a fixed allocation so it can never be squeezed, and the status check warns whenever
the game server is running. The rule is documented: stop the runner first, or don't start the game.

**Lesson.** On a host where RAM cannot be upgraded, write the incompatible pairs down where you will see them.
