# CLI, target files, and output

The grammar below is a **proposed interface** satisfying the subject's free-form
argument allowance. It is a contract to implement, not a claim that code exists.

## Usage and defaults

```text
ft_nmap --help
ft_nmap --ip TARGET [--ports PORTS] [--speedup N] [--scan SCANS]
ft_nmap --file PATH [--ports PORTS] [--speedup N] [--scan SCANS]

TARGET  one IPv4 address or hostname/FQDN
PORTS   comma-separated ports and inclusive ranges: 22,80,443,8000-8010
N       integer 0..250; default 0 (serial, no worker threads)
SCANS   comma-separated names: SYN,NULL,ACK,FIN,XMAS,UDP
```

Ports default to `1-1024`. Scans default to all six. Allow options in any order,
but reject repeated options, empty values, unknown flags, and stray positional
arguments. Accept scan names case-insensitively, normalize to uppercase, and
deduplicate the scan set. A standalone `--help` exits successfully without
opening sockets or pcap; mixed help/scan invocations can be rejected consistently.

Examples, to run only after implementing the scanner in your lab:

```sh
./ft_nmap --help
sudo ./ft_nmap --ip 192.168.56.20 --ports 8080,8081 --scan SYN
sudo ./ft_nmap --ip target.lab.test --ports 22,80,443 --scan SYN,ACK --speedup 8
sudo ./ft_nmap --file targets.txt --ports 8080-8083 --speedup 4
```

## Validate before touching the network

1. Parse flags into a configuration object with no socket activity.
2. Parse unsigned decimal values strictly: reject signs, empty strings, trailing
   characters, and overflow. With `strtoul`, inspect `errno`, `endptr`, and bounds.
   Reject a leading minus explicitly; `strtoul` itself accepts signs.
3. Expand inclusive ranges after verifying `start <= end`. Use an integer type
   wider than `uint16_t` for loops so incrementing 65535 cannot wrap forever.
4. Deduplicate ports with a 65536-entry bitmap and enforce the final count.
   Reject more than 1024 unique ports, including a range such as `1-65535`.
5. Validate exactly one target source and every nonempty target line.
6. Resolve targets, select routes/interfaces, then start capture and sending.

Parsing boundary cases:

| Input | Expected contract |
| --- | --- |
| `--ports 1-1024` | Accept: 1024 ports |
| `--ports 1-1025` | Reject: 1025 distinct ports |
| `--ports 22,22,20-22` | Accept: 20, 21, 22 |
| `--ports 65535` | Accept |
| `--ports 0`, `65536`, `-1`, `8-3`, `80x`, `22,,80` | Reject |
| `--speedup 0`, `1`, `250` | Accept |
| `--speedup -1`, `251`, `1x`, or an overflowing integer | Reject |
| `--scan SYN,UDP` | Accept: two distinct scan types |
| `--scan CONNECT` or empty selection | Reject |

## Target file format

Choose one target per line; trim leading/trailing spaces and CRLF endings.
Ignore empty lines and full-line `#` comments. Do not silently interpret inline
comments or multiple whitespace-separated targets unless you document support.

```text
# Local VM targets
192.168.56.20
target.lab.test
```

Reject an empty effective target list, unreadable file, overlong line, embedded
NUL, or invalid target with filename and line number. Choose and document a
maximum line length, e.g. 4096 bytes, to bound allocations. Treat the final line
without a newline as valid. Deduplicate resolved addresses if you want to avoid
scanning the same target twice; retain the original name for display.

Use `inet_pton(AF_INET, ...)` for numeric addresses and libc `getaddrinfo` with
`hints.ai_family = AF_INET` for names. Free results with `freeaddrinfo`; use
`gai_strerror` for resolver errors. An IPv6-only answer cannot satisfy this
IPv4-only scanner. The resolver may block independently of scan deadlines.
See [getaddrinfo(3)](https://man7.org/linux/man-pages/man3/getaddrinfo.3.html).

## Service names and output

Look up `(port, protocol)` after scanning, in a single reporting phase.
`getservbyport(htons(port), "tcp")` and the corresponding `"udp"` lookup return
local service database labels; copy returned names immediately because the
non-reentrant API uses shared storage. Use `unknown` when absent. This is a
conventional port label, not evidence of the application actually listening.
See [getservent(3)](https://man7.org/linux/man-pages/man3/getservent.3.html).

Show the input name, resolved IPv4, selected ports/count, scans, requested and
effective workers, and elapsed seconds measured with a monotonic clock. Sort
rows by target, port, and transport so concurrent completion order never changes
the report. Keep diagnostics on stderr and results on stdout.

Suggested illustrative layout (hypothetical observations):

```text
Target: target.lab.test (192.168.56.20)
Ports: 2    Scans: SYN,ACK,UDP    Workers: 8    Elapsed: 2.014 s
PORT/PROTO  SERVICE  PER-SCAN RESULTS                      CONCLUSION
8080/tcp    unknown  SYN=open ACK=unfiltered                open
8080/udp    unknown  UDP=open|filtered                      open|filtered
8081/tcp    unknown  SYN=closed ACK=unfiltered              closed
8081/udp    unknown  UDP=closed                             closed
```

A numeric port can have different TCP and UDP states. Display both rather than
letting one protocol's result overwrite the other. Errors such as capture
failure are operational failures, not port states. Choose stable exit codes,
e.g. 0 for a completed scan/help, 2 for invalid input, 1 for runtime failure.
