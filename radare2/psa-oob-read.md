# radare2: out-of-bounds reads in psa string detection

**Component:** `libr/core/cmd_anal.inc.c`
**Fix:** [radareorg/radare2#27376](https://github.com/radareorg/radare2/pull/27376)
**Impact:** Out-of-bounds read; the printed byte can come from adjacent heap memory

## Background

The `psa` command scans the current block for Pascal and UTF-16 strings. `check_string_at()` allocates exactly `core->blocksize` bytes for the block, then calls `strnullpad_check()` and reads the Pascal length. Several paths assumed the block was larger than it was.

Three cases:

1. `strnullpad_check()` walks the buffer in two-byte pairs. For odd block sizes (1, 3, 5, 7, 9), it read `buf[i + 1]` on the final byte.
2. `check_string_at()` read the 16-bit Pascal length with `r_read_le16(buf)` without checking `len >= 2`, so a one-byte block read past the allocation.
3. The Pascal2 check required only `plen < len`. A 255-byte payload in a 256-byte block passed, and `r_str_ndup()` scanned one byte past the buffer.

## Reproduction

```
printf '\x00' > sample.bin
r2 -N -qc 'b 1; psa' sample.bin
```

## Fix

- Stop the UTF-16 pair walk before the trailing byte.
- Reject zero-length blocks.
- Gate the 16-bit length read on `len >= 2`.
- Require `plen + 2 <= len` for Pascal2.

The unused `clen` argument was removed from `strnullpad_check()`.

## Tests

Added `psa null byte with odd block size` and `psa null byte with 3-byte block` to `test/db/cmd/cmd_print_strings`. The existing `pxj` UTF-8 case is unchanged. I also verified the old code faults and the new code stays in bounds with a guard-page harness that places the buffer against a `PAGE_NOACCESS` page, and ran the new tests under ASan.

## Notes

The Pascal2 path is the most interesting of the three: `plen < len` allowed a length that consumed the entire block, and `r_str_ndup()` then walked one byte past the end. The printed byte is not guaranteed to be attacker-controlled, but it is an out-of-bounds read and can expose adjacent heap data in output.
