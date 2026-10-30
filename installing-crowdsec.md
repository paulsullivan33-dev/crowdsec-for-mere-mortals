# Installing CrowdSec

Outcome: CrowdSec installed from the
project's own repository, the agent
registered with its own Local API, and the
first checks run — the checks, not the
installer's exit code, are what prove it.

## Where to get it

Two honest routes:

- **The project's repository** (packagecloud,
  for Debian and Ubuntu; there are also RPM
  repositories and packages for other
  systems). This is what the lab used and
  what this guide teaches: it carries the
  current release — 1.8.2 when the lab ran.
- **Your distribution's own packages.**
  Ubuntu 24.04 ships CrowdSec too, but an
  older line (1.4.6). It works, and its
  cscli differs in places from this guide's
  transcripts. Prefer the project's
  repository unless you have a reason not
  to; the project publishes install scripts
  that add the repository and install in
  one step:

```bash
curl -s https://install.crowdsec.net | sudo sh
sudo apt install crowdsec
```

Read any script you pipe into a root shell
first — this one is short, and reading it
is part of the install. The firewall
bouncer is a separate package
(`crowdsec-firewall-bouncer-nftables` or
`-iptables`, chapter 9); the agent alone
detects and decides, and installing it
first, on its own, is the right order.

However you obtained it, verify the
provenance the way your route allows —
signed repository metadata for apt, or,
for a file in hand, the hash the index
publishes. The lab downloaded the `.deb`
files directly and checked both SHA-256
hashes against the repository index before
running anything (transcript `t00`); the
first, parallel download arrived truncated,
failed the check, and was discarded. That
is what the check is for.

## What a real install does

The package is not just binaries. Reading
its installer script (the lab did — it is
part of the package) shows a real install
doing, in order:

1. Laying down the binaries (`crowdsec`,
   `cscli`), the configuration tree under
   `/etc/crowdsec/`, the grok patterns, and
   the notification plugins under
   `/usr/lib/crowdsec/plugins/`.
2. Asking, through the package's questions,
   whether to register with the Central API
   (chapter 10 — the default answer is
   safe to accept or decline; the lab
   declined, and everything local still
   works).
3. Registering the machine with the Local
   API — `cscli machines add --auto` — and
   writing the agent's credentials to
   `/etc/crowdsec/local_api_credentials.yaml`.
4. Downloading the hub index and upgrading
   hub items (`cscli hub update`,
   `cscli hub upgrade`).
5. Running `cscli setup unattended`, which
   detects the services on the machine
   (an SSH daemon, a web server…) and
   installs the matching hub collections
   and acquisition files for them.
6. Enabling and starting the `crowdsec`
   service, plus a `crowdsec-hubupdate.timer`
   that refreshes the hub index on a
   schedule.

So after a normal install, an SSH server
is usually already being watched: the
setup step did chapter 4 and half of
chapter 5 for you. This guide still does
those chapters by hand, because "it was
set up automatically" is not the same as
understanding it — and the manual route is
what you need the first time the automatic
one guesses wrong.

## The lab's parallel route

The guide's lab could not install packages
into its shared sandbox, so it unpacked
the same two `.deb` files into a folder
(`dpkg-deb -x`), rewrote the paths in
`config.yaml` to point into that folder,
registered the machine with the same
`cscli machines add --auto`, and ran the
daemon inside a throwaway network
namespace on the address 127.0.0.1:18085
(a packaged install uses 127.0.0.1:8080).
One genuine wrinkle fell out of doing it
by hand, and it is worth knowing on a real
machine too: the tools expect a file
`/etc/crowdsec/online_api_credentials.yaml`
to *exist*. Registration with the Central
API fills it; an empty file is the
on-disk way of saying "not registered",
and if the file is missing entirely,
cscli stops with:

```text
Error: cscli machines add: failed to load Local API: loading online
client credentials: open /etc/crowdsec/online_api_credentials.yaml:
no such file or directory
```

Creating the empty file fixed it, and the
registration went through (transcript
`t02`):

```text
$ cscli machines add --auto
Machine '0a3408554c4d4e6dbc67faae93a57b41y4iqFAIhDTwt0b72' successfully added to the local API.
API credentials written to '/etc/crowdsec/local_api_credentials.yaml'.
```

A packaged install creates all of this for
you; the lab's transcript exists so that
if you ever see the error, you know which
missing piece it names.

## First checks

Three checks, in order. Each has a stated
expected result — if yours differs, stop
and read the log before going on.

**1. The versions agree.**

```bash
crowdsec --version
cscli version
```

Both should print the same release (lab:
`v1.8.2`, transcript `t01`). `cscli --help`
lists the command families — alerts,
decisions, hub, machines, bouncers,
metrics, and more. You will use about six
of them constantly.

**2. The machine is registered and the
daemon answers.**

```bash
sudo cscli machines list
sudo systemctl status crowdsec
```

The list shows your machine with a recent
heartbeat once the daemon is running.
`cscli machines list` reads the database
directly, so it can work with the daemon
stopped; the commands that *cannot* are
the ones in chapter 6, and their failures
are a feature — they tell you the daemon
is down before anything else does.

**3. The daemon's own log says the API is
listening.**

```bash
grep "Local API listening" /var/log/crowdsec.log
```

The lab's daemon printed
`CrowdSec Local API listening on 127.0.0.1:18085`
(its test address; a packaged install
prints 8080), alongside a line worth
recognising early:

```text
msg="push and pull to Central API disabled"
```

That is the not-registered state, stated
by the daemon itself. It is not an error.
Chapter 10 is about the day you choose to
change it.

## Where things are, from here on

```text
/etc/crowdsec/config.yaml                     the agent's configuration
/etc/crowdsec/acquis.yaml, acquis.d/          what it reads
/etc/crowdsec/profiles.yaml                   alerts → decisions
/etc/crowdsec/local_api_credentials.yaml      the agent's own API login
/etc/crowdsec/online_api_credentials.yaml     Central API login (or empty)
/etc/crowdsec/{parsers,scenarios,collections}/  hub items, installed
/var/lib/crowdsec/data/crowdsec.db            the database
/var/log/crowdsec.log                         the agent's log
```

Next: the hub — where the parsers and
scenarios that make detection mean anything
come from.
