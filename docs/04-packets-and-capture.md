# Packet construction, capture, and correlation

Implement and verify this in the Linux VM. The design uses raw IPv4 sockets for
transmission and libpcap for reception; it avoids implementing Ethernet/ARP
transmission by hand. This is a design choice, not a required architecture.

## 1. Select the route and source address

Do not pick the first interface returned by enumeration. Loopback, host-only VM
networks, and default-route traffic can use different interfaces and local IPs.
For each resolved destination, a temporary UDP socket `connect` followed by
`getsockname` can reveal the selected local IPv4 without doing a TCP handshake.
No UDP datagram needs to be sent. Match that address against local interfaces,
then capture on the correct interface. Report route-selection failures clearly.
Process targets one at a time initially to keep this lifecycle manageable.

Create `socket(AF_INET, SOCK_RAW, IPPROTO_RAW)`, send a complete IPv4 datagram,
and explicitly configure `IP_HDRINCL` for clarity. Linux requires suitable
privileges, normally root or `CAP_NET_RAW`. This socket is for sending; receive
through pcap. Linux fills certain IPv4 fields with `IP_HDRINCL`, including total
length and header checksum, but your serializer should still construct correct
bytes and compute transport checksums itself.
See [raw(7)](https://man7.org/linux/man-pages/man7/raw.7.html).

## 2. Serialize bytes deliberately

| Layer | Initial layout | Important fields |
| --- | --- | --- |
| IPv4 | 20 bytes, no options | version=4, IHL=5, total length, identification, TTL, protocol, addresses, checksum |
| TCP | 20 bytes, no options/payload | source/destination ports, sequence, acknowledgment, data offset=5, flags, window, checksum |
| UDP | 8 bytes plus optional payload | source/destination ports, length, checksum |

For example, an empty TCP probe is 40 bytes at IPv4 level; an empty UDP probe is
28. There is no Ethernet header in these raw-socket send buffers. IPv4 protocol
numbers are TCP=6 and UDP=17. Keep packet lengths in host order while calculating
them, then encode multi-byte fields exactly once in network order.

Avoid sending arbitrary compiler struct layouts: C bitfields, padding, and
alignment can differ. Byte arrays with explicit offsets and `memcpy`/byte stores
are straightforward to audit. If using system header structs, verify sizes and
encoding and never assume captured packet memory is aligned for a struct cast.

Packet format references: [IPv4, RFC 791](https://www.rfc-editor.org/rfc/rfc791.html),
[TCP, RFC 9293 section 3.1](https://www.rfc-editor.org/rfc/rfc9293.html#section-3.1),
[UDP, RFC 768](https://www.rfc-editor.org/rfc/rfc768.html).

## 3. Compute checksums

The Internet checksum sums 16-bit words, folds carries back into 16 bits, and
complements the result. Zero the checksum field first. For an odd-length input,
pad the final byte with a zero for calculation only; do not transmit a padding
byte unless it belongs to the actual packet.

IPv4 checksums cover the IPv4 header. TCP/UDP checksums cover a pseudo-header
(source/destination IPv4, zero byte, protocol, transport length) followed by the
transport header and payload. A computed UDP checksum of zero is encoded as
`0xffff`; IPv4 UDP's wire value zero instead means no checksum supplied.
See [RFC 1071](https://www.rfc-editor.org/rfc/rfc1071.html) and
[RFC 768](https://www.rfc-editor.org/rfc/rfc768.html).

Create fixed-byte checksum test vectors and verify independently with a capture
decoder. NIC checksum offload can make outgoing captures misleading; compare
with a capture on the receiving VM before diagnosing a serialization bug.

## 4. Open capture before sending

Suggested sequence per capture handle:

```text
pcap_create(interface)
set snapshot length, non-promiscuous mode, short buffering timeout
pcap_activate(handle)               # negative=failure, positive=warning
pcap_datalink(handle)               # choose supported link decoder
pcap_compile(filter), pcap_setfilter(filter), pcap_freecode(filter)
pcap_setnonblock(handle, 1, errbuf)
publish capture-ready, then permit sending
```

Choose a snap length such as 65535 for the initial lab. Check every API result;
close inactive handles too when activation fails. Start capturing first so a
fast SYN+ACK cannot race setup. A pcap buffering timeout is not a scan deadline.
See the [libpcap overview](https://raw.githubusercontent.com/the-tcpdump-group/libpcap/master/pcap.3pcap.in).

A conceptual BPF filter is:

```text
dst host LOCAL_IP and ((src host TARGET_IP and (tcp or udp)) or icmp)
```

Add a destination source-port range restriction to the TCP/UDP branch if useful.
Do not require ICMP's outer source to equal the target: an intervening router
may send the error. Keep software validation even with a kernel BPF filter.

Use the link type returned by `pcap_datalink`, not an unconditional 14-byte skip.
For Ethernet, account for supported VLAN tags before IPv4. Linux `any` capture
can produce cooked SLL/SLL2, and loopback behavior varies. Implement the link
types your VM uses and reject unsupported ones explicitly; do not misparse them.
Prefer the route's concrete interface over `any` in the initial implementation.

## 5. Bounds-check every received layer

Use the captured length (`caplen`), never just the original wire length (`len`).
Before reading any field, prove its bytes are present. Validate IPv4 version,
IHL >= 5, header length, total length, and transport minimum sizes. For direct
TCP/UDP replies, require the bytes being decoded to fit both caplen and the
declared IPv4/transport length. Validate TCP data offset and UDP length.

Fragmented packets need reassembly to interpret safely. For the small local-lab
responses, choose to reject fragmented input with a diagnostic rather than
treating a fragment as a full transport header. This is a documented limitation;
never read ports from non-initial fragments.

## 6. Match direct and quoted replies

Direct TCP or UDP replies must reverse the active probe's tuple:

```text
response.src_ip   == probe.target_ip
response.dst_ip   == probe.local_ip
response.src_port == probe.destination_port
response.dst_port == probe.source_port
response.protocol == probe.transport
```

For SYN+ACK, validate the acknowledgment against `probe.sequence + 1` (modulo
2^32). When an RST carries ACK, validate against the transmitted sequence and
its SYN/FIN sequence-space consumption; do not require ACK on every RST.
An ACK probe's reset can instead use your acknowledgment token as its sequence.
Design these checks from the reset rules, not one blanket ACK formula.
See [RFC 9293 section 3.5.2](https://www.rfc-editor.org/rfc/rfc9293.html#section-3.5.2).

For ICMP type 3, decode the quoted **original** IPv4 header and at least its
first eight transport bytes. Validate inner IHL, addresses, protocol, and ports;
for TCP those bytes also expose the original sequence token. The quotation
need not include the full original packet: its inner total length normally
exceeds the quoted bytes. Require enough bytes for the matching fields, not the
entire original datagram. See [RFC 792](https://www.rfc-editor.org/rfc/rfc792.html).

Do not classify a port using an unrelated router error, an outgoing probe, or
an earlier job's late reply. Reserve distinct source ports for different jobs
to the same target, retaining reservations until that target's scan ends. With
1024 ports and six scans, at most 6144 reservations are needed per target. A
chosen unbound local high-port interval must have sufficient available slots;
check/hold local reservations or fail cleanly if exhausted. Sequence tokens
provide an additional check, not a substitute for tuple validation.

## 7. Make capture deadlines reliable

For nonblocking capture, drain bounded packet batches, check the monotonic job
deadline, and wait only up to the smaller of a short poll interval and the
remaining time. A selectable pcap FD plus `poll` can help on Linux; still try a
nonblocking drain after a poll timeout to accommodate buffering behavior.
Do not spin at 100% CPU, and do not let unrelated traffic postpone a deadline.

If using `pcap_next_ex`, distinguish packet=1, live timeout=0, error=-1, and
offline end/break=-2. Packet/header pointers are owned by libpcap and cannot be
retained across further reads without copying. See
[pcap_next_ex](https://raw.githubusercontent.com/the-tcpdump-group/libpcap/master/pcap_next_ex.3pcap).

If using `pcap_loop` in a different architecture, note that `pcap_breakloop`
is not a universal wakeup mechanism and its effects depend on capture backend
and API. A worker-owned nonblocking handle avoids needing another thread to
close a handle during a read. See
[pcap_breakloop](https://raw.githubusercontent.com/the-tcpdump-group/libpcap/master/pcap_breakloop.3pcap).
