# Security Research

Source-level audits of binary-analysis and reverse-engineering tools, with a focus on malformed-input handling in parsers and protocol code.

Each entry below has a minimal reproducer, a root-cause writeup, and a link to the upstream fix.

## Merged fixes

| Project | Issue | Impact | Fix |
| --- | --- | --- | --- |
| Ghidra | LZFSE FSE input flush performs an 8-byte load after the last input byte | One byte past the input is read; crashes under ASan, no data disclosure | [#10189](https://github.com/NationalSecurityAgency/ghidra/pull/10189) |
| Ghidra | ISF connection handler busy-loops after a client disconnect | A connected client that closes the socket leaves a handler thread consuming a CPU core | [#10185](https://github.com/NationalSecurityAgency/ghidra/pull/10185) |
| Ghidra | Version Tracking comment encoder does not escape backslashes | Windows paths are corrupted when comments are saved and restored; a trailing backslash throws | [#10191](https://github.com/NationalSecurityAgency/ghidra/pull/10191) |
| Ghidra | Swift context descriptor kind can be null | A malformed or newer Swift binary throws a `NullPointerException` and aborts metadata parsing | [#10167](https://github.com/NationalSecurityAgency/ghidra/pull/10167) |
| radare2 | Out-of-bounds reads in `psa` string detection | One-byte and odd-sized blocks read past the allocated buffer | [#27376](https://github.com/radareorg/radare2/pull/27376) |

## Upstream fixes under review

| Project | Issue | Status |
| --- | --- | --- |
| lzfse | One-byte out-of-bounds read in the 64-bit FSE input flush | [PR #66](https://github.com/lzfse/lzfse/pull/66) |

## Writeups

- [Ghidra ISF: a disconnect that pins a CPU core](ghidra/isf-eof-busy-loop.md)
- [Ghidra Version Tracking: the backslash that disappeared](ghidra/version-tracking-backslash.md)
- [Ghidra Swift: a null descriptor kind](ghidra/swift-null-descriptor.md)
- [radare2: out-of-bounds reads in psa string detection](radare2/psa-oob-read.md)
- [lzfse: one byte past the FSE payload](lzfse/fse-flush64-oob.md)

## Method

- Source review and differential testing
- AddressSanitizer builds and custom in-process fuzzers
- Minimal reproducers and regression tests
- Coordinated disclosure before publishing security-impacting details

## Disclosure

Security-impacting issues are reported privately to the maintainers first. Technical details are published after a fix or advisory is available.

## Contact

GitHub: [@blaze0089](https://github.com/blaze0089)
