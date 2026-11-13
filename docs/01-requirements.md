# Requirements and ambiguities

This checklist paraphrases the supplied version 4.0 subject. **Required** means
the PDF explicitly requests it. **Choice** means a proposed contract for your
implementation. Recheck against any newer subject or evaluation sheet you are
actually assigned; no evaluation sheet was supplied.

## Submission and build

| ID | Required behavior | Printed page | Verification |
| --- | --- | --- | --- |
| R01 | Implement in C; name the executable `ft_nmap` | 4, 5 | Build and run `./ft_nmap --help` |
| R02 | Submit a Makefile with usual rules; rebuild/relink only when needed | 4 | Test `all`, `clean`, `fclean`, `re`; repeat `make` without edits |
| R03 | Handle errors without crashes, bus errors, or double frees | 4 | Invalid inputs and failure/cleanup tests |
| R04 | Mandatory dependencies are libc, pcap, and pthread | 3, 4 | Review includes, build flags, and linked libraries |
| R05 | Implement the scanner yourself; do not call the real Nmap from the program | 6 | Inspect source and runtime subprocess behavior |
| R06 | Submit all work in the Git repository; humans perform evaluation | 4, 9 | Review repository contents and explain implementation |
| R07 | Demonstrate in a VM with Linux kernel greater than 3.14 | 9 | `uname -r` inside the VM |

The subject says grading was designed for Debian 7.0 stable. This is a historical
environment note, alongside the separate kernel requirement, not a reason to
expose an obsolete installation to the Internet. Prepare an isolated Linux VM,
and verify the evaluator's actual environment and API availability.

## Inputs and execution

| ID | Required behavior | Printed page | Verification |
| --- | --- | --- | --- |
| R08 | Provide help | 5, 6 | Help works without network privileges |
| R09 | Support individual IPv4 addresses and hostnames/FQDNs | 5 | Numeric address, local hostname, fully qualified hostname |
| R10 | Read IPv4 addresses and hostnames from a file; file format is free | 5, 6 | Multiple valid targets and invalid file cases |
| R11 | Choose threads: default 0, maximum 250 | 5 | Test 0, 1, several workers, 250, invalid values |
| R12 | Support SYN, NULL, ACK, FIN, XMAS, UDP | 5 | Exercise each scan separately |
| R13 | Without a scan selection, run all six types | 5 | Default plan includes all six |
| R14 | Permit individual scan types and several types together | 5 | A multi-scan run retains separate results |
| R15 | Select ports individually or by ranges | 5 | Single, list, range, mixed selection |
| R16 | Without ports, scan 1 through 1024 | 5 | Inclusive default has exactly 1024 ports |
| R17 | At most 1024 ports may be scanned | 5 | Reject 1025 distinct selected ports |
| R18 | Resolve service types, without service version detection | 5 | Protocol-specific service names and unknown fallback |
| R19 | Present clear scan results and readable elapsed time | 6, 7 | Configuration, per-scan states, conclusions, duration |

The 1024 limit is the **number of selected ports**, not a maximum port number.
A selection such as `8080,8443` is therefore compatible with the subject.
For one target, the default plan has 1024 x 6 = 6144 scan jobs before retries.

## Decisions to document explicitly

| Ambiguity | Proposed choice | Reason |
| --- | --- | --- |
| Default thread count is 0 | Main thread executes scans serially; no worker threads | Gives zero a useful meaning without creating an empty worker pool |
| Does 250 include helper threads? | Keep **all created pthreads** at or below 250 | Conservative interpretation; architecture uses workers for both sending and capture |
| FQDN support but no obligation to implement DNS resolution | Use libc `getaddrinfo` with `AF_INET`; do not implement DNS packets | Hostnames still need system resolution; libc can use hosts files or DNS |
| What if a hostname has several IPv4 answers? | Choose one deterministically and display it | Single-target argument stays a single target; document the selection |
| What if both `--ip` and `--file` appear? | Reject; require exactly one source | The subject presents two usage forms |
| Is port zero accepted? | Accept 1..65535 and reject 0 | Conventional useful scanning range; zero policy is not specified |
| Repeated and overlapping ports | Deduplicate, then enforce 1024 distinct ports | Prevents duplicate work and ambiguous counts |
| Grammar for multiple scans | Comma-separated list, e.g. `SYN,ACK,UDP` | The subject explicitly leaves argument format free |
| Timeout/retry numbers | Start with 1000 ms and one retry per job in the local lab | Engineering starting point, not a subject requirement |
| Contradictory observations | Keep every observation and report inconsistent TCP evidence | Avoid inventing certainty from dissimilar probes |
| File contains a bad target | Validate/resolve before scanning and reject the run with line context | Avoid a silently partial scan |

System socket APIs are necessary for this project even though the PDF loosely
describes the allowed libraries as the standard C library plus pcap and pthread.
On Linux, the libc socket/resolver APIs fit that dependency boundary; avoid
adding libnet, an external DNS library, or a subprocess to do mandatory work.
If your campus supplies a narrower function list, check it before coding.

## Do not copy the examples' mistakes

The prose specifies `--ports`, while result examples use `--port`. Use `--ports`
as the documented option; an optional alias is a choice. The examples also count
70-90 as 20 ports and 75-85 as 10. Inclusive ranges have 21 and 11 ports.

Some illustrated states and final conclusions are inconsistent. Treat the
screenshots as layout examples, not a packet-classification specification.
Use [the behavior tables](03-scan-behavior.md) and preserve TCP/UDP separation.

## Bonuses: defer until mandatory work is complete

Printed page 8 suggests DNS/version management, OS detection, additional flags,
and more advanced network behavior. It explicitly excludes `-v`/`-V` as a valid
bonus and makes bonus assessment conditional on a fully working mandatory part.
The roadmap does not implement advanced evasion or source concealment features.
Service-name lookup and basic FQDN acceptance already belong to mandatory work;
do not count them again as bonuses.
