# Recipes

Outcome: the short, repeatable forms of
the jobs this guide has already proven —
each one assembled from steps run in the
lab, with its check named.

## Ban an address now, by hand

*For: the address in front of you in a log,
a report, a colleague's message.*

```bash
sudo cscli decisions add --ip 203.0.113.99 --duration 24h \
    --reason "Caught in the mail log, 400 attempts"
sudo cscli decisions list
```

Check: the decision is in the list with
your reason and origin `cscli`, and —
firewall bouncer running — appears in
`nft list set ip crowdsec crowdsec-blacklists-cscli`
(the table's family is `ip`, not `inet`;
chapter 14 keeps the error that teaches
it) on the bouncer's next poll. Always give a
duration and a reason: the default is four
hours, and future-you, reading the list at
midnight, deserves the reason. Undo with
`cscli decisions delete --ip 203.0.113.99`.

## Rehearse a new scenario in simulation

*For: a scenario you do not trust yet —
yours, or the hub's newest.*

```bash
sudo cscli simulation enable crowdsecurity/ssh-slow-bf
sudo systemctl reload crowdsec
sudo cscli alerts list            # over the following days
```

Its alerts arrive labelled, and
`cscli alerts inspect <id>` shows
`Simulation: true` with **no** decision
attached — that absence is the check.
When its judgement has earned trust,
`cscli simulation disable
crowdsecurity/ssh-slow-bf`, reload, and it
enforces like the rest.

## Test a log you have never fed it

*For: "will CrowdSec understand this
service's logs?" — before wiring anything.*

```bash
tail -n 50 /var/log/the-service.log > sample.txt
sudo cscli explain --type syslog -v --file sample.txt
```

Read the tail of the output: *parser
success* and a non-empty scenario list
mean the pipeline understands the lines.
A wall of unparsed lines means the
collection for that service is missing, or
the `type:` is wrong — fix the label or
install the collection before touching the
acquisition file. (The daemon must be
running for explain in the usual
configuration; chapter 14 has the failure
that teaches why.)

## Send every alert to a file you can grep

*For: a mailbox-free audit trail, or
feeding another tool.*

In `/etc/crowdsec/profiles.yaml`, add to
the first profile:

```yaml
notifications:
  - file_default
```

In `/etc/crowdsec/notifications/file.yaml`,
set `log_path` to where the file should
live, then reload. Check:

```bash
sudo cscli decisions add --ip 192.0.2.1 --duration 5m --reason "Notification test"
tail -1 /path/to/your/alerts.json
```

One line of JSON with your test decision
in it proves the whole chain — profile,
plugin, file — before any real attack
tests it for you. The lab's file collected
five alerts this way, including exactly
such hand-made ones.

## See what the crowd would ban — before joining it

*For: evaluating chapter 10's bargain with
the lights on.*

There is no offline preview, and any tool
promising one is guessing. The honest
version of this recipe is a reading list,
in order: `cscli capi status` (your state),
`cscli console status` (what would be
forwarded, switch by switch), and chapter
10 itself. If you register, register one
machine first, and read its CAPI-origin
firewall set after a day (the bouncer
keeps one set per origin, chapter 9): the
size and churn of that set *are* the
blocklist's character, measurable on your
own machine.

## Unban a regular who fat-fingered

*For: the false positive, handled calmly.*

```bash
sudo cscli decisions list                 # find the decision, note its alert
sudo cscli alerts inspect <id>            # read the evidence — all of it
sudo cscli decisions delete --ip <address>
```

Then the durable fix, chosen by cause: a
mistyped password twice is not an attack
and needs no fix — the bucket already
forgave it. A monitoring system that logs
in wrongly every hour needs its credential
fixed, not its address exempted. And an
address that must *never* be banned —
your own office, a health-check service —
belongs in a whitelist: the hub's
whitelist collections and parser-level
whitelists exist for it, and exempting by
*parser* keeps the exemption in front of
every scenario at once.
