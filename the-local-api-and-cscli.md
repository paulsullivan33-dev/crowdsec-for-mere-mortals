# The Local API and cscli

Outcome: you know who may talk to the
Local API and how each kind of client
proves itself, and you can take an alert
apart with `cscli alerts inspect` — the
difference between "it banned something"
and knowing exactly what, why, and for how
long.

## One door, three kinds of caller

The Local API is an HTTP service the agent
runs (default `127.0.0.1:8080`; the lab used
127.0.0.1:18085). Everything in a CrowdSec
installation goes through it:

- **The agent itself**, as a *machine* (the
  API's word is also *watcher*): it logs in
  with a name and password, receives a
  token, and posts the alerts it raises.
  Its credentials live in
  `/etc/crowdsec/local_api_credentials.yaml`
  — created by the installer, or by
  `cscli machines add` when you add another
  reporting machine by hand.
- **`cscli`**, acting with the same machine
  credentials, for nearly everything in
  this guide.
- **Bouncers**, each with its own API key
  (chapter 9), allowed to read decisions
  and report what they have enforced — and
  nothing else.

Two trust settings in `config.yaml` draw
the boundary. The API listens on loopback
by default, so "the network" is this one
machine unless you deliberately change it;
and `trusted_ips` (127.0.0.1 and ::1 as
shipped) lists who may use the
administrative endpoints at all. Running
one agent whose API serves several
machines' agents and bouncers is a
supported shape — it is also the moment to
think hard about who else can reach that
port, and to put TLS in front of it
(the configuration file has a `tls:`
section for exactly this). The lab ran
the default shape: everything on one
private loopback.

A consequence worth internalising: because
cscli is a client, its decision and alert
commands need the daemon **up**. The lab
met the failure in its sharpest form:

```text
msg="crowdsec init: while initializing LAPIClient: authenticate watcher
(0a34...): Post \"http://127.0.0.1:18085/v1/watchers/login\": performing
jwt auth: dial tcp 127.0.0.1:18085: connect: connection refused"
```

That is not corruption or a bad install —
it is cscli knocking on a door nobody is
behind. (A few commands, like
`cscli machines list`, read the database
file directly and work either way; the
difference shows in chapter 14's table.)

## Taking an alert apart

`cscli alerts list` is the summary;
`cscli alerts inspect <id>` is the record.
The lab's brute-force alert, transcript
`t06-alert-inspect.txt`:

```text
$ cscli alerts inspect 1
 - ID           : 1
 - Date         : 2026-10-30T22:31:30Z
 - Machine      : 0a3408554c4d4e6dbc67faae93a57b41y4iqFAIhDTwt0b72
 - Simulation   : false
 - Remediation  : true
 - Kind         : crowdsec
 - Reason       : crowdsecurity/ssh-bf
 - Events Count : 6
 - Scope:Value  : Ip:203.0.113.17
 - Country      : 
 - AS           : 
 - Begin        : 2026-10-30T22:31:25Z
 - End          : 2026-10-30T22:31:29Z
 - UUID         : 237d2749-575e-465b-a2e4-41b7ac7f60fd

| Active Decisions                                                  |
| ID |   scope:value   | action | expiration |      created_at      |
| 1  | Ip:203.0.113.17 | ban    | 3h59m54s   | 2026-10-30T22:31:30Z |

 - Context  :
|     Key     |  Value |
| target_user | admin  |
| target_user | oracle |
| target_user | test   |
| target_user | guest  |
```

Read it as a small dossier. The machine
that reported it (the long ID is the agent
itself, registered in chapter 3). Whether
it was a simulation (chapter 8 — a dry run
would say `true` here, and produce no real
decision). The scenario that fired and the
six events that fed it, over a four-second
window. The decision it produced, expiring
in just under four hours. And the
**context**: the `bf_base` context from the
SSH collection recording the distinct user
names the attacker tried — evidence you
would want in a report, gathered
automatically.

`cscli alerts inspect` has a `--details`
flag that adds the events themselves;
between list, inspect, and the metrics of
chapter 5, there is very little about an
alert you cannot establish from the
outside. That is deliberate, and it is the
habit to build: **never act on a ban you
have not inspected.** The decision it made
may be right; your job is to be able to
say why.

## The API keeps the score

The metrics output had one more table
worth knowing (transcript `t08`): the
Local API's own request counts, by route —
`/v1/alerts` posted and read, `/v1/decisions`
deleted once, `/v1/watchers/login` a dozen
times. Every actor in this chapter appears
there as traffic. On a machine where
something feels haunted — decisions
vanishing, alerts you do not recognise —
that table, `cscli machines list`, and
`cscli bouncers list` together enumerate
*everyone who is allowed to do anything*,
which is most of an audit.

Next: decisions themselves — the types,
the scopes, and making them by hand.
