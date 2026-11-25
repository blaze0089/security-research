# lzfse: one byte past the FSE payload

**Component:** `src/lzfse_fse.h`
**Fix:** [lzfse/lzfse#66](https://github.com/lzfse/lzfse/pull/66), [NationalSecurityAgency/ghidra#10189](https://github.com/NationalSecurityAgency/ghidra/pull/10189)
**Impact:** One-byte out-of-bounds read at the end of the input

## Background

LZFSE's 64-bit FSE input stream refills in 8-byte chunks:

```c
fse_bit_count nbits = (63 - s->accum_nbits) & -8;
const uint8_t *buf = (*pbuf) - (nbits >> 3);
...
memcpy(&incoming, buf, 8);
```

When `accum_nbits` is already 63, `nbits` is zero and `buf` does not move. `fse_in_checked_init64()` positions the input pointer at `buf_end - 7`, so the unconditional 8-byte load reads one byte past the end of the input. The 32-bit implementation already skips the load when `nbits == 0`.

## Reproduction

A 147-byte `bvx2` stream with the LMD payload ending at the end of the input triggers an AddressSanitizer report. The reproducer is [reproducer.bvx2](reproducer.bvx2).

```
ERROR: AddressSanitizer: heap-buffer-overflow
READ of size 8
#0 fse_in_flush lzfse_fse.h
#1 lzfse_decode_lmd lzfse_decode_base.c:196
```

The overread byte is masked out when `nbits == 0`, so the practical impact is limited, but the read is still out of bounds.

## Fix

Return early when `nbits == 0`, matching the 32-bit implementation:

```c
fse_bit_count nbits = (63 - s->accum_nbits) & -8;
if (nbits == 0)
  return 0;
```

## Verification

- The upstream test suite builds and all 15 CTest round-trip tests pass.
- The fixed decoder produced byte-identical output for the existing sample files.
- A fuzzer built against the fixed header ran about 1.16 million iterations without another ASan report.

## Notes

Ghidra vendors this decoder and invokes it through its LZFSE CLI wrapper. The Ghidra-side fix was merged as [NationalSecurityAgency/ghidra#10189](https://github.com/NationalSecurityAgency/ghidra/pull/10189) on 2026-11-24.
