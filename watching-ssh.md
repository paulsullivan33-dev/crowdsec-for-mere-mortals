# Watching SSH — Your First Alert

Outcome: CrowdSec reading an SSH log, a
brute-force attack recognised, and an alert
plus a four-hour ban produced — with each
step *checked*: first that two failures
correctly produce nothing, then that ten
produce exactly one alert.

## Telling the agent what to read

The agent reads what its acquisition files
name. Ours, for this chapter, is one file
(`crowdsec-lab/acquis.yaml` in the lab's
checkout holds it):

```yaml
filenames:
  - /var/log/auth.log
labels:
  type: syslog
```

On Debian and Ubuntu, sshd logs password
failures to `/var/log/auth.log` in syslog
format; on a packaged install the setup
step usually writes this for you. The
`type: syslog` label is the key that starts
the parser chain: syslog lines are opened
by the syslog parser, and lines from sshd
are then claimed by the sshd parser you
installed in chapter 4.

## The attack, on paper first

The lab's attack is a fixture: thirteen
lines in `crowdsec-lab/auth-fixture.log`.
Ten are failed logins from one address,
one is a failure from a second address,
one is a successful login, one is junk:

```text
Oct 30 22:31:25 labhost sshd[2408]: Failed password for invalid user admin from 203.0.113.17 port 51845 ssh2
Oct 30 22:31:25 labhost sshd[2457]: Failed password for invalid user oracle from 203.0.113.17 port 51609 ssh2
Oct 30 22:31:25 labhost sshd[2428]: Failed password for invalid user admin from 198.51.100.23 port 51693 ssh2
Oct 30 22:31:25 labhost sshd[2501]: Accepted password for marina from 192.0.2.44 port 51300 ssh2
this line is not a log line at all
...then, seconds later, eight more failures from 203.0.113.17,
    for admin, test, oracle, guest, admin, root, root, admin.
```

The addresses are from the ranges reserved
for documentation (203.0.113.0/24,
198.51.100.0/24, 192.0.2.0/24) — real
enough to parse, guaranteed to belong to
nobody. The host and users are invented.
Everything about the *shape* is faithful:
this is what a real password-guessing run
looks like in a real auth.log, only shorter
and with the names changed.

## The rule that will judge it

Before feeding anything, read the scenario
that will do the judging —
`/etc/crowdsec/scenarios/ssh-bf.yaml`,
abridged here to its working parts:

```yaml
type: leaky
name: crowdsecurity/ssh-bf
description: "Detect ssh bruteforce"
filter: "evt.Meta.log_type == 'ssh_failed-auth'"
leakspeed: "10s"
capacity: 5
groupby: evt.Meta.source_ip
blackhole: 1m
labels:
  service: ssh
  remediation: true
  label: "SSH Bruteforce"
```

Read it as a sentence: *for every event the
sshd parser stamped `ssh_failed-auth`, pour
one unit into the bucket belonging to that
source address. The bucket holds five and
leaks one unit every ten seconds. If it
fills, raise an alert that asks for
remediation — then ignore that address's
further events for one minute (the
blackhole), so one attack makes one alert,
not forty.*

Two consequences follow before any log is
read. A person who mistypes their password
twice will never be banned: two units leak
away in twenty seconds. And the sixth
failure inside the leak window is the one
that matters — the alert, when it comes,
should carry **six events**. Hold the
scenario to that prediction below.

## Rehearsing with `cscli explain`

You do not have to feed the daemon live
logs to see the pipeline work. `cscli
explain` pushes sample lines through the
real parsers and scenarios and prints the
journey (transcript `t05`):

```bash
cscli explain --type syslog -v --file sample-lines.txt
```

For a failed login it shows the stages —
the syslog parser opening the envelope, the
sshd parser extracting
`source_ip: 203.0.113.17`,
`target_user: root`, `log_type: ssh_failed-auth`,
the enrichers adding date and geography,
and at the end the list of scenarios the
event was poured into, ssh-bf among them.
For the *successful* login line, a
different parser (sshd-success-logs) claims
it and no brute-force scenario applies.
Two practical notes, both earned in the
lab: explain needs the `crowdsec` binary on
the PATH (it runs a test instance of it),
and with the daemon's credentials in place
it expects the daemon to be **running** —
stopped, it fails with a connection-refused
error naming the API address, which
chapter 14 keeps verbatim as a diagnostic,
not a mystery.

## Two failures: correctly, nothing

The daemon starts, tails the log file, and
the first lines are appended: the two
failures from 203.0.113.17, one from
198.51.100.23, the success, the junk.
Then, the check this whole chapter exists
for (transcript `t06-alerts-before.txt`):

```text
$ cscli alerts list   (after only two failures from the busiest address)
No active alerts
```

Nothing is the correct result, and seeing
nothing *on purpose* is what separates
testing from hoping. The bucket for
203.0.113.17 holds two units and is already
leaking; the other address holds one.

## Ten failures: one alert, one ban

Eight more failures from 203.0.113.17 are
appended — ten in total, well inside a
minute. Seconds later (transcript
`t06-alerts-after.txt`):

```text
$ cscli alerts list   (after ten failures from 203.0.113.17 in under a minute)
+----+-----------------+----------------------+---------+----+-----------+----------------------+----------+
| ID |      value      |        reason        | country | as | decisions |      created_at      |   kind   |
+----+-----------------+----------------------+---------+----+-----------+----------------------+----------+
| 1  | Ip:203.0.113.17 | crowdsecurity/ssh-bf |         |    | ban:1     | 2026-10-30T22:31:25Z | crowdsec |
+----+-----------------+----------------------+---------+----+-----------+----------------------+----------+

$ cscli decisions list
+----+----------+-----------------+----------------------+--------+---------+----+--------+------------+----------+
| ID |  Source  |   Scope:Value   |        Reason        | Action | Country | AS | Events | expiration | Alert ID |
+----+----------+-----------------+----------------------+--------+---------+----+--------+------------+----------+
| 1  | crowdsec | Ip:203.0.113.17 | crowdsecurity/ssh-bf | ban    |         |    | 6      | 3h59m55s   | 1        |
+----+----------+-----------------+----------------------+--------+---------+----+--------+------------+----------+
```

Every prediction from the scenario file is
visible: one alert (the blackhole swallowed
the four failures that arrived after the
overflow), **six events** in it (five to
fill the bucket, the sixth to tip it), and
a ban whose expiry counts down from the
default profile's four hours. The other
addresses never alerted — one failure each
is weather, not an attack. Note what is
*not* in the output: a country. The
enrichers ran (explain showed them), but
documentation addresses have no GeoIP
entry, so the column is blank. On a real
attack the same columns carry the country
and network the address belongs to.

And note what has not happened: nothing is
blocked yet. There is a decision in a
database. Enforcement is chapter 9's
bouncer; until then, 203.0.113.17 could
keep knocking. Detection did its whole
job — no more, no less.

## The numbers behind it: metrics

`cscli metrics` is the agent's report card,
and the lab's (transcript `t08`) accounts
for all thirteen lines:

```text
| Source          | Lines read | Lines parsed | Lines unparsed | Lines poured to bucket |
| file:...auth.log| 13         | 12           | 1              | 51                     |
```

Thirteen read; twelve parsed (ten failures,
the second address's failure, the success);
one unparsed — the junk line, counted and
discarded exactly as it should be. "Poured"
is larger than the line count because each
failed-login event is offered to every
matching scenario's bucket (the ssh-bf
family, the user-enumeration variants, the
time-based ones). The parser table in the
same output shows the sshd parser's
children doing the work and the syslog
parser passing most lines through. When a
real service seems unwatched, this table is
the diagnosis: lines read but unparsed
means the wrong parser or the wrong
`type:` label; lines parsed but never
poured means no scenario matches that
event type.

You now have the full detection loop:
acquisition, parsing, a bucket, an alert,
a decision. Next, the service everything
talks through — and how to interrogate an
alert properly.
