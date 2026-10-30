# Troubleshooting

Outcome: the failure in front of you has a
name, a check, and a next step. Every
entry here was met by this guide's lab or
follows from a mechanism the lab proved;
the lab's own traps are marked as such.

## Nothing alerts, ever

Work the pipeline in order — each step has
a check that splits the problem in half.

1. **Is the agent reading the file?**
   `cscli metrics`, Acquisition table:
   *Lines read* for your source. Zero means
   the acquisition file names the wrong
   path, or the daemon was not reloaded
   after you wrote it. (`systemctl reload
   crowdsec` — the step chapter 4 warned
   is forgotten most.)
2. **Are the lines understood?** Same
   table: *Lines unparsed*. All unparsed
   means the `type:` label is wrong for
   the file, or the service's collection
   is not installed. `cscli explain` a few
   real lines (recipe, chapter 12) and see
   which parser, if any, claims them.
3. **Are the events judged?** Parsed but
   never alerting means the bucket is not
   filling: check the scenario's capacity
   against your test (two failures will
   never trip a bucket of five — chapter
   5's first check was *supposed* to be
   empty), and check simulation mode:
   `cscli simulation status`. A simulated
   scenario alerts without deciding, by
   design.
4. **Alerts but no decisions?** The
   scenario's labels must ask for
   remediation, and a profile must match.
   `cscli alerts inspect` shows
   `Remediation:` and `Simulation:` — one
   of them is lying about what you
   intended.

## Decisions exist, but nothing is blocked

Detection has done its job; this is the
enforcement half. `cscli bouncers list`:
is your bouncer there, valid, and pulling
recently? Then the bouncer's own log, then
its machinery: for the firewall bouncer,
`nft list ruleset` should show its tables
and the address in a set. No tables: the
bouncer is down or keyless (below). Tables
but no address: the decision's *type* is
one this bouncer does not consume
(captcha at a firewall), or the decision
expired before the bouncer's first pull.

## cscli answers "connection refused"

```text
... Post "http://127.0.0.1:18085/v1/watchers/login": ...
dial tcp 127.0.0.1:18085: connect: connection refused
```

(the lab's test address; a packaged
install shows its own, 127.0.0.1:8080)

The daemon is not running (or not on the
address your credentials name). Alert,
decision, bouncer and explain commands all
talk to the Local API; this error is the
sound of nobody answering. Start or fix
the daemon first — `systemctl status
crowdsec`, then `/var/log/crowdsec.log`.
Note the trap's shape: some commands
(`cscli machines list`, the hub family)
read files directly and keep working,
which can make a dead daemon look like a
half-broken install. It is one breakage,
in one place.

*Lab trap, kept as evidence:* `cscli
explain` failed with exactly this error
when run with the daemon stopped
(transcript `t05b`), and worked with it
running (`t05`). Same command, same files —
the difference was the daemon.

## "online_api_credentials.yaml: no such file or directory"

```text
Error: cscli machines add: failed to load Local API: loading online
client credentials: open /etc/crowdsec/online_api_credentials.yaml:
no such file or directory
```

The file must *exist*; empty means "not
registered", missing means cscli stops
before it starts. A packaged install
maintains it; a hand-assembled one may
not. Create it empty if you do not intend
to register, or run `cscli capi register`
if you do. Met by the lab while
registering its machine by hand
(transcript `t02`).

## The bouncer starts and dies: "config does not contain LAPI key"

```text
level=fatal msg="unable to configure bouncer: config does not contain LAPI key or certificate"
```

The bouncer's config has an empty or
missing `api_key:` — typically a template
copied and never filled, or a key pasted
into the wrong file. Get a key with
`cscli bouncers add <name>` (it is shown
once), put it in the bouncer's own config,
and prove the config before starting the
service: run the bouncer by hand with
`-t` — the same fatal appears there, in
the open, instead of in a service log.
The lab's first bouncer attempt died
exactly here (its key extraction had
picked the wrong string out of cscli's
output); the second, with the real key,
passed `-t` cleanly and built its tables.

## nft says "No such file or directory" about a set

```text
$ nft list set inet crowdsec crowdsec-blacklists
Error: No such file or directory
```

Two mistakes compound here, both the
lab's (transcript `t10-nft-after-decision.txt`).
The bouncer's IPv4 table lives in family
`ip`, not `inet`. And the set's full name
carries its origin: `crowdsec-blacklists`
is the configured *prefix*; the sets are
`crowdsec-blacklists-crowdsec`,
`-cscli`, and so on. When in doubt, stop
guessing names: `nft list ruleset` prints
the whole truth.

## A collection refuses to be removed

```text
level=warning msg="crowdsecurity/sshd belongs to collections: [crowdsecurity/linux]"
```

The item is a member of another
collection; cscli will not orphan the
parent silently. Remove the parent, or
mean it explicitly with `--force` — and
expect the parent to show
`enabled,tainted` afterwards, which is the
listing telling the truth about the
surgery (chapter 11, transcript `t12`).

## The GEOIP columns are empty

Country and AS come from the GeoIP
enrichers and their databases, downloaded
with the hub items. Empty columns with
working alerts mean the enrichment found
no entry — genuinely the case for private
and documentation addresses (the lab's
fixture ranges have no geography, and its
tables are honestly blank) — or the
databases failed to download, which the
daemon's log records at startup. Check
there before doubting the parsers.

## A hub install "did nothing"

Reload. `sudo systemctl reload crowdsec`.
The daemon loaded its rulebook at startup;
files installed since are inert on disk.
This is the most common CrowdSec support
answer in existence, printed at the end of
every hub operation, and still the first
thing to check — the lab's transcripts
show the reminder arriving after *every*
install and removal, because it is always
relevant.
