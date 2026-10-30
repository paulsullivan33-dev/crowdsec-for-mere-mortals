# What CrowdSec Is

Outcome: by the end of this chapter you can
say what CrowdSec does, what it pointedly
does *not* do, and why the difference
matters every time something later seems
not to work.

## The problem it solves

Leave a server on the internet for a day
and read its SSH log. Among the real logins
you will find hundreds or thousands of
failed ones you do not recognise: passwords
tried for `root`, for `admin`, for users who
have never existed. This is not someone
targeting you personally. It is machinery,
sweeping every address on the internet,
trying the passwords that worked somewhere
else. The defence is boring and known:
strong passwords or keys, and making the
machinery go away when it misbehaves.

Making it go away is the job. A human
cannot watch a log at 3 a.m. A simple
script can — that idea is decades old, and
chapter 13's first neighbour, Fail2Ban, has
done it since 2004. CrowdSec is the modern,
larger answer to the same idea, with two
things added: a proper separation between
*noticing* and *acting*, and a crowd.

## The two halves

CrowdSec is built as two halves that never
meet except through a list of decisions:

- **The agent detects.** It reads log files
  (or other sources), recognises patterns of
  bad behaviour — five failed logins from one
  address in under a minute, say — and
  records an **alert**. Following the
  **profiles** you configure, an alert
  becomes a **decision**: *ban address
  203.0.113.17 for four hours*. The agent
  writes the decision to its database and
  stops there. It has no firewall rules, no
  packet filter, no power to block anything.
- **Bouncers enforce.** A bouncer is a small
  separate program — for the firewall, for a
  web server, for a content-management
  system — that asks the agent's API, over
  and over, "what is currently decided?",
  and applies those decisions with whatever
  power *it* has. The firewall bouncer
  writes the banned addresses into an
  nftables set, and the kernel drops their
  packets.

Everything useful in this guide hangs on
that split. If alerts appear but nothing is
blocked, detection is fine and enforcement
is missing — a bouncer problem. If a
bouncer is running but the decision list is
empty, enforcement is fine and detection
saw nothing — a log or rules problem.
Chapter 14 returns to this; hold on to it
from here.

## The crowd

The second idea is sharing. A single
server sees only the attacks aimed at it.
CrowdSec's central service — the **Central
API**, usually shortened to CAPI — lets
agents report the addresses they have
caught (as sanitised signals: the address,
the scenario that fired, the time — not
your log lines) and, in return, pull a
**community blocklist**: decisions distilled
from what the whole crowd has reported.
An address that spent the morning attacking
servers elsewhere can already be banned on
yours when it arrives.

That sharing is optional and off until you
register, and it has a price worth naming
plainly: to receive the crowd's signals you
send your own. Chapter 10 lays out exactly
what leaves the machine, what comes back,
and what registration involves, with the
status messages the tools print when it is
off — which, in this guide's lab, it
deliberately stayed. Detection and local
decisions, everything in chapters 3 to 9,
work completely without it.

There is also a web **console**
(app.crowdsec.net) where registered
machines can be viewed together, and paid
tiers beyond the free one. The console is a
window onto the same machinery, not a part
of it; your server keeps working if you
never open it.

## What kind of tool it is

CrowdSec is free and open source (MIT
licence), written in Go, and runs as a
small set of programs: the `crowdsec` agent,
the `cscli` command-line tool you manage it
with, and whichever bouncers you add.
Its rules — how to read each kind of log,
what counts as an attack — are not compiled
in. They are data files downloaded from a
catalogue called the **hub**, written by
the project and its community, and updated
like packages. Chapter 4 is about the hub;
the point for now is that "CrowdSec knows
how SSH brute force looks" really means
"a scenario file on your disk says what SSH
brute force looks like", and you can read
that file.

What it is not:

- **Not a firewall.** It has no rules of
  its own about who may connect. Your
  firewall keeps doing that job; the
  bouncer just hands it a changing list of
  addresses to drop.
- **Not an antivirus or a file scanner.**
  It reads behaviour in logs, not files on
  disk.
- **Not a guarantee.** It sees only what is
  logged, decides only what its scenarios
  describe, and its bans expire by design.
  An attacker with a fresh address and a
  slow hand is a different problem, and
  chapter 14 says so without flinching.

## The shape of the rest of the guide

Detection first, enforcement second, the
crowd last — the same order the system
itself is built in. Next chapter: the
moving parts, one by one, with what each
one holds and what each one is for.
