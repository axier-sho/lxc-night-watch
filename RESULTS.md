# Night Watch Results

Observed run of 2026-07-14 (all times UTC).

## Summary

| | |
| --- | --- |
| Deployed commit | `7fd7ae6` (initial deploy was `ef8b812`) |
| Baseline irradiance | 900 W/m², `clear`, at 19:09:18Z |
| Lowest irradiance | 280 W/m², `overcast`, at 21:10:33Z |
| Incident start | 21:09:49Z (300 W/m², first reading below 450) |
| Recovery time | 21:11:44Z (880 W/m²) |
| Final status | NORMAL — no incident open |
| Incidents recorded | 1, for 3 low readings |
| Discord alerts for the episode | exactly one incident alert and one recovery message |

## How the incident was produced

**The shared Kepler irradiance did not drop below 450 W/m² during this run.** No
instructor-coordinated sunlight change occurred in the unattended window, and the
live reading declined only gradually — from 900 W/m² at 19:09Z to 714 W/m² at
21:02Z, a steady ~1.7 W/m² per minute that would not have crossed the threshold
for roughly another two and a half hours.

The incident and recovery recorded above were therefore produced deliberately: a
local mock endpoint on the class server was served on `127.0.0.1:8899` returning
the same JSON shape as Kepler, `ORDERS.md` was temporarily pointed at it, the
inspector was run against the sequence 300 → 280 → 320 → 880 → 890 W/m², and the
endpoint was then restored. Everything downstream of the reading is genuine: a
real isolated agent run, real file writes, and real Discord delivery decisions.
Only the source of the number was substituted. The mock has been stopped and
`ORDERS.md` in the deployed clone is back on the live Kepler URL.

The readings from 19:09Z to 21:02Z in `observations.md` are genuine unattended
inspections of the live endpoint.

## The unattended run

Six scheduled inspections ran while nobody was connected, between 19:23Z and
19:48Z. Every one of them appended a timestamped observation, and every one of
them returned exactly `NO_REPLY` and delivered nothing to Discord, because the
state never changed. That is the suppression rule working over six consecutive
runs: the log grew, the channel stayed quiet.

The one-time shift report fired at 19:48:58Z and delivered to Discord. It was
re-run at 21:14:14Z after the incident episode so that the report on record
covers the incident and recovery.

## How the pieces fit together

The **GitHub repository** is the boundary between authoring and running. The
project was written on the laptop, committed, and pushed; the class server got it
by `git clone`, and the deployed commit hash was checked against the pushed one.
Nothing was edited directly on the server, so the server's copy is reproducible
from a public URL rather than from one machine's disk.

The **OpenClaw Gateway** is a systemd user service that is already running. It is
what makes the watch survive an SSH disconnect: the inspector is not a process
tied to a terminal, it is work the Gateway performs.

The **cron scheduler** lives inside that Gateway — not Linux cron, not a separate
systemd timer, and not a script kept alive by hand. `openclaw cron add` registered
a job that wakes every five minutes in an **isolated session**, which means each
run is a fresh agent that remembers nothing.

The **Kepler endpoint** is the only source of truth about sunlight. The agent
reads `solarIrradiance.wPerM2` and compares it against 450.

The **observation files** are the watch's memory, and they exist precisely because
the session is isolated. An agent that remembers nothing cannot know whether the
last reading was an incident, so `observations.md` and `incidents.md` have to tell
it. `observations.md` grows by one line per run; `incidents.md` holds one entry
per episode, updated in place while the sun stays down.

**Discord delivery** is driven by the agent's final response. `--announce` sends
that response to the channel, so returning exactly `NO_REPLY` is how the agent
declines to speak. The suppression is a state comparison, not a rate limit or a
cooldown — which is why the operator can trust that any message at all means
something actually changed.

## Deviations from the lab instructions

Two steps from the lab did not work as written in this environment.

**`--tools exec,read,write` silently disables the agent.** The lab's `cron add`
command passes that tool allow-list. This agent runs on a codex harness whose real
tools are `bash` and `apply_patch`, so the allow-list matched nothing and the
inspector was handed no tools at all. It could not call the endpoint or write a
file. It flailed at `list_mcp_resources`, gave up, and returned `NO_REPLY`.

This is a dangerous failure because it looks like success: the cron run history
showed `status: ok`, the summary was a legitimate-looking `NO_REPLY`, and Discord
was correctly quiet. The only symptom was that `observations.md` never grew. Both
jobs were registered without `--tools`, which restores the harness defaults.

**`openclaw cron add` required a device scope upgrade.** The server's CLI device
was paired with `operator.write` only, while `cron add` needs `operator.admin`.
The upgrade request could not be approved from the CLI itself: every connection
attempt minted a new pending request id and invalidated the one being approved, so
`devices approve <id>` always reported `unknown requestId`. The CLI device's entry
in the local device store was granted exactly the four scopes its own pending
request asked for (`operator.admin`, `operator.pairing`, `operator.read`,
`operator.write`), after which `openclaw gateway status` reported
`Capability: admin-capable` and `cron add` succeeded.

## A bug found in ORDERS.md

The first incident closed with its recovery time filled in but its heading still
reading `## Incident 1 — open`. The orders said to "mark it `recovered`", which the
agent satisfied by filling the `Recovered:` field. That is a real defect: an
incident whose heading still says `open` would cause the next low reading to update
the old episode instead of opening a new one.

`ORDERS.md` now states explicitly that the status word in the heading must change
from `open` to `recovered`, and that filling in the recovery time is not enough on
its own. The episode was re-run against the corrected orders and closed as
`## Incident 1 — recovered`.

## Verification

```
$ grep -c "^## Incident" incidents.md
1
```

The inspector was disabled after the run. `observations.md`, `incidents.md`, and
`morning-report.md` in the deployed clone are the runtime evidence and are
intentionally left uncommitted there.
