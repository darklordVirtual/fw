# fw

Plain-text IP blocklists for firewall use.

This repository carries **data, not rules**. Nothing here is meant to be executed.

## Files

| File | Format | Contents |
|---|---|---|
| `blocklist/attackers-v4.txt` | one IPv4 address or CIDR per line | Addresses observed conducting unsolicited scanning, brute-force or similar unwanted traffic against monitored edges |

## Format

- One entry per line: `203.0.113.4` or `203.0.113.0/24`
- `#` starts a comment; blank lines are ignored
- IPv4 only in this file
- Entries are expected to be routable public addresses

Anything that does not match those rules is malformed and should be discarded by the consumer rather than interpreted.

## Consuming a list

Two endpoints serve the same file:

```
# CDN-cached, no rate limit
https://raw.githubusercontent.com/darklordvirtual/fw/main/blocklist/attackers-v4.txt

# uncached, 60 requests/hour per source address without authentication
https://api.github.com/repos/darklordvirtual/fw/contents/blocklist/attackers-v4.txt
   with header:  Accept: application/vnd.github.raw
```

**Prefer the API endpoint if removals need to take effect promptly.** Measured 2026-11-11: an *addition* reached the raw endpoint in about 10 seconds, while a *removal* was still being served from CDN cache more than four minutes after the commit. A cache-busting query string does not help — the CDN does not vary on query. Removal is the direction that carries urgency, because that is how a false positive gets withdrawn.

**Do not apply the file verbatim.** A consumer is expected to:

1. Keep only lines that parse as an IPv4 address or CIDR.
2. Drop private, loopback, link-local, CGNAT, multicast and reserved space — `10/8`, `172.16/12`, `192.168/16`, `127/8`, `169.254/16`, `100.64/10`, `224/4`, `240/4` — so a list can never affect internal traffic.
3. Drop prefixes shorter than `/8`.
4. Subtract a **locally held** allow list of addresses that must never be blocked. That list belongs on the consumer, not in this repository.
5. Refuse the update if the result is implausibly small compared with the set already in use.
6. Swap the result in atomically, and keep the previous set on any failure.

Steps 2, 4 and 5 are what make it safe to consume an unsigned list over the public internet. With them, the worst outcome of a stale, broken or tampered list is over-blocking of external addresses — never loss of internal connectivity, and never lockout.

Bind the resulting rule to the external interface only. A blocklist applied to all interfaces is a self-inflicted outage waiting for its first bad entry.

## Scope and limits

- Entries are **observations, not adjudications**. An address can be shared hosting, a rotating allocation, a reassigned block or a transient compromise. Treat removal requests seriously; open an issue.
- No freshness or availability guarantee. A consumer must tolerate an unreachable or unchanged source and keep running on its current set.
- The raw endpoint is served through a CDN and can lag a commit by a few minutes. Poll on an interval; do not expect immediate propagation.
- Blocking an address inbound is low-risk. Blocking it outbound as well can break a legitimate service that happens to share the address, and should be a separate, deliberate decision.

## Changing a list

Changes go through pull request against `main`. Keep the files sorted and free of duplicates.

Say what was observed, and when, in the **commit message** — not in the file. The files stay plain so that any consumer can parse them without special cases.
