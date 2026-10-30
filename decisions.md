# Decisions

Outcome: you can read a decision list the
way an auditor would — who decided, about
whom, what action, until when — and you
can add and remove decisions by hand,
including the two types a firewall cannot
express.

## The anatomy of a decision

Everything a bouncer enforces is a row in
one list:

```bash
sudo cscli decisions list
```

A decision carries: a **scope** and value
(what it applies to), a **type** (what to
do), a **duration** (decisions always
expire; there is no permanent ban in this
system, only long ones), a **reason** (the
scenario, or your own words), and an
**origin** (where it came from). The lab
produced decisions from two origins in one
afternoon — `crowdsec`, the agent acting on
an alert, and `cscli`, a person at a
keyboard:

```text
| ID |  Source  |   Scope:Value   |              Reason              | Action  | expiration |
| 1  | crowdsec | Ip:203.0.113.17 | crowdsecurity/ssh-bf             | ban     | 3h59m54s   |
| 2  | cscli    | Ip:203.0.113.99 | Manual decision added from cscli | ban     | 1h59m59s   |
| 3  | cscli    | Ip:203.0.113.200| manual 'captcha' from '0a34…'    | captcha | 59m59s     |
```

A third origin, `CAPI`, appears when the
community blocklist is enabled (chapter
10). The origin is not bookkeeping: the
firewall bouncer in chapter 9 keeps a
*separate firewall set per origin*, so the
list's Source column is visible all the way
down in the kernel.

## Types: ban is not the only verb

- **ban** — refuse the source entirely.
  The firewall bouncer drops its packets;
  web bouncers refuse its requests.
- **captcha** — challenge the source
  instead of refusing it: show a CAPTCHA
  and let a human through. Only bouncers
  that serve people (web servers,
  applications) can express this; a
  firewall bouncer ignores captcha
  decisions it cannot perform.
- **throttle** — slow the source down
  rather than stopping it, where a bouncer
  supports rate limiting.

The type is part of the decision, chosen
by the profile for automatic decisions
(chapter 8) or by you for manual ones. The
default profile bans; a gentler profile
for a public website might captcha, on the
reasoning that a false positive then costs
a puzzle, not a lockout.

## Scopes: how wide the net is

- **Ip** — one address. The normal case;
  every decision in this guide's lab.
- **Range** — a CIDR range (`203.0.113.0/24`),
  for when a whole network is misbehaving.
  The default profiles include a second
  profile for range-scoped alerts.
- **Country** — a whole country, by the
  GeoIP enrichment. Powerful, blunt, and
  occasionally the honest answer to a
  flood; treat it as a decision you
  *justify*, not one you reach for.
- Others exist for other bouncers —
  **username** for application bouncers,
  for instance.

## Making decisions by hand

The everyday case: you are reading a log,
or a report, and an address needs to be
banned *now*, evidence and judgement
supplied by you. The lab did exactly this
(transcript `t07`):

```bash
cscli decisions add --ip 203.0.113.99 --type ban \
    --duration 2h --reason "Manual decision added from cscli (lab)"
cscli decisions add --ip 203.0.113.200 --type captcha --duration 1h
```

Two things happen that are worth noticing.
First, each manual decision **creates an
alert** of its own (the list above shows
Alert IDs 2 and 3) — so a hand-made ban is
a first-class record you can inspect
later, with your reason attached, and it
flows to notifications like any alert
(chapter 8 confirms this from the
notification file). Second, the default
duration is four hours if you omit
`--duration` — the same four hours as the
default profile, and a common surprise:
people expect a manual ban to be permanent
and find it gone by morning. If you want a
week, say `--duration 168h`.

Deleting is the mirror:

```bash
cscli decisions delete --ip 203.0.113.200
```

```text
level=info msg="1 decision(s) deleted"
```

Deletion is how a false positive is undone:
the decision leaves the API at once, and
every bouncer picks the change up on its
next poll (seconds, for the firewall
bouncer). The *alert* stays in the history
— you removed the punishment, not the
record, which is the right way round.
There is also `cscli decisions import` for
adding decisions in bulk from a CSV or
JSON file — a blocklist of your own, for
instance — with columns for duration,
reason, scope, type and value.

## Decisions expire; alerts remain

Left alone, every decision counts down to
zero and vanishes from the list, while its
alert stays in the database (alerts and
decisions have their own retention;
`db_config` in `config.yaml` flushes old
items — the shipped default keeps at most
5000 items and nothing older than seven
days at the flush check). That asymmetry
is the design: enforcement is temporary by
default, evidence is kept a while longer.
A banned address that returns and attacks
again simply earns a fresh decision —
and, with the profile trick in chapter 8,
a longer one.

Next: the profiles that decide durations
for you, and the notifications that tell
you it happened.
