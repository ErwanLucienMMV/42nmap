# Defense preparation and source notes

## Questions you should be able to answer

1. Why is this a packet scanner rather than a `connect()` loop? Which packet
   fields differ across the five required TCP scans?
2. Why does SYN+ACK mean open for SYN, while RST means unfiltered for ACK?
3. Why does silence mean filtered for SYN but open|filtered for UDP and FIN?
4. How do you compute TCP/UDP checksums, and why include the pseudo-header?
5. How do you select the route's source IP and capture interface?
6. How do you handle Ethernet, VLAN, loopback/cooked capture, and short packets?
7. Why can an ICMP error's outer source differ from the scanned target?
8. How does an eight-byte quoted transport header identify the original probe?
9. How do you distinguish a previous probe's delayed response from this job?
10. What exactly does `--speedup 0` do? How many pthreads are created at 250?
11. Which thread owns each handle, job, result, and resource? What happens if
    the seventeenth worker fails to initialize?
12. Why is a pcap buffering timeout different from a monotonic scan deadline?
13. Why can 250 workers reduce accuracy or increase memory use?
14. Why is a service name only a conventional database label?
15. How do you avoid merging TCP and UDP state for the same numeric port?
16. How do you show contradictory evidence without hiding it?
17. Which subject details are explicit, and which policies did you choose?
18. What proves your Makefile does not relink unnecessarily?

Bring a short reproducible demonstration: help, invalid input, a serial SYN scan,
each other scan alone, a mixed threaded scan, a target file containing a hostname,
and a capture explaining one UDP or ICMP result. Keep the lab services available.

## Subject provenance

Primary assignment source: user-supplied `ft_nmap.pdf`, **version 4.0** on its
cover, reviewed visually across all ten PDF pages on 12 November 2026.

```text
Source location at preparation: C:\Users\erwan\Downloads\ft_nmap.pdf
SHA-256: 088E07C70850A1B4A0E117E4B73481C42E6FA5116356A1575C1AA445CEFA5FB6
```

| Physical PDF page | Printed page | Content |
| --- | --- | --- |
| 1 | cover | Project name and version |
| 2 | 1 | Contents |
| 3 | 2 | Introduction |
| 4 | 3 | Objectives: pcap and threads |
| 5 | 4 | General instructions |
| 6 | 5 | Mandatory interface and features |
| 7 | 6 | Formatting, no Nmap delegation, help/results examples |
| 8 | 7 | Additional result example |
| 9 | 8 | Bonuses and assessment condition |
| 10 | 9 | Repository submission and Linux VM |

The PDF is image-based: plain text extraction produced no readable content.
The notes were derived from rendered pages, not inferred from a different
online subject. The original PDF is not duplicated in the repository.

These documents paraphrase assignment requirements and propose an implementation;
they do not replace the subject or assert that any scanner tests have passed.

## Primary technical references

Technical sources were checked on 12 November 2026. Use the installed manual
pages too, especially if the evaluation VM has older libpcap APIs. The linked
libpcap upstream `master` manuals can change after this date.

| Topic | Primary source |
| --- | --- |
| All Nmap options | [Nmap manual](https://nmap.org/book/man.html) |
| SYN packet states | [SYN scan](https://nmap.org/book/synscan.html) |
| NULL, FIN, XMAS | [Scan behavior and limitations](https://nmap.org/book/scan-methods-null-fin-xmas-scan.html) |
| ACK | [ACK response classification](https://nmap.org/book/scan-methods-ack-scan.html) |
| UDP | [UDP response classification](https://nmap.org/book/scan-methods-udp-scan.html) |
| General scan reference | [Techniques, including ICMP code lists](https://nmap.org/book/man-port-scanning-techniques.html) |
| IPv4 format | [RFC 791](https://www.rfc-editor.org/rfc/rfc791.html) |
| TCP format/reset rules | [RFC 9293](https://www.rfc-editor.org/rfc/rfc9293.html) |
| UDP format/checksum zero | [RFC 768](https://www.rfc-editor.org/rfc/rfc768.html) |
| ICMP quotation | [RFC 792](https://www.rfc-editor.org/rfc/rfc792.html) |
| Checksum algorithm | [RFC 1071](https://www.rfc-editor.org/rfc/rfc1071.html) |
| Linux raw sockets | [raw(7)](https://man7.org/linux/man-pages/man7/raw.7.html) |
| Capture lifecycle | [libpcap overview in its source repository](https://raw.githubusercontent.com/the-tcpdump-group/libpcap/master/pcap.3pcap.in) |
| Reading packets | [pcap_next_ex](https://raw.githubusercontent.com/the-tcpdump-group/libpcap/master/pcap_next_ex.3pcap) |
| Stopping capture | [pcap_breakloop](https://raw.githubusercontent.com/the-tcpdump-group/libpcap/master/pcap_breakloop.3pcap) |
| Hostname resolution | [getaddrinfo(3)](https://man7.org/linux/man-pages/man3/getaddrinfo.3.html) |
| Service database | [getservent(3), including getservbyport](https://man7.org/linux/man-pages/man3/getservent.3.html) |

The assignment controls required scope. Protocol and API references explain
implementation behavior. Where they disagree with illustrative subject output,
retain the subject's explicit feature requirement and explain your evidence-based
classification. No private evaluator rubric or hidden requirement was consulted.
