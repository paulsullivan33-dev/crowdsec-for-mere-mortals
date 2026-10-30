# The Hub — Parsers, Scenarios and Collections

Outcome: the hub index downloaded, the SSH
collection installed and read on disk, and
a clear picture of what "installing a
collection" actually changed — because
almost everything CrowdSec knows arrives
this way.

## The catalogue

The hub is a catalogue of configuration:
parsers that read logs, scenarios that
judge behaviour, postoverflows that clean
up after alerts (whitelists for search
engine crawlers and CDNs live here),
contexts that attach extra facts to alerts,
and collections that bundle them per
service. Refresh the local copy of the
catalogue with:

```bash
sudo cscli hub update
```

The lab's download reported the catalogue's
size as it loaded (transcript `t03`):

```text
Loaded: 171 parsers, 13 postoverflows, 790 scenarios, 10 contexts,
26 appsec-configs, 236 appsec-rules, 176 collections
```

The index is one JSON file
(`/etc/crowdsec/hub/.index.json`); items
themselves download on install, from the
hub's data host — including, for the SSH
collection, the GeoIP databases the
enrichers use. `cscli hub list` shows what
is *installed*; before anything is, it
answers "No items to display", which is a
statement about your disk, not about the
catalogue. `cscli hub search <word>` and
the per-kind list commands (`cscli parsers
list`, `cscli scenarios list`,
`cscli collections list`) show both states:
an item can be in the catalogue, installed,
or *enabled* — installed and enabled are
the same thing in practice, since install
enables.

## Installing a collection

Protect SSH with one command:

```bash
sudo cscli collections install crowdsecurity/sshd
```

On the lab's machine the broader
`crowdsecurity/linux` collection went in
first, and it matters that it did: linux
*bundles* sshd. The install printed an
action plan before touching anything
(transcript `t04`), and the plan is the
honest picture of what a collection is:

```text
collections: crowdsecurity/linux (0.4), crowdsecurity/sshd (0.9),
             crowdsecurity/whitelist-good-actors (0.4)
scenarios:   crowdsecurity/ssh-bf, ssh-slow-bf, ssh-time-based-bf,
             ssh-cve-2024-6387, ssh-refused-conn, ssh-generic-test
parsers:     crowdsecurity/syslog-logs, sshd-logs, sshd-success-logs,
             dateparse-enrich, geoip-enrich, public-dns-allowlist
postoverflows: rdns, cdn-whitelist, seo-bots-whitelist,
             google-special-crawlers-whitelist
contexts:    crowdsecurity/bf_base
```

Twenty-six items for one line of command:
two parsers to read the lines (syslog's
envelope, then sshd's own formats), six
scenarios to judge them (the brute-force
family, a slow-and-low variant, a
time-based variant, one for a specific
2024 SSH vulnerability, and two more),
enrichers for dates and geography,
whitelists so that well-known crawlers and
CDNs are not banned by accident, and a
context that records *which user names were
tried* into every alert — you will see
that context again in chapter 6, in a real
alert, listing `admin`, `oracle`, `test`,
`guest`.

Asking for `crowdsecurity/sshd` afterwards
answered `Nothing to install or remove.` —
already present, as part of linux. Remember
that membership; chapter 11 meets its
consequence when removing.

Each installed item lands on disk as a
readable YAML file:

```text
/etc/crowdsec/parsers/s00-raw/syslog-logs.yaml
/etc/crowdsec/parsers/s01-parse/sshd-logs.yaml
/etc/crowdsec/scenarios/ssh-bf.yaml
/etc/crowdsec/collections/sshd.yaml
```

Read them. They are the actual rules, not
documentation about rules — the scenario
chapter 5 leans on is nineteen lines long.
Items the hub manages should not be edited
in place (upgrades would fight you, and a
changed item becomes *tainted* — a state
chapter 11 shows for real); local additions
go in as separate files alongside, which is
how the project expects you to extend it.

## Installing is not loading

Every hub operation ends with the same
reminder in the lab's transcripts:

```text
Run 'sudo systemctl reload crowdsec' for the new configuration to be effective.
```

The running daemon loaded its parsers and
scenarios at startup. Files installed since
then are on disk but not in memory. On a
packaged install, `sudo systemctl reload
crowdsec` picks them up; forgetting this is
the single most common reason a fresh
collection "does nothing" — chapter 14
keeps it at the top of the list.

## What else is in the catalogue

The same pattern — one collection per
service — covers the common internet
exposure of a small server: web servers
(nginx, Apache, Caddy, Traefik), mail,
FTP, WordPress and other applications, and
generic collections for web attacks. There
are also collections whose job is purely
defensive bookkeeping, like the
whitelist-good-actors bundle the linux
collection pulled in. You do not install
the catalogue; you install the collections
for the services you actually run, because
every parser you load is work done on
every log line. The method does not change
per service: install the collection, give
the agent the service's logs (next
chapter), reload, and check the metrics
show the lines being parsed.

Next: the point of it all — a log file, an
attack, and an alert.
