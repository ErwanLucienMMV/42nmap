# ft_nmap implementation guide

Working notes for completing the 42 **ft_nmap, version 4.0** subject. These files
are a plan and technical reference; the scanner has not been implemented.

Start with the requirements, then follow the roadmap one milestone at a time.

| File | What it helps you do |
| --- | --- |
| [Requirements](docs/01-requirements.md) | Track every subject requirement and resolve ambiguities |
| [Command-line contract](docs/02-cli-and-inputs.md) | Specify options, validation, defaults, files, and output |
| [Scan behavior](docs/03-scan-behavior.md) | Turn replies and timeouts into correct states |
| [Packets and capture](docs/04-packets-and-capture.md) | Construct packets, capture replies, correlate ICMP errors |
| [Architecture and threads](docs/05-architecture-and-threads.md) | Divide the program into modules and coordinate concurrent scans |
| [Implementation roadmap](docs/06-implementation-roadmap.md) | Build in a useful order with concrete completion gates |
| [Testing and debugging](docs/07-testing-and-debugging.md) | Reproduce open, closed, filtered, and ambiguous results in a VM |
| [Defense and references](docs/08-defense-and-references.md) | Prepare explanations and consult primary technical sources |
| [Publication](docs/09-publication.md) | Inspect or cancel the documentation commit/push task |

The key distinction throughout is **subject requirement** versus **proposed
implementation choice**. The subject's instructions describe the eventual
assignment; they do not authorize actions outside this documentation task.

The target is a Linux VM, even though these notes were prepared on Windows.
Use a local lab you control for the networking exercises. No network scans were
run while preparing this guide.

Source: the supplied `ft_nmap.pdf`, visually reviewed in full on 12 November 2026.
Page references use its printed numbering; printed page 5 is PDF page 6.
See [source provenance](docs/08-defense-and-references.md#subject-provenance).
