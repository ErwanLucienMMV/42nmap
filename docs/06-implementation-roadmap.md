# Implementation roadmap

Complete each gate before moving to the next. The deliverable of this repository
update is documentation; the C files and tests below still need to be written.

## M0: Prepare the Linux VM

Confirm `uname -r` is greater than 3.14, install a C compiler, make, and libpcap
development headers, and configure a host-only lab network. On a suitable Debian
or Ubuntu VM, the package names are commonly:

```sh
sudo apt-get update
sudo apt-get install build-essential libpcap-dev tcpdump iproute2 python3
```

Add Nmap only as an external comparison tool and a firewall tool for fixtures.
Check package availability on your actual distribution. Develop in the VM's
native filesystem where possible rather than relying on Windows build tools.

**Gate:** a tiny C program can link against pcap and pthread; the VM can reach
your lab target and identify the correct route/interface.

## M1: Build skeleton and CLI

Create `main`, a help screen, config cleanup, and a Makefile. Typical compiler
options are `-Wall -Wextra -Werror -pthread`; link with `-lpcap` and use `-pthread`
at link time too. `-MMD -MP` can generate header dependencies. Treat any language
standard selection as your project choice; use a compiler supported by the VM.

Use the usual `all`, `clean`, `fclean`, `re` targets. `clean` removes generated
objects/dependency files, `fclean` also removes `ft_nmap`, and `re` rebuilds.
Build the executable from object prerequisites; do not mark the executable phony.
With parallel make, ensure `re` runs cleaning before rebuilding, rather than
declaring two unordered prerequisites that can race.

**Gate:** strict argument tests pass, all defaults print correctly, and a second
`make` performs no compilation or linking. Touching a header rebuilds its users.

## M2: Target loading and scan planning

Implement target-file validation, IPv4/FQDN resolution, route/source selection,
port deduplication, and deterministic job construction. Resolve service names
only in the reporting phase. Keep this stage usable without raw-socket access.

**Gate:** for one target the default creates exactly 6144 jobs, while
`--ports 8080,8081 --scan SYN,ACK` creates four. Invalid input emits useful context
and exits without sending a packet.

## M3: Packet serialization and pure logic

Write checksum, IPv4, TCP, UDP, bounded parser, and classification functions.
Feed them fixed byte arrays and fabricated parsed events. Separate matching
from classification so a classifier cannot accept unrelated traffic.

**Gate:** checksums match independently decoded packets; all scan response and
silence cases pass table-driven tests; truncated and malformed data never read
out of bounds. No live traffic is needed for these tests.

## M4: One target, one port, serial SYN

Open capture before transmission. Verify one known-open and one known-closed
TCP port in the lab, then a deliberately dropped port. Inspect both outgoing
probes and incoming replies with tcpdump.

**Gate:** distinguish `open`, `closed`, `filtered`; capture setup errors are
runtime failures; an unexpected SYN+ACK is reset without completing a handshake.

## M5: Complete serial scanning

Add NULL, FIN, XMAS, ACK, then UDP using shared packet/capture machinery. Add
bounded retries, absolute deadlines, ICMP quoted-packet matching, port reservation,
and per-scan evidence. Then expand to port lists and target files.

**Gate:** every scan works alone and in combinations; no-response jobs terminate
as specified; late replies cannot cross-contaminate sequential jobs.

## M6: Add pthread workers

Reuse the serial job executor. Add the bounded worker pool, setup gate, queue
lock, explicit shutdown state, and partial-startup cleanup. Run small mixed
TCP/UDP plans before trying many workers.

**Gate:** equivalent lab results at speedup 0, 1, 8, and 250; no duplicate or
missing jobs, hangs, interleaved output, leaked handles, or thread-data races.
Timing may vary and UDP rate limiting may require pacing; more workers do not
guarantee better accuracy or faster completion.

## M7: Finish reporting and failure paths

Add sorted results, protocol-specific service labels, readable duration, and
conservative conclusions. Test missing permissions, route/capture errors, memory
allocation failure where injectable, interrupted runs, and partial thread setup.

**Gate:** a fresh clone builds in the VM, the mandatory checklist is satisfied,
and you can explain every transmitted field and observed state.

## M8: Prepare the defense, then consider bonuses

Record the actual compiler, libpcap version, kernel, commands, captures, and
expected/actual states. Repeat the mandatory checks after any bonus change.
Do not spend time on bonus features while any mandatory behavior is uncertain.

The [test guide](07-testing-and-debugging.md) gives reproducible lab cases and
the [defense notes](08-defense-and-references.md) list questions to rehearse.
