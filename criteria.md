# Writing criteria for this guide

Every "for Mere Mortals" guide is written to the same nine-point
standard, so a reader who trusts one can trust the rest. Here it
is, with the verification notes for this particular guide below.

1. **Fix it first.** Reader outcomes lead, not action items: each
   chapter opens by naming the outcome it delivers, shows a check
   the reader can run to confirm they got it, and a reasonable
   reader finishes able to state what was done, what it means,
   and what to try next.
2. **Plain language.** Short sentences, everyday vocabulary,
   technical terms defined on first use. No slang, and no
   idioms that only make sense in one country — the reader may
   be translating as they go.
3. **No dead ends.** A guide never tells the reader to do
   something it cannot show them how to verify. Every claim a
   reader relies on — an install, a command, an output — is
   something this guide demonstrates or clearly labels
   documented-only below.
4. **Live-verified where it matters.** Procedures are run for
   real in the guide's lab wherever the build machine allows it,
   and the transcripts kept in the repository are the evidence.
   What could not be run is listed honestly below, never
   presented as if it had been.
5. **General audiences.** Nothing in this guide — examples,
   fixtures, names, addresses — is modelled on any particular
   reader or author's real life, network, or organisations.
   The attack log uses documentation-only address ranges and
   invented names throughout.
6. **Standalone.** A reader needs this guide, its lab, and
   nothing else. Where another guide would genuinely help, it
   is pointed to once, by name, and the chapter still completes
   without it.
7. **Honest about limits.** Chapter 14 names what the tool
   cannot do: it sees only what is logged, it enforces nothing
   itself, its bans expire by design, the crowd is hearsay, the
   Local API is a trust boundary, and a reflex is not a record.
8. **The ethics rule is in the text, early.** CrowdSec's
   sharing half moves data off the reader's machine, so the
   rule is stated in chapter 1 and accounted in full in chapter
   10: what a registered machine sends (sanitised signals —
   addresses, scenarios, counts; never log lines), what it
   receives, which console forwarding switches exist and how
   they default. Registration is an account with an outside
   service and is presented as a decision, not a default.
9. **Builds clean.** The site builds with MkDocs in strict mode
   with no warnings, and a fresh reader following the chapters
   in order reaches the stated outcome of each.

## Verification notes for this guide

The build machine is a shared Ubuntu 24.04 sandbox whose holder
cannot let one guide rebuild the system, and whose own firewall
the lab had no business touching (the host refuses `nft` calls
outright). CrowdSec **1.8.2** and the firewall bouncer
**0.0.36** were therefore obtained as the official `.deb`
packages from the project's own packagecloud repository, their
SHA-256 hashes checked against the repository index before use
(transcript `t00` — a first, truncated download failed that
check and was discarded), unpacked with `dpkg-deb -x` into a
scratch tree, and run from there with the stock `config.yaml`'s
paths rewritten into the scratch tree and the Local API moved
to 127.0.0.1:18085. Parts that open sockets or touch a firewall
ran inside a throwaway network namespace created for the run
(`unshare -Urn`): a private loopback and a private nftables,
both deleted with the namespace when each phase ended. No
package was installed system-wide, no other tenant's process
was touched, and every daemon the lab started was stopped by
the lab. On a reader's machine a packaged install (chapter 3)
does all of this under systemd with the same files in their
standard places; the guide teaches that path and uses the lab's
only where the two genuinely differ.

Run for real, with transcripts in `crowdsec-lab/transcripts/`
as evidence:

- Versions and the full cscli command surface (t01).
- Machine registration (`cscli machines add --auto`,
  `machines list`), including the missing-file error that
  preceded it and its empty-file fix (t02).
- `cscli hub update` against the live hub — 171 parsers, 790
  scenarios, 176 collections, and the rest — and installing
  `crowdsecurity/linux` (which bundles `crowdsecurity/sshd`)
  with its full action plan; GeoIP databases downloaded by the
  install (t03, t04).
- `cscli explain` of failure and success lines through the
  real parser stages into the scenario list (t05), and its
  verbatim failure with the daemon stopped (t05b).
- The daemon tailing a fixture log: two failures → no alert
  (t06-alerts-before); ten failures → one ssh-bf alert with
  exactly six events and a 4h ban (t06-alerts-after), inspected
  in full, context included (t06-alert-inspect).
- Manual decisions from cscli: a 2h ban and a 1h captcha added,
  listed, and the captcha deleted (t07). Simulation enabled
  and disabled for one scenario (t07b).
- `cscli metrics` accounting for all 13 fixture lines —
  12 parsed, 1 unparsed, 51 poured — plus parser, alert,
  decision and Local API route tables (t08).
- The file notification plugin, enabled in the profile,
  writing real JSON alert lines for all five lab alerts,
  including the manual ones (t09).
- The firewall bouncer end to end: registration and the
  one-time key (redacted in the transcript), `-t` config test,
  its nftables tables and per-origin sets read back with the
  live decisions and their timeouts inside, a further decision
  arriving on a poll ("1 decision added"), and the tables
  removed by the bouncer on shutdown (t10 series). The first
  attempt's failure — a config with an empty key, fatal message
  verbatim — is in chapter 14 and the same transcripts.
- The not-registered state of the crowd half, verbatim:
  `cscli capi status`, `cscli console status` with its
  enrolment-options table, `cscli console enroll` with no
  token, and the daemon's own "push and pull to Central API
  disabled" log lines (t11, t06-daemon-start).
- Removing a hub member: the refusal without `--force`, the
  forced removal, and the parent collection left
  `enabled,tainted` (t12).

Documented only — described from the shipped files, the
package's own installer script, or the project's documentation,
not run, and the text does not claim otherwise:

- The packaged installation itself (the lab read the `.deb`'s
  postinst to describe its steps faithfully: debconf question,
  machine registration, hub update and upgrade, `cscli setup
  unattended`, service and hub-update timer enablement).
- CAPI registration and everything downstream of an account:
  the confirmation email round-trip, signal push in production,
  the community blocklist arriving as CAPI-origin decisions,
  console enrolment with a real token, and the console's paid
  tiers.
- The notification plugins other than `file` (email, http,
  slack, splunk, sentinel) — same broker mechanism, configs
  read from the shipped templates.
- Bouncers other than the nftables firewall bouncer (the
  iptables variant shares its program; web-server and
  application bouncers share the registration pattern).
- The AppSec component; PostgreSQL/MySQL database backends;
  multi-machine Local API deployments and TLS for the API;
  decisions import in bulk (its formats are quoted from the
  tool's own help); the duration-expression profile (quoted
  from the shipped profiles.yaml, not enabled).
- The neighbours of chapter 13 — characterised from their own
  documentation and the series' guides, not run side by side.

Fixtures are this guide's own and deliberately generic: a
thirteen-line auth.log in syslog format using only the
documentation address ranges (203.0.113.0/24, 198.51.100.0/24,
192.0.2.0/24), the invented host `labhost`, and stock user
names (admin, oracle, test, guest, marina). The lab machine's
password and the bouncer's API key were generated fresh for
the lab, never left the lab's scratch tree, are redacted in
every transcript, and died with it.
