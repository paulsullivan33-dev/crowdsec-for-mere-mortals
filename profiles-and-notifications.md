# Profiles and Notifications

Outcome: you can read `profiles.yaml` and
predict what any alert will cause; you can
make repeat offenders earn longer bans;
and you can have CrowdSec tell *you* when
it acts — proven with a notification that
the lab's alert really produced.

## The profile file

An alert, by itself, changes nothing. The
agent runs each new alert through the
profiles in `/etc/crowdsec/profiles.yaml`,
top to bottom. A profile whose **filter**
matches contributes its **decisions**;
`on_success: break` stops the chain. Here
is the shipped default, complete:

```yaml
name: default_ip_remediation
filters:
  - Alert.Remediation == true && Alert.GetScope() == "Ip"
decisions:
  - type: ban
    duration: 4h
on_success: break
---
name: default_range_remediation
filters:
  - Alert.Remediation == true && Alert.GetScope() == "Range"
decisions:
  - type: ban
    duration: 4h
on_success: break
```

That is the entire policy behind chapter
5's ban: an alert whose scenario asked for
remediation (`Remediation == true` — the
ssh-bf scenario's labels said so), scoped
to one address, earns a four-hour ban.
Filters are small expressions over the
alert's fields — the scenario's name, its
labels, the country, the event count — and
profiles are how local policy is written:
different durations per service, captcha
instead of ban for the website's
scenarios, no decision at all for a
scenario you only want to watch. The file
is reloaded with the daemon
(`systemctl reload crowdsec`).

## Longer bans for repeat offenders

The shipped file contains one more idea,
commented out — a duration *expression*:

```yaml
#duration_expr: Sprintf('%dh', (GetDecisionsCount(Alert.GetValue()) + 1) * 4)
```

Uncomment it in place of the fixed
`duration: 4h` and the arithmetic of
chapter 5 changes: the first ban for an
address is 4 hours, the second 8, the
third 12 — counted from the decisions that
address has already earned. It is the
standard answer to "the same address comes
back every four hours and five minutes",
and it costs nothing until it is needed.
Say plainly what it is not: it is not a
permanent ban, and a patient attacker on
rotating addresses is untouched by it.
Chapter 14 keeps that boundary honest.

## Simulation: profiles without teeth

Before trusting a scenario with real bans,
you can tell CrowdSec to *pretend*:
simulation mode makes alerts — and records
the decisions it *would* have taken — but
issues no real decisions. The lab
switched it on for the ssh-bf scenario
(transcript `t07b`):

```text
$ cscli simulation status
global simulation: disabled

$ cscli simulation enable crowdsecurity/ssh-bf
simulation mode for "crowdsecurity/ssh-bf" enabled

$ cscli simulation status
global simulation: disabled
Scenarios in simulation mode:
  - crowdsecurity/ssh-bf
```

Alerts raised in simulation say so in
`cscli alerts inspect` (`Simulation: true`)
— chapter 6's field, seen from the other
side. Simulation can be global or
per-scenario, as here; per-scenario is
the useful form, letting a new, unproven
scenario report for a week while the
trusted ones keep enforcing. Like hub
changes, it takes effect on reload.

## Notifications: CrowdSec tells you

A ban at 3 a.m. that nobody hears about is
half a control. Profiles carry a
`notifications:` list naming **plugins** —
programs the agent starts and hands each
matching alert to. The package ships
plugins for email, a file, HTTP webhooks,
Slack, Splunk and an alerting service
(Sentinel), configured one per file in
`/etc/crowdsec/notifications/`.

The lab switched on the simplest — the
file plugin — by adding two lines to the
first profile:

```yaml
notifications:
  - file_default
```

and pointing `notifications/file.yaml`'s
`log_path` at a file of its own. The
plugin's format template renders each
alert as one line of JSON. When the
chapter 5 attack fired, the file gained
exactly that (transcript `t09`, trimmed
here to its load-bearing fields):

```json
{ "scenario": "crowdsecurity/ssh-bf",
  "events_count": 6,
  "remediation": true,
  "source": { "ip": "203.0.113.17", "scope": "Ip", "value": "203.0.113.17" },
  "decisions": [ { "duration": "4h", "origin": "crowdsec",
                   "type": "ban", "value": "203.0.113.17" } ] }
```

The file went on to hold five lines: every
alert the lab created, including the
manual decisions from chapter 7 — a
hand-made decision is an alert like any
other, and the profile's notification list
sees it. (Its *decisions* are not
duplicated — the manual alert already
carries the decision its author gave it.)

The other plugins share the mechanism and
differ only in configuration, and are
taught here from their shipped files
rather than run: **email** (SMTP settings
and a recipient list), **http** (a URL,
method and headers — the shape a webhook
to a chat or ticketing system takes),
**slack** (a channel webhook). Two knobs
on every plugin matter in practice:
`group_wait` (collect alerts for a while
before sending, so an attack is one
message, not fifty) and `group_threshold`
(send early if this many alerts pile up).
Set them before the first noisy night,
not after.

Two honest limits. Notifications are
*notifications*: the chain cscli → profile
→ plugin has no acknowledgement that a
human saw anything, and if the plugin
fails, the ban still stands and the alert
is still in the database — the record is
the system; the message is a courtesy.
And a notification is not a monitoring
system: chapter 13's neighbours include
the tools that watch the watchers.

Next: the half that makes it all real —
the bouncer, and a firewall that actually
drops the packets.
