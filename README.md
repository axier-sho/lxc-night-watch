# LXC Night Watch

An OpenClaw agent that watches Kepler's solar irradiance while nobody is logged
in, and wakes the operator on Discord only when the situation actually changes.

The watched value is `solarIrradiance.wPerM2` from
`https://planet.turingguild.com/world/solar-irradiance`. Below **450 W/m²** the
habitat is not collecting enough power to sustain normal operation, so that is
the line between quiet and alarming.

## The loop

Every five minutes the OpenClaw Gateway starts the inspector in an **isolated
session**: a fresh agent with no memory of the previous run. It then runs one
pass of the same five steps.

**Wake.** The Gateway's cron scheduler starts the run. There is no long-lived
process polling in a loop and no terminal left connected — between runs nothing
of the Night Watch is running at all.

**Inspect.** The agent queries the Kepler endpoint and reads `wPerM2` and
`condition` from the JSON.

**Decide.** It classifies the reading as NORMAL, INCIDENT, or RECOVERED. Because
the session is isolated, "what was happening last time" cannot come from the
agent's memory — it reads `observations.md` and `incidents.md` off the disk. Those
files *are* the watch's memory.

**Record.** Every run appends a timestamped line to `observations.md`, including
the runs that report nothing. The log is the evidence that the watch was awake,
so a silent night still has to leave a trail.

`incidents.md` holds one entry per low-sunlight *episode*. A four-hour outage
observed 48 times is one incident that gets updated 48 times, not 48 incidents.

**Report.** The agent's final response decides delivery. If the state has not
changed, it returns exactly `NO_REPLY` and OpenClaw sends nothing. Only a
transition — normal into incident, or incident into recovery — produces a Discord
message.

This is the point of the design. The inspector runs every five minutes forever,
but a night with one power dip produces exactly two messages: one alert and one
recovery. The suppression is not a rate limit or a cooldown; it is a state
comparison, so the operator can trust that a message means something is different.

## Files

| File | Purpose |
| --- | --- |
| `ORDERS.md` | The inspector's standing orders — endpoint, threshold, and the recording and reporting rules |
| `observations.md` | Append-only log, one line per inspection |
| `incidents.md` | One entry per low-sunlight episode, opened and later closed |
| `morning-report.md` | Shift summary, written by the scheduled report run |
| `RESULTS.md` | What was actually observed during the unattended run |

## Running it

The project is deployed by cloning this repository onto the class server. The
inspector and the shift report are OpenClaw cron jobs, registered with
`openclaw cron add` and executed inside the already-running Gateway — there is no
system cron entry, no extra systemd unit, and no script to keep alive.
