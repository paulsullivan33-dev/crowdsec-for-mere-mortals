# Bouncers — Making Decisions Bite

Outcome: a bouncer registered, configured
and running; banned addresses visible
*inside the firewall itself*, with expiry
times; a new decision arriving while it
runs; and the firewall clean again when it
stops. This is the chapter where CrowdSec
starts dropping packets.

## Registering a bouncer

A bouncer needs its own credential: an API
key, created against the running daemon:

```bash
sudo cscli bouncers add my-firewall-bouncer
```

```text
API key for 'lab-firewall-bouncer':

   <a long random string>

Please keep this key since you will not be able to retrieve it!
```

That warning is literal: the key is shown
once; the API stores only what it needs to
check it. Lose it and the fix is to delete
the bouncer (`cscli bouncers delete`) and
add a new one. `cscli bouncers list` shows
the fleet — name, whether its key is
valid, when it last pulled, its version:

```text
 Name                  Valid  Last API pull  Type     Auth Type
 lab-firewall-bouncer  ✔️                                    api-key
```

(The lab's copy shows no pulls yet — the
screenshot predates the bouncer's first
run.) Keys authenticate; they do not
authorise much: a bouncer key can read the
decision stream and report back. It cannot
create decisions, read your logs, or
change configuration. A leaked bouncer key
is a nuisance; a leaked machine password
(chapter 6) is worse. Treat them
accordingly.

## The firewall bouncer

The package is
`crowdsec-firewall-bouncer-nftables` (or
`-iptables` for the older backend — same
program, different machinery underneath).
Its configuration,
`/etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml`,
is short. The lab's copy, key aside, is the
whole working set:

```yaml
mode: nftables
update_frequency: 5s
api_url: http://127.0.0.1:18085/
api_key: <the key from registration>
deny_action: DROP
blacklists_ipv4: crowdsec-blacklists
blacklists_ipv6: crowdsec6-blacklists
```

Notes that matter: `api_url` is the Local
API of chapter 6 (a packaged setup uses
port 8080). `update_frequency` is how
often it asks for changes — 10s by default;
the lab used 5s to make the polling
visible. `deny_action` defaults to DROP:
the packet vanishes and the sender learns
nothing; REJECT, which answers, is the
alternative the configuration offers.
Before first run, the bouncer can check
its own configuration:

```bash
crowdsec-firewall-bouncer -c /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml -t
```

Run it — the packaged systemd unit runs
exactly this test itself as a pre-start
step, so a bad config never half-starts.
It catches the classic failure —
a config with no key in it — with a fatal,
unambiguous message (the lab earned this
transcript the hard way, starting the
bouncer with an empty key first;
chapter 14 keeps it).

## What it builds in the firewall

Started against the lab's daemon — which
still held the chapter 5 ban, the chapter
7 manual ban, and one more — the bouncer
built this, read back with
`nft list ruleset` (transcript
`t10-nft-after-start.txt`, the IPv6 table
abbreviated):

```text
table ip crowdsec {
    set crowdsec-blacklists-cscli {
        type ipv4_addr
        flags timeout
        elements = { 198.51.100.77 timeout 58m35s, 203.0.113.99 timeout 1h57m6s }
    }
    set crowdsec-blacklists-crowdsec {
        type ipv4_addr
        flags timeout
        elements = { 203.0.113.17 timeout 3h57m1s }
    }
    chain crowdsec-chain-input {
        type filter hook input priority filter; policy accept;
        ip saddr @crowdsec-blacklists-cscli counter drop
        ip saddr @crowdsec-blacklists-crowdsec counter drop
    }
}
table ip6 crowdsec6 { ... }
```

Four facts to take from that listing.
**The decisions are in the kernel**, each
carrying its own countdown — the nftables
`timeout` on each element *is* the
decision's remaining duration, so expiry
would happen even if CrowdSec and the
bouncer both died. **The sets are split by
origin** (`-cscli`, `-crowdsec`; `CAPI`
would be a third), which is why chapter 7
told you to read the Source column.
**Everything lives in the bouncer's own
tables** (`crowdsec`, `crowdsec6`) — your
own firewall rules are untouched, and the
chain only drops listed source addresses.
And there are **counters**: the `processed`
counter counted the namespace's own
traffic during the lab (30 packets, then
73) while the blacklist counters sat at
zero — the fixture attackers, being
imaginary, sent no packets for it to drop.
On a real server those counters are the
proof of work.

Then, while it ran, one more decision was
added from cscli. On its next poll the
bouncer's log recorded `1 decision added`,
and the set showed the refreshed expiry.
No restart, no reload: enforcement follows
the API within seconds, in both directions
— decisions deleted in chapter 7 would
leave the set just as quietly.

## Stopping, and the shape of trust

Stopped, the bouncer removes its tables
(transcript `t10-nft-after-stop.txt`):

```text
$ nft list ruleset
(no output: the tables are gone)
```

No orphaned rules, no stale bans outliving
the service that made them — and,
symmetrically, no protection while it is
down, even though the decisions still
exist in the API. Bouncers are stateless
enforcers: on every start they pull the
full current decision stream and rebuild.
Run more than one — a firewall bouncer
*and* a web-server one — and each keeps
its own copy of the same list, in the only
form it can use.

## The other bouncers

The firewall bouncer is the generalist.
The hub and the package repositories offer
bouncers that live inside applications —
for nginx and Apache, for WordPress and
other web applications, for cloud
firewalls and CDNs. They exist because of
the decision types chapter 7 named: only a
bouncer that serves people can show a
`captcha`, only one inside the application
can act on a **username**, and only one at
the CDN edge can block traffic before it
reaches your network at all. They register
the same way, with their own keys, and are
described here rather than run — the
pattern, register → configure → poll →
enforce, does not change. Pick bouncers by
the decisions you intend to make: if every
decision you will ever issue is an IP ban,
the firewall bouncer is the whole story.

Next: the half of CrowdSec this guide has
deliberately kept off — the crowd.
