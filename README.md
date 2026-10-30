# CrowdSec for Mere Mortals

Most break-in attempts on a server are not
clever. They are the same passwords tried
against the same SSH port, ten thousand
times a day, from addresses that were doing
exactly the same thing to somebody else
yesterday. **CrowdSec** is a free, open
source program that watches your server's
log files for that kind of behaviour,
recognises it, and produces a decision —
*ban this address for four hours* — that a
small separate program, a **bouncer**, then
enforces in your firewall. Its second idea
is the one in its name: machines that use
it can share what they see with a central
service, so an address that attacked a
server in another country this morning can
be on your blocklist before it reaches you.

This guide installs the real thing, feeds
it a real (invented) brute-force attack,
and watches every step: the log lines going
in, the alert coming out, the ban landing
in an actual firewall table, and the ban
being lifted again. CrowdSec itself blocks
nothing — the split between *detecting* and
*enforcing* is its central design decision,
and this guide keeps returning to it,
because most confusion about CrowdSec is
really confusion about which half did what.

The lab behind this guide ran genuine
**CrowdSec 1.8.2**, from the project's own
package repository: the `.deb` file's
SHA-256 hash was checked against the hash
the repository index publishes before
anything ran. The sandbox the lab works in
is shared with other tenants and cannot be
rebuilt for one guide, so the packages were
unpacked into a folder and run from there
rather than installed system-wide, with the
configuration's paths rewritten to match.
The parts that open network sockets or
touch a firewall ran inside a throwaway
network namespace — a private, empty
network created for the run and deleted
with it — because the sandbox's own
firewall is not the lab's to touch. Inside
that namespace the daemon listened on its
test address, 127.0.0.1:18085, and the
firewall bouncer built real nftables rules
that the lab could read back. Every command
in the guide was run against the official
binaries, unchanged; `criteria.md` keeps
the full ledger of what ran and what did
not, and the transcripts are in
`crowdsec-lab/`.

Against that setup the lab registered the
agent, downloaded the real hub catalogue
(790 scenarios, 176 collections), installed
the SSH collection, and fed the daemon a
thirteen-line log: ten failed SSH logins
from one address, one from another, one
successful login, and one line of junk. Two
failures produced nothing at all — the
guide shows the empty alert list. Ten
produced an alert and a four-hour ban.
The lab then added and deleted decisions by
hand, switched on notifications and read
the alert that arrived as a line of JSON,
registered the firewall bouncer and watched
the banned addresses appear inside an
nftables set with their expiry times
attached, and finally removed a collection
again — including the refusal you get first
if you ask the wrong way.

What the lab could not do, it did not fake.
Joining the central service — the Central
API, or CAPI — requires registering with an
email address you control and confirming
it, which is an account step, not a lab
step; the same is true of enrolling in the
web console, and of the community blocklist
that registration unlocks. So the sharing
half of CrowdSec is taught in chapter 10
from its verifiable edges: the exact status
messages the tools print when you are *not*
registered, the daemon's own log lines
saying the connection is off, and the
console's enrolment options listed by the
tool itself. The guide states plainly what
would leave your machine if you did
register, so you can decide with the facts
in front of you. The other notification
plugins (email, Slack, webhooks), the other
bouncers (nginx and friends), CrowdSec's
web-application firewall component, and
running the database on PostgreSQL instead
of the built-in SQLite are likewise
described, not run, and labelled as such.

One thing to know before you start:
CrowdSec is a detection engine with a
memory, not a magic shield. It reads logs —
so anything that does not get logged, it
cannot see. It decides — but until a
bouncer enforces the decision, nothing is
blocked. And its bans expire: the attacker
it stops today is welcome to try again in
four hours unless the evidence piles up.
Chapter 14 is honest about the rest of the
edges.

## What this guide covers

- [**What CrowdSec Is**](what-crowdsec-is.md) —
  the idea, the two halves (detection and
  enforcement), and the one question to ask
  of every claim: which half did that?
- [**The Moving Parts**](the-moving-parts.md) —
  the agent, the Local API, cscli, the hub's
  parsers and scenarios, profiles, and
  bouncers — and which part holds what
- [**Installing CrowdSec**](installing-crowdsec.md) —
  the official repository, what a real
  install does step by step, registering the
  machine, and the first checks
- [**The Hub — Parsers, Scenarios and Collections**](the-hub.md) —
  the catalogue, installing the SSH
  collection, what a collection pulls in,
  and why installing is not the same as
  loading
- [**Watching SSH — Your First Alert**](watching-ssh.md) —
  telling CrowdSec which logs to read, the
  leaky bucket, two failures (nothing) and
  ten (an alert and a ban), and proving each
  step from the outside
- [**The Local API and cscli**](the-local-api-and-cscli.md) —
  the service everything talks to, who is
  allowed to talk to it, and reading alerts
  in detail
- [**Decisions**](decisions.md) — bans,
  captchas and throttles; scopes and
  durations; adding and deleting decisions
  by hand; where decisions come from
- [**Profiles and Notifications**](profiles-and-notifications.md) —
  the file that turns alerts into decisions,
  longer bans for repeat offenders, and
  getting alerted yourself — a notification
  plugin run for real
- [**Bouncers — Making Decisions Bite**](bouncers.md) —
  registering the firewall bouncer, its
  nftables tables read back live, new
  decisions arriving on a poll, and the
  tables removing themselves on shutdown
- [**The Central API and the Console**](the-central-api-and-the-console.md) —
  the sharing half, honestly: what
  registration involves, what leaves your
  machine, what you get back, and the
  console's options as the tool lists them
- [**Keeping It Updated, and Removing It**](keeping-it-updated-and-removing-it.md) —
  hub updates, upgrading configurations,
  removing a collection (and the refusal),
  and uninstalling cleanly
- [**Recipes**](recipes.md) — ban an address
  now and by hand, test a scenario in
  simulation before trusting it, rehearse a
  new log type with `cscli explain`, and
  send alerts to a file you can grep
- [**The Neighbours**](the-neighbours.md) —
  Fail2Ban, firewalls, Wazuh and the
  intrusion-detection old guard: what each
  does that CrowdSec doesn't, and vice versa
- [**Limits**](limits.md) — it only sees what
  is logged, decisions are not enforcement,
  bans expire, and the rest of the edges
- [**Troubleshooting**](troubleshooting.md) —
  the alert that never fires, the explain
  that cannot connect, the collection that
  will not be removed, the bouncer with no
  key, and the traps this guide's lab fell
  into
- [**Glossary**](glossary.md) — the terms,
  collected in one place

## What it doesn't cover

Running CrowdSec as a business: the paid
console tiers, blocklist subscriptions and
the enterprise features are the project's
own shop window and are named, not taught.
The AppSec component — CrowdSec acting as a
web application firewall in front of a site —
is a second engine with its own rules and
its own guide's worth of material; here it
is introduced and bounded in chapters 2 and
14. Protecting services one by one beyond
SSH (each has a hub collection; the method
is the one chapter 5 teaches) is shown by
example, not repeated fifteen times.
And this guide does not tune detection for
a specific busy production server: the
defaults it teaches are the ones the hub
ships.

## How to use this guide

Read chapter 1 for the model and chapter 2
before installing anything — knowing which
part holds what turns later confusion into
a named problem. Chapters 3 and 4 get you
an agent and its rulebook; chapter 5 raises
your first alert in the same sitting, and
is the one to read slowly: its checks (the
empty list, then the full one) are how you
know detection works before you trust it
with a firewall. Chapters 7 to 9 turn
detection into enforcement. Read chapter 10
before you register for anything — it is
short, and it is the chapter the whole
crowd idea stands or falls on. The lab's
transcripts live in `crowdsec-lab/transcripts/`;
its attack log is `crowdsec-lab/auth-fixture.log`
and the sample lines used to test the parser
are `crowdsec-lab/sample-lines.txt`.

---

## Part of the series

This guide is one of the [For Mere Mortals](https://paulsullivan33-dev.github.io/mere-mortals/) guides: plain-language guides to the tools programmers and makers use.

Related guides:

- [Fail2Ban for Mere Mortals](https://paulsullivan33-dev.github.io/fail2ban-for-mere-mortals/)
- [UFW for Mere Mortals](https://paulsullivan33-dev.github.io/ufw-for-mere-mortals/)
- [SSH for Mere Mortals](https://paulsullivan33-dev.github.io/ssh-for-mere-mortals/)

## License

The writing in this guide is licensed under [CC BY 4.0](https://github.com/paulsullivan33-dev/crowdsec-for-mere-mortals/blob/main/LICENSE). Code examples and lab scripts are licensed under the [MIT License](https://github.com/paulsullivan33-dev/crowdsec-for-mere-mortals/blob/main/LICENSE-CODE).
