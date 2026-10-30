# The Neighbours

Outcome: you can place CrowdSec among the
tools it resembles — and say, for each,
what it does that CrowdSec doesn't, and
the reverse. These neighbours are
characterised, not benchmarked; where a
neighbour has its own guide in this
series, it is named.

## Fail2Ban — the parent

[Fail2Ban for Mere Mortals](https://paulsullivan33-dev.github.io/fail2ban-for-mere-mortals/)
is the tool CrowdSec descends from in
spirit: a daemon that scans logs with
regular expressions (its *filters*),
counts failures per address, and bans by
writing firewall rules itself (its
*actions*). The differences are of shape,
not of idea:

- **One program, no split.** Fail2Ban
  detects *and* enforces; there is no API,
  no separate bouncer, no decisions list to
  inspect. Simpler to hold in your head;
  nothing for a second machine or a web
  layer to consume.
- **Local rules.** Its filters and jails
  are yours, on your box, in a config
  language of regexes. CrowdSec's are a
  shared, versioned catalogue from the hub
  — less yours, more maintained.
- **No crowd.** Fail2Ban has no central
  service, no blocklist, no console. What
  your machine sees is all it knows.
- **Footprint and fuss.** Fail2Ban is a
  small Python program that asks nothing.
  CrowdSec is a small platform.

If one SSH daemon on one server is the
whole problem, Fail2Ban is honestly
enough, and saying so costs this guide
nothing. CrowdSec earns its keep as the
estate grows: more services, more
machines, a wish for shared intelligence
and an API other tools can use.

## The firewall itself

[UFW for Mere Mortals](https://paulsullivan33-dev.github.io/ufw-for-mere-mortals/)
and its nftables/iptables underpinnings
answer a different question: not "who has
misbehaved?" but "who may connect at all?"
A firewall's rules are *standing policy* —
the SSH port is open to the world, or to
three addresses, forever, regardless of
behaviour. CrowdSec's decisions are
*reactions*: temporary, evidence-based,
expiring. They compose rather than
compete — chapter 9's bouncer is literally
a program that writes one tool's output
into the other — and a server with a sane
firewall and no CrowdSec is far safer than
the reverse.

## SSH hardening

Most of the brute force in chapter 5 is
only possible because password logins are
switched on. [SSH for Mere Mortals](https://paulsullivan33-dev.github.io/ssh-for-mere-mortals/)
covers the structural answers — key-only
logins, no direct root login — after which
the password-guessing crowd is knocking on
a door that does not open for passwords at
all. Do that first. What remains for
CrowdSec is everything hardening cannot
remove: the other services, the other log
files, and the noise reduction of banning
persistent knockers so your logs stay
readable. Detection is the second layer,
not a substitute for the first.

## Wazuh and the SIEM end of the street

Wazuh (the maintained descendant of
OSSEC) watches logs too — and files,
registry keys, running processes, and
vulnerabilities, across fleets of agents,
into a central indexer with dashboards and
compliance reports. It is a security
*observatory*; CrowdSec is a security
*reflex*. Wazuh will tell you everything
about the attack next week in a beautiful
timeline; CrowdSec's bouncer dropped the
attacker's packets in the first ten
seconds and has nothing more to say. Many
shops run both, for exactly those reasons.
Choosing Wazuh *instead* is choosing
visibility over reaction; choosing
CrowdSec instead is the reverse. Know
which problem is yours.

## Suricata and Snort — watching the wire

Network intrusion detection watches
*packets*, not logs: signatures and rules
matched against traffic as it crosses the
wire, including traffic for services that
log badly or not at all. That vantage sees
things CrowdSec structurally cannot
(chapter 14), at the price of sitting in
the traffic path, drowning in encrypted
payloads it cannot read, and generating
alerts a human must staff. CrowdSec reads
the diary after the fact, in the service's
own words, and acts on its own. Different
sense organs; the comparison is anatomy,
not quality.

## Prometheus and Grafana — who watches this, then?

CrowdSec exposes metrics (chapter 5's
cscli tables are the human form of them)
and [Prometheus and Grafana for Mere Mortals](https://paulsullivan33-dev.github.io/prometheus-and-grafana-for-mere-mortals/)
is the series' way to graph them: alerts
per day, decisions by origin, the
unparsed-line count that warns you a log
format changed underneath a parser. Add
to that an uptime check on the daemon and
the bouncer, and the reflex has a
supervisor. None of it is required; all of
it is the difference between a tool you
installed and a control you operate.
