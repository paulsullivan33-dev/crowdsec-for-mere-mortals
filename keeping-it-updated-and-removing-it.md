# Keeping It Updated, and Removing It

Outcome: a routine for keeping the hub
rulebook current, a safe way to remove
pieces of it (including the refusal you
will meet first), and a clean uninstall
that leaves neither rules nor state behind
— or keeps exactly the state you meant to.

## Two things update, on two clocks

**The software** updates like any package:
the project's repository publishes new
releases, and `apt upgrade` brings them.
**The rulebook** — hub items — updates
separately, and this is the part people
forget, because nothing about the *program*
looks stale while its scenarios age.

```bash
sudo cscli hub update     # refresh the catalogue itself
sudo cscli hub upgrade    # bring installed items to their latest versions
```

A packaged install enables a
`crowdsec-hubupdate.timer` that refreshes
the index on a schedule; upgrading the
items themselves is the deliberate step
above (per-kind variants exist —
`cscli collections upgrade`, and friends).
As with installing, upgraded files take
effect on reload: `sudo systemctl reload
crowdsec`. New scenarios arrive this way —
the catalogue's 790 are a moving target,
and a server installed a year ago is
judging with a year-old idea of attacks
until somebody runs these two commands.

## Tainted items

An item you have edited in place is
**tainted**: the hub still manages it, but
your copy no longer matches the published
one, upgrades treat it with suspicion, and
listings flag it. You will see the state
in a moment. The rule is simple: don't
edit hub files; add local files beside
them (a parser or scenario with your own
name, in the same folders) and let the hub
keep its own. Local files are never
tainted, never upgraded, and never
surprised by an upgrade.

## Removing a collection: expect the refusal

Removing is where membership, from
chapter 4, collects its debt. The lab
asked for the sshd collection back
(transcript `t12`):

```text
$ cscli collections remove crowdsecurity/sshd
level=warning msg="crowdsecurity/sshd belongs to collections: [crowdsecurity/linux]"
level=warning msg="Run 'sudo cscli collections remove crowdsecurity/sshd --force'
if you want to force remove this collection"
Nothing to install or remove.
```

That refusal is good design, not friction:
sshd arrived as part of linux, and silently
ripping a member out would leave the parent
claiming to protect something it no longer
does. The forced removal works — and
leaves the evidence of the surgery in the
listing:

```text
$ cscli collections remove crowdsecurity/sshd --force
❌ disable
 collections: crowdsecurity/sshd
 scenarios: crowdsecurity/ssh-bf, ssh-slow-bf, ssh-time-based-bf,
            ssh-cve-2024-6387, ssh-refused-conn, ssh-generic-test
 parsers: crowdsecurity/sshd-logs, sshd-success-logs

$ cscli collections list
 crowdsecurity/linux   ⚠️ enabled,tainted   0.4
```

`enabled,tainted` is the parent's honest
new state: still enabled, no longer as
published. Reinstalling the member
(`cscli collections install crowdsecurity/sshd`)
restores it to untroubled enabled. Reload
the daemon either way; until you do, the
running agent keeps judging with the
scenarios you just removed from disk.

Removing single parsers and scenarios
works the same way (`cscli scenarios remove
…`), with the same respect for membership:
items that belong to an installed
collection ask you to mean it.

## Uninstalling CrowdSec itself

In the other direction — the whole tool —
the order matters:

1. **Stop the bouncers first**, or let the
   package manager stop them with their
   service. A stopped firewall bouncer
   removes its own tables (chapter 9); a
   bouncer left running against a dead API
   keeps enforcing the last decisions it
   pulled, which is either a safety net or
   a haunting, depending on whether you
   knew.
2. **Remove the packages** (`apt remove
   crowdsec crowdsec-firewall-bouncer-nftables`).
   The service stops; the firewall bouncer
   cleans its tables on the way down.
3. **Decide about the state.** A plain
   remove leaves `/etc/crowdsec/` and
   `/var/lib/crowdsec/` in place — your
   configuration, your hub items, and the
   database with its alert history survive
   for a reinstall. `apt purge` removes the
   package-owned configuration; deleting
   `/var/lib/crowdsec/data/` by hand is
   what actually destroys the history, and
   removing `/etc/crowdsec/online_api_credentials.yaml`
   is what unregisters the machine locally.
   If the machine was enrolled in the
   console, remove it there too — a
   dashboard entry is also state.
4. **Check the firewall.** `nft list ruleset`
   should show no `crowdsec` tables. If it
   does — a bouncer killed ungracefully —
   the tables are self-contained and safe to
   delete by name (`nft delete table ip crowdsec`,
   and `ip6 crowdsec6`).

The lab's copy needed none of this — its
"CrowdSec" was a folder, deleted with the
lab — but every step above is the package
route's own machinery, and the two cscli
halves of it (hub removal, credential
files) are the ones the lab ran for real.

Next: recipes — the short forms of the
things you will actually do on a Tuesday.
