# Testing, lab setup, and debugging

The commands below are a **future test procedure**, not tests already run on an
implemented scanner. Use two isolated Linux VMs when possible: scanner
`192.168.56.10`, target `192.168.56.20`. Substitute your own host-only addresses
and interface names everywhere. Keep each capture together with its command,
environment, expected result, actual result, and timestamp.

## Establish known target conditions

On the target VM, verify addresses and listeners:

```sh
ip -4 address
ss -lntu
```

Run a TCP listener on 8080 in one terminal:

```sh
python3 -m http.server 8080 --bind 192.168.56.20
```

Run an empty-datagram-friendly UDP responder on 8081 in another:

```sh
python3 - <<'PY'
import socket
with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as sock:
    sock.bind(('192.168.56.20', 8081))
    while True:
        payload, peer = sock.recvfrom(65535)
        sock.sendto(b'lab-ok', peer)
PY
```

Ensure neither protocol has a listener on 8082. An open UDP application that
ignores an empty payload is a separate useful fixture: change the responder to
receive without replying, then expect `open|filtered` instead of `open`.

For packet-drop cases, add a dedicated nftables table on the **target lab VM**:

```sh
sudo nft add table inet ft_nmap_lab
sudo nft add chain inet ft_nmap_lab input '{ type filter hook input priority -10; policy accept; }'
sudo nft add rule inet ft_nmap_lab input ip saddr 192.168.56.10 ip daddr 192.168.56.20 tcp dport 8083 drop
sudo nft add rule inet ft_nmap_lab input ip saddr 192.168.56.10 ip daddr 192.168.56.20 udp dport 8083 drop
sudo nft add rule inet ft_nmap_lab input ip saddr 192.168.56.10 ip daddr 192.168.56.20 udp dport 8084 reject with icmpx type admin-prohibited
sudo nft list table inet ft_nmap_lab
```

Use a new table name if that name already exists. Other firewall chains can
still drop packets accepted by this chain, so inspect existing rules and verify
the intended behavior in captures. When done, remove only this lab table:

```sh
sudo nft delete table inet ft_nmap_lab
```

These commands alter only the example lab's firewall when you choose to run
them. They are not needed to read or use the documentation.

## Expected results

Assuming a typical Linux target stack, correct routes, no unrelated firewall
rules, and successful delivery:

| Target condition | SYN | NULL/FIN/XMAS | ACK | UDP |
| --- | --- | --- | --- | --- |
| TCP listener, 8080/tcp | open | open\|filtered | unfiltered | Separate transport; do not infer |
| UDP echo, 8081/udp | Separate transport | Separate transport | Separate transport | open |
| No listener, 8082 | closed | closed | unfiltered | closed, if ICMP reply arrives |
| Silent DROP, 8083 | filtered | open\|filtered | filtered | open\|filtered |
| UDP admin-prohibited REJECT, 8084 | Separate transport | Separate transport | Separate transport | filtered |

Each expectation is conditional on the actual packet evidence. High-rate UDP
tests can lose/rate-limit ICMP, and different TCP stacks can respond differently.
The classifier's packet fixtures should test deterministic logic independently
of these live-network uncertainties.

## Observe the wire before judging the result

On the scanner VM, replace `eth1` with its actual lab interface:

```sh
ip route get 192.168.56.20
sudo tcpdump -ni eth1 -nn -vvv 'host 192.168.56.20 and (tcp or udp or icmp)'
sudo tcpdump -ni eth1 -s 0 -w lab.pcap 'host 192.168.56.20 and (tcp or udp or icmp)'
```

For routed ICMP from a gateway, capture `icmp` more broadly; the outer error may
not contain the target address. Check outgoing flags, source/destination ports,
sequence/ACK values, checksums at the receiver, and ICMP's quoted inner packet.

After implementation:

```sh
sudo ./ft_nmap --ip 192.168.56.20 --ports 8080-8084 --scan SYN --speedup 0
sudo ./ft_nmap --ip 192.168.56.20 --ports 8080-8084 --scan NULL,FIN,XMAS,ACK --speedup 4
sudo ./ft_nmap --ip 192.168.56.20 --ports 8080-8084 --scan UDP --speedup 1
sudo ./ft_nmap --ip 192.168.56.20 --ports 8080-8084 --speedup 8
```

Use real Nmap **manually as an external oracle**, never from `ft_nmap` itself.
Compare one TCP technique per invocation: Nmap does not accept all the required
TCP scan modes together in the same way your project must.

```sh
sudo nmap -n -Pn -sS --reason -p 8080-8084 192.168.56.20
sudo nmap -n -Pn -sN --reason -p 8080-8084 192.168.56.20
sudo nmap -n -Pn -sA --reason -p 8080-8084 192.168.56.20
sudo nmap -n -Pn -sF --reason -p 8080-8084 192.168.56.20
sudo nmap -n -Pn -sX --reason -p 8080-8084 192.168.56.20
sudo nmap -n -Pn -sU --reason -p 8080-8084 192.168.56.20
```

`-n` avoids reverse DNS and `-Pn` avoids host-discovery differences. Nmap may
use protocol payloads, adaptive timing, and retries that differ from your
program, so inspect packet causes rather than assuming every text difference
is a bug. Source for options and modes:
[Nmap reference](https://nmap.org/book/man.html).

## Automated checks worth writing

| Area | Meaningful cases |
| --- | --- |
| CLI | Missing values, unknown flags, empty lists, numeric overflow, negative speedup, repeated options |
| Ports | Inclusive ends, 65535 loop termination, overlap deduplication, exactly 1024 vs 1025 |
| Files | CRLF, final line without newline, comments, empty file, unreadable file, bad hostname with line context |
| Planning | Correct job count, default six types, one target from multi-answer hostname |
| Serialization | Fixed TCP/UDP byte layouts, odd-length checksum data, zero-result UDP checksum |
| Parsing | Short link/IP/TCP/UDP/ICMP headers, invalid IHL/lengths, VLAN, unsupported link type, fragments |
| Matching | Wrong IP/protocol/ports/token, unrelated ICMP, short ICMP quote, late replies, outgoing packets |
| Classification | Every table entry, timeouts after retries, capture error distinct from silence |
| Workers | Every job exactly once, unique active tuples, partial setup failure, stop during wait |
| Reporting | Sorted stable rows, unknown service, TCP/UDP distinction, conflicting scan evidence |

Mock time and send/capture events for timeout tests instead of waiting real
seconds. Make ICMP quote fixtures include only the original header plus eight
bytes to verify that truncated-by-design quotations still match correctly.

## Integration and failure checklist

- Run the same small plan at speedup 0, 1, 8, 250; compare evidence and job counts.
- Check `--help` and input failures without root; raw-socket failure should name
  the failed operation and permission requirement.
- Test a hostname/FQDN through a lab hosts-file entry, multiple target lines,
  duplicate targets, and an IPv6-only/unresolvable hostname.
- Test bad routes, unsupported capture link types, send failures, and allocation
  or thread-start failures using injectable wrappers where practical.
- Interrupt both serial and threaded scans. All workers must stop and join;
  handles must close once. No cleanup is performed inside the signal handler.
- Run with no matching traffic and with continuous unrelated traffic. Both must
  honor deadlines, without high CPU spin or indefinite blocking.
- Check 250-worker capture-buffer memory use and file descriptor limits.
- Test repeated runs for leaked descriptors and lingering threads.
- Verify that a second `make` is a no-op and changing a header rebuilds its users.

Use AddressSanitizer/UndefinedBehaviorSanitizer for memory/encoding checks and
ThreadSanitizer separately for worker races when supported by your toolchain.
Clean between builds with different instrumentation. Packet parsing and pure
logic tests can run unprivileged; privileged integration testing needs the same
lab permissions as a normal raw-packet run. Do not combine ASan and TSan in one
binary. Record which checks you actually ran rather than marking the plan done.

## Symptom to investigate

| Symptom | First checks |
| --- | --- |
| Every TCP port looks filtered | Route/source IP, interface, capture ready before send, TCP checksum, BPF |
| UDP never shows closed | ICMP included, quoted tuple decoded, target rate limiting, firewall |
| Works serially, fails with workers | Shared pcap misuse, reused source ports, result ownership, locks |
| Fast replies are missing | Capture activation/filter installed too late, wrong local address |
| Program waits forever | Blocking capture, deadline reset on unrelated packets, startup gate failure |
| Wrong service name | `htons(port)`, correct protocol, copied static result, local services database |
| Scanner prints closed for ACK RST | Classifier incorrectly reuses SYN/FIN rules |
| NULL scan claims open from silence | Ambiguous state was incorrectly collapsed |
