# Glossary

**Acquisition** — the configuration that
tells the agent what to read: files (or
other sources) plus a type label, in
`/etc/crowdsec/acquis.yaml` and `acquis.d/`.

**Agent** — the `crowdsec` program itself:
reads logs, runs parsers and scenarios,
raises alerts, serves the Local API.
Detects and decides; never enforces.

**Alert** — the record that a scenario's
conditions were met: who, what scenario,
which events, what context. Alerts become
decisions only through a profile.

**API key** — a bouncer's credential for
the Local API. Created with
`cscli bouncers add`, shown once, good for
reading decisions and reporting back.

**AppSec** — CrowdSec's optional
web-application-firewall component, which
inspects live HTTP requests rather than
logs. Bounded, not taught, in this guide.

**Blackhole** — a scenario setting: after
an overflow, ignore that source's further
events for a while (1m for ssh-bf), so one
attack raises one alert.

**Bouncer** — any program that polls the
Local API for decisions and enforces them
where it lives: a firewall, a web server,
an application. Also called a remediation
component.

**Bucket (leaky)** — the counter most
scenarios use: events pour in grouped by
source, leak at a fixed rate, and an
overflow raises an alert. Capacity and
leakspeed define the threshold.

**CAPI** — the Central API, CrowdSec's
sharing service: agents push sanitised
attack signals and pull the community
blocklist. Off until registered with
`cscli capi register`.

**Collection** — a hub bundle for one
service: its parsers, scenarios,
postoverflows and contexts, installed
together.

**Community blocklist** — decisions
distilled by the Central API from the
whole crowd's signals, delivered to
registered machines with origin `CAPI`.

**Console** — the web dashboard
(app.crowdsec.net) over a CAPI account:
machines, alerts, blocklists, and paid
extras.

**Context** — extra facts a hub item
attaches to an alert (for SSH brute force:
the user names that were tried). Visible
in `cscli alerts inspect`.

**cscli** — the command-line management
tool: hub items, machines, bouncers,
alerts, decisions, metrics. A client of
the Local API for most commands.

**Decision** — the unit of enforcement: a
scope and value, a type (ban, captcha,
throttle), a duration, a reason, an
origin. Decisions always expire.

**Enrichment** — facts added to an event
after parsing: parsed dates, GeoIP
country and network, reverse DNS,
allowlists.

**Event** — one parsed log line, as a set
of named fields (`source_ip`,
`target_user`, `log_type`…).

**Explain** — `cscli explain`: runs sample
lines through the real parsers and
scenarios and prints their journey. The
rehearsal tool.

**Grok pattern** — the named text patterns
(in `/etc/crowdsec/patterns/`) parsers use
to pull fields out of log lines.

**Hub** — the catalogue of parsers,
scenarios, collections and more, indexed
locally by `cscli hub update` and
installed from by name.

**LAPI** — the Local API: the agent's HTTP
service (default 127.0.0.1:8080) through
which machines report, bouncers read, and
cscli manages.

**Machine (watcher)** — a client allowed
to *post* alerts to the Local API — the
agent itself, or another reporting host —
authenticated by a login and password.

**Overflow** — the moment a bucket fills:
the alert-raising event.

**Parser** — a hub file that matches a
kind of log line and extracts an event.
Runs in stages s00-raw, s01-parse,
s02-enrich.

**Postoverflow** — a parser-like stage
that runs *after* an alert is raised:
whitelists, more enrichment, draining
alerts that should not stand.

**Profile** — a filter in
`profiles.yaml` plus the decisions (and
notifications) to apply to matching
alerts. Where alerts become bans.

**Remediation** — a scenario label saying
its alerts ask for action. Profiles
typically require it before deciding
anything.

**Scenario** — a hub file that judges
events: a filter, a bucket, labels.
ssh-bf is the worked example of this
guide.

**Scope** — what a decision applies to:
an Ip, a Range, a Country, a username…

**Signal** — the sanitised summary of an
alert sent to the Central API by a
registered machine: address, scenario,
times, counts — no log lines.

**Simulation** — a mode in which scenarios
alert (marked as simulated) but cause no
real decisions. The rehearsal space.

**Tainted** — a hub item whose local copy
no longer matches the published one —
edited in place, or left behind by a
forced member removal. Listings flag it.

**Whitelist / allowlist** — exemptions
applied by parsers and postoverflows so
known-good sources (search crawlers, CDN
ranges, your own addresses) are never
judged or never banned.
