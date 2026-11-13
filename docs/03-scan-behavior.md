# Probe behavior and result classification

These tables describe an implementation policy grounded in the official Nmap
documentation. They supplement the subject, which lists scan types but does not
define every reply. Final classification requires a correctly correlated reply
or expiration after the configured attempts; a capture failure is not silence.

## Packet flags

| Scan | Transport | TCP flags | Hex |
| --- | --- | --- | --- |
| SYN | TCP | SYN | `0x02` |
| NULL | TCP | none | `0x00` |
| ACK | TCP | ACK | `0x10` |
| FIN | TCP | FIN | `0x01` |
| XMAS | TCP | FIN + PSH + URG | `0x29` |
| UDP | UDP | not applicable | not applicable |

Use a nonzero sequence token for TCP correlation. Give an ACK probe a deliberate
nonzero acknowledgment token as well. Packet construction belongs in one module;
scan type should select flags and classification, not duplicate serializers.

## TCP replies

| Scan | Matching SYN+ACK | Matching RST | No reply after retries |
| --- | --- | --- | --- |
| SYN | `open` | `closed` | `filtered` |
| NULL / FIN / XMAS | Unexpected: keep waiting/log evidence | `closed` | `open|filtered` |
| ACK | Unexpected: keep waiting/log evidence | `unfiltered` | `filtered` |

A matched ICMP destination-unreachable message (type 3) normally means
`filtered`: use codes 0,1,2,3,9,10,13 for SYN and 1,2,3,9,10,13 for
NULL/FIN/XMAS/ACK as listed by the respective Nmap tables. Other errors can be
recorded diagnostically without forcing a state. Nmap's general reference and
individual book tables differ slightly on code 0; specify your tested policy.

ACK measures reachability through a filter, not whether a listener exists.
NULL/FIN/XMAS cannot prove openness from silence; different TCP stacks may reset
these probes even when a service listens. Reset a SYN half-open exchange after
observing SYN+ACK rather than completing a connection; the local kernel may
already send RST, so inspect the actual capture.

Sources: [SYN scan](https://nmap.org/book/synscan.html),
[NULL/FIN/XMAS](https://nmap.org/book/scan-methods-null-fin-xmas-scan.html),
[ACK scan](https://nmap.org/book/scan-methods-ack-scan.html), and the
[general technique reference](https://nmap.org/book/man-port-scanning-techniques.html).

## UDP replies

| Matching observation | State |
| --- | --- |
| UDP response from the probed service endpoint | `open` |
| ICMP type 3, code 3 (port unreachable) quoting this probe | `closed` |
| ICMP type 3, code 0,1,2,9,10,13 quoting this probe | `filtered` |
| No response after retries | `open|filtered` |

The general Nmap reference includes code 0 while its dedicated UDP table omits
it; this guide chooses to treat a matched network-unreachable as filtered.
Sending an empty UDP payload is sufficient as an initial probe. A listening
application can ignore it, so lack of a reply cannot prove that UDP is open.
ICMP responses can be rate-limited, making even closed UDP ports appear
ambiguous at high send rates. Pace probes and make attempts bounded.

Sources: [UDP scan](https://nmap.org/book/scan-methods-udp-scan.html) and
[general scan reference](https://nmap.org/book/man-port-scanning-techniques.html).

## Preserve evidence before producing conclusions

Store state per `(target IPv4, destination port, transport, scan type)` with
attempt count, response kind, and timing. Use `pending` internally; it is not a
final port state. Keep an explicit operational error channel.

The subject requests readable conclusions but gives no authoritative merge
algorithm. Proposed conservative reporting rules:

1. Preserve every per-scan result, including unexpected/contradictory evidence.
2. Compute TCP and UDP conclusions independently.
3. For TCP, SYN `open` or `closed` is direct listener evidence. If NULL/FIN/XMAS
   says `closed` while SYN says `open`, annotate a contradiction rather than
   silently erasing either observation.
4. Without SYN evidence, matched RSTs for NULL/FIN/XMAS support `closed`.
   ACK `unfiltered` alone never supports `open` or `closed`.
5. If the remaining observations cannot establish one state, retain their
   ambiguity, e.g. `open|filtered; ACK unfiltered`, rather than inventing a
   stronger conclusion.
6. UDP conclusion is its own UDP observation, independent of TCP.

Firewall rules can treat each flag combination differently. Do not assume that
intersecting sets of possible states from different scans always proves a state.

## Deadline policy

A useful local-lab starting choice is an initial transmission plus one retry,
1000 ms per attempt. Neither number is prescribed by the subject. Measure with
`CLOCK_MONOTONIC`; wall-clock changes must not extend or shorten jobs. Store
absolute deadlines, check them even when unrelated packets arrive continuously,
and finalize the silence state only after the last deadline.

With serial execution and these settings, 6144 silent default jobs may take
about 12,288 seconds (roughly 3 h 25 min). Parallelism reduces waiting but does
not remove ICMP rate limits, packet loss, or the need to bound work.
