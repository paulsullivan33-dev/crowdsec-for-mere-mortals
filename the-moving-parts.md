# The Moving Parts

Outcome: you can name every part of a
CrowdSec installation, say where it lives
on disk, and predict — when something
misbehaves later — which part to look at
first.

## The agent: `crowdsec`

One program does the watching. The agent
reads log sources, runs them through
parsers and scenarios, raises alerts,
applies profiles, and serves everything
through its API. On a packaged install it
runs as a systemd service, configured by
one file:

```text
/etc/crowdsec/config.yaml
```

That file names every other path the agent
uses — configuration directory, data
directory, the database, the API's listen
address. In this guide's lab the same file
was used with its paths rewritten into the
lab's folder; nothing else about it
changed. The defaults matter, so here they
are as the package ships them: the Local
API listens on `127.0.0.1:8080` (loopback
only), the database is a single SQLite file
at `/var/lib/crowdsec/data/crowdsec.db`,
and a metrics endpoint listens on
`127.0.0.1:6060`. PostgreSQL and MySQL are
supported database options for busier or
multi-agent setups; the lab used the
default SQLite and the guide teaches that.

The agent's own log is
`/var/log/crowdsec.log`. When anything in
this guide "just didn't work", that file
is where the reason was written — chapter
14 leans on it heavily.

## Acquisition: what the agent reads

The agent learns what to read from
**acquisition** files:
`/etc/crowdsec/acquis.yaml` and anything in
`/etc/crowdsec/acquis.d/`. Each entry names
a source — usually log files — and a
**type** label that tells the parsers what
kind of log it is:

```yaml
filenames:
  - /var/log/auth.log
labels:
  type: syslog
```

That small file is the whole contract
between your server's logs and CrowdSec's
rulebook. Point it at the wrong file, or
label the file with the wrong type, and
detection silently has nothing to chew on.
Chapter 5 builds one by hand so the
contract is visible; on a normal install,
chapter 3's setup step writes these for
the services it detects.

The agent can also read from journald,
Docker containers, cloud log services and
more — the version banner in transcript
`t01` lists the data sources compiled in.
Files are the common case and the one this
guide runs.

## Parsers: turning lines into events

A log line is text. A **parser** is a hub
file that matches a kind of line and pulls
named facts out of it: source address,
target user, whether the login failed.
Parsers run in stages you will see in
chapter 5's `explain` output:

- **s00-raw** — the format readers. The
  syslog parser works here, recognising the
  envelope (date, host, program) of every
  line.
- **s01-parse** — the service readers. The
  sshd parser works here, recognising
  "Failed password for invalid user …" and
  stamping the event with a type:
  `ssh_failed-auth`.
- **s02-enrich** — the enrichers. Date
  parsing, GeoIP lookup of the source
  address (country, network), reverse DNS,
  allowlists of known public resolvers.

A line no parser claims is **unparsed** —
counted in the metrics, then dropped. That
count is the first thing to check when a
service seems invisible to CrowdSec.

## Scenarios: turning events into alerts

A **scenario** is a hub file that watches
parsed events and decides when they add up
to an attack. Most use a **leaky bucket**:
each matching event pours into a bucket
labelled by (usually) source address; the
bucket leaks at a fixed rate; if it fills,
it **overflows** — and the overflow is an
alert. The SSH brute-force scenario,
quoted in full in chapter 5, holds five
events and leaks one every ten seconds:
six quick failed logins from one address
overflow it, two never will.

Scenarios also come in *trigger* (one
event is enough — some things are never
innocent) and *conditional* flavours, and
each carries labels: what service it
concerns, how confident it is, whether its
alerts should produce **remediation** at
all. An alert is a record — who, what,
when, the events that caused it — stored
in the database and visible with
`cscli alerts list`.

## Profiles: turning alerts into decisions

Between alerts and decisions sits one more
file, `/etc/crowdsec/profiles.yaml`. A
**profile** is a filter plus a list of
decisions to create when an alert matches.
The default profile says, in effect: *for
any alert that asks for remediation, scoped
to a single address, ban that address for
four hours.* Profiles are where "an alert"
becomes "a ban", where repeat offenders can
be given longer bans, and where
notifications are attached — chapter 8
takes the file apart.

A **decision** is the unit bouncers
consume: a scope (an address, a range, a
country, a user name), a type (`ban`,
`captcha`, `throttle`), a duration, a
reason, and an origin — `crowdsec` for the
agent's own, `cscli` for ones typed in by
hand, `CAPI` for ones from the community
blocklist. Decisions expire on their own;
an expired decision simply stops being
served.

## The Local API and `cscli`

Everything talks to everything else through
the **Local API** (LAPI), an HTTP service
the agent runs — loopback-only by default.
Two kinds of client connect:

- **Machines** (also called watchers) push
  alerts and pull their configuration: that
  is the agent itself, and anything else
  that reports. A machine authenticates
  with a login and password the
  administrator creates with cscli; the
  agent stores its own in
  `/etc/crowdsec/local_api_credentials.yaml`.
- **Bouncers** pull decisions. A bouncer
  authenticates with a single API key,
  created with `cscli bouncers add` and
  shown exactly once.

**`cscli`** is the management tool: it
installs hub items, registers machines and
bouncers, lists and inspects alerts, adds
and deletes decisions by hand, and prints
metrics. It is a client like any other —
most of its commands are conversations
with the Local API, which is why several
of them fail, instructively, when the
daemon is stopped (chapter 14 has the
transcript).

## The hub

The **hub** is the catalogue all parsers,
scenarios and their bundles come from.
`cscli hub update` downloads its index
(the lab's copy listed 171 parsers, 790
scenarios, 176 collections and more);
items are then installed by name. A
**collection** is a bundle for one service
— parsers, scenarios, and extras — so
"protect SSH" is one install. Installed
items live under `/etc/crowdsec/` in
folders by kind (`parsers/`, `scenarios/`,
`collections/`, `postoverflows/`), as real
files you can read. Chapter 4 does this
for real.

## Bouncers

A **bouncer** (the project also calls them
remediation components) is any program
that polls the Local API for decisions and
enforces them where it lives. The
**firewall bouncer** is the one this guide
runs: it keeps the banned addresses in
nftables (or iptables) sets, updating them
every few seconds and removing its tables
when it stops. Others exist for web
servers and applications — they enforce
`captcha` and `throttle` decisions a
firewall cannot express. Chapter 9.

## Two more parts, named and bounded

- **AppSec** — an optional component that
  inspects live web requests (not logs)
  against web-attack rules: CrowdSec as a
  small WAF. It is real, hub-fed, and
  outside this guide's lab; chapters 14
  mentions it only to bound it.
- **The Central API and the console** —
  the sharing half, chapter 10. Off until
  registered; nothing else depends on it.

## Who holds what — the map to keep

| Question | Part |
|---|---|
| Why wasn't this line understood? | Parsers (metrics: unparsed) |
| Why didn't these events alert? | Scenarios (the bucket) |
| Why is there an alert but no ban? | Profiles |
| Why is there a ban but no blocking? | The bouncer |
| Why is a stranger's address already banned? | A manual decision, or CAPI |
| Where is it all recorded? | The database, via the Local API |

Next: installing the agent, and meeting
`cscli` properly.
