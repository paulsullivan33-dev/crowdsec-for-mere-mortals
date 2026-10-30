# The Central API and the Console

Outcome: you know exactly what joining the
crowd involves, what data leaves your
machine if you do, what you receive in
return, and how to check the state of all
of it — because this is the chapter where
CrowdSec stops being only yours.

This chapter is different in kind from the
rest: registration is an account with an
outside service, confirmed by email, so
the lab did not perform it, and nothing
here pretends otherwise. What the lab
*could* do, it did: it ran every status
command in the not-registered state and
kept the answers verbatim (transcript
`t11`). The mechanics of what registration
starts are described from those edges and
the project's own configuration, and are
labelled documented-only in `criteria.md`.

## The off state, verbatim

Out of the box, the daemon tells you the
crowd is off, twice, in its own log:

```text
msg="push and pull to Central API disabled"
msg="Communication with CrowdSec Central API disabled from configuration file"
```

And cscli, asked directly:

```text
$ cscli capi status
Error: cscli capi status: the Central API (CAPI) must be configured
with 'cscli capi register'

$ cscli console status
| Central API (CAPI) | ❌ not registered, see 'cscli capi register' |
```

The on-disk marker of the off state is the
empty file from chapter 3:
`/etc/crowdsec/online_api_credentials.yaml`.
Registration fills that file with the
machine's Central API credentials; until
then there is nothing to phone home with.
Local detection, decisions and bouncers do
not consult it at all.

## Registration: what it is

`cscli capi register` asks for an email
address, sends the machine's credentials
request to the Central API, and waits for
you to confirm from the mailbox — the same
email round-trip as any account signup,
because that is what it is: an account,
keyed by email, that your machines belong
to. The packaged installer offers to do it
during installation (chapter 3); doing it
later is the same command. Nothing about
your local configuration changes: same
parsers, same profiles, same bouncers.
Two flows simply switch on.

## What leaves the machine

**Push.** When one of your scenarios
raises an alert, the agent sends a
*signal* to the Central API: the attacking
address, the scenario that fired, the
timestamps and event counts, the country
and network enrichment — the shape of
chapter 6's inspect output. Not sent: your
log lines, your file paths, your users'
names beyond what the scenario's context
holds (the lab's alert context carried
target user names the *attacker* tried —
admin, oracle — which are the attacker's
guesses, not your accounts). The agent
also prepares anonymous usage metrics —
counts of alerts, decisions and API calls,
of the kind chapter 6's metrics table
shows being posted to the Local API — for
the console's statistics. The console's
enrolment options (below) let you narrow
which alert classes are forwarded at all.

**Pull.** In return, your agent
periodically downloads the **community
blocklist**: decisions distilled by the
Central API from the signals of the whole
crowd, weighted by how many independent
machines reported an address and how
recently. On your machine they appear as
ordinary decisions with origin **CAPI** —
which is why chapter 9's firewall sets
have a per-origin split: on a registered
machine, a third set, for the CAPI origin,
sits beside the others, banning addresses
that have never touched your server, on
the crowd's say-so. That is the entire bargain,
stated as plainly as it can be: your
attackers become everyone's blocklist, and
everyone's attackers become yours.

## The console

The web console (app.crowdsec.net) is a
dashboard over the same account: your
machines, their alerts and decisions, the
blocklist, and — in the paid tiers —
extras like more capable blocklists and
longer history. Enrolling a machine is a
second, separate step after CAPI
registration: `cscli console enroll` takes
a token the console gives you. The
enrolment *options* are visible without an
account, and the lab kept the tool's own
table (transcript `t11`):

```text
| Option Name | Activated | Description                                          |
| custom      | ✅        | Forward alerts from custom scenarios to the console  |
| manual      | ❌        | Forward manual decisions to the console              |
| tainted     | ✅        | Forward alerts from tainted scenarios to the console |
| context     | ❌        | Forward context with alerts to the console           |
```

Read the defaults: alerts from your own
custom scenarios and from hub scenarios
you have modified (tainted) are forwarded;
the decisions you make *by hand* and the
alert **context** (chapter 6's dossier —
the tried user names, and whatever else
your contexts collect) are **not**, unless
you turn them on. Those two off-by-default
switches are the console's privacy posture
in miniature, and they are yours to set
per machine with `cscli console` toggles.

## The decision, framed

Register if the blocklist is worth more
than the signals cost you — for most small
internet servers it is a genuinely good
trade: the crowd sees vastly more attacks
than any one machine, and what you send is
a narrow, factual summary of attacks you
actually suffered. Decline, or delay, if
your alerts are themselves sensitive — a
scenario's name can reveal what you run,
and a signal proves an attack reached you.
Either way, notice that the choice is
*only* about sharing: everything this
guide built in chapters 3 to 9 — detection,
decisions, enforcement, notifications —
stands complete without it. CrowdSec the
local tool does not nag, degrade or phone
home unregistered; the lab's daemon log
says "disabled", and disabled is a
perfectly respectable steady state.

Next: keeping the rulebook fresh, and the
way out.
