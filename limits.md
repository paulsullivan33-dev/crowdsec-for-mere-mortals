# Limits

Outcome: you can state, without being
prompted, the six edges of this tool —
so that what it never promised is never
mistaken for a malfunction.

**It only sees what is logged.** CrowdSec
reads logs. An attack against a service
that logs nothing, logs elsewhere, or logs
in a format no parser knows is invisible
to it — not missed, *invisible*, and the
metrics' unparsed count is the only
witness. The network-level neighbours in
chapter 13 exist because this edge is
real. Every service you care about needs
two things verified, not assumed: that it
logs the events you care about, and that
a parser claims them (chapter 5's metrics
check).

**Detection is not enforcement.**
CrowdSec itself blocks nothing. An alert
is a record; a decision is a row; only a
running bouncer turns either into a
dropped packet, and each bouncer enforces
only the decision types it can express —
the firewall bouncer will never show a
captcha, and decisions of a type no
bouncer consumes are intentions, not
actions. Half of this guide's
troubleshooting is locating which side of
that line a failure lives on.

**Bans expire, by design.** Four hours is
the default; longer durations and the
repeat-offender expression exist, but
there is no "forever" in the model, and
an attacker with patience or a pool of
addresses treats bans as weather. The
community blocklist widens the net; it
does not change the nature of the net.
CrowdSec raises the attacker's cost and
cleans your logs. It does not end the
attacking.

**The crowd is a trust decision.** The
blocklist bans addresses your machine has
never seen, on other people's evidence,
weighted by a service you do not control.
False positives are possible in both
directions: a shared or reassigned address
can arrive pre-banned, and an address
*you* ban is reported onward. Chapter 10
is the full accounting; the limit to
carry away is that CAPI decisions are
hearsay — usually excellent hearsay, and
hearsay all the same. Your own scenarios'
false positives are the mirror image: a
scenario measures what it measures, and a
bucket cannot tell a botnet from a
company's whole office NAT forgetting its
password at once. Hence simulation mode,
hence `alerts inspect` before outrage.

**The Local API is a trust boundary.**
Everything CrowdSec knows and does is
behind one API. Loopback-only and
`trusted_ips` make the default posture
sound; the moment you expose the API to a
network — a legitimate shape, for
multi-machine setups — you own its
authentication, its TLS, and its firewall
rules, and a machine credential is a key
to the whole building: it can post alerts
and, through them, cause bans. Bouncer
keys, by contrast, are read-mostly and
scoped. The design gives you the
distinction; keeping it is your job.

**It is a reflex, not a record.**
Retention is finite (the database flush
defaults bound the history), the console's
long memory is a paid and off-machine
thing, and nothing here replaces logging,
monitoring or backups. CrowdSec will not
tell you what happened last month, prove
compliance, or notice the quiet,
successful login hidden among the failed
ones — its scenarios judge patterns of
*recorded misbehaviour*, and the one
polished, successful intrusion is
precisely the event with no pattern at
all. Pair it with the watchers of chapter
13 and with logs you actually read.

And the small print, honestly: hub
scenarios are other people's judgement,
versioned and updated — reading the ones
you rely on (they are short) is part of
operating the tool; AppSec, the
web-request engine, is a second system
this guide bounded rather than taught;
and performance, while a non-issue at the
scale of this guide, is not free — every
parser runs on every line, and a very busy
service wants its acquisition scoped and
its metrics watched.
