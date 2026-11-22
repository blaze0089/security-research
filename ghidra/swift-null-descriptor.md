# Ghidra Swift: a null descriptor kind

**Component:** `Ghidra/Features/Swift`
**Fix:** [NationalSecurityAgency/ghidra#10167](https://github.com/NationalSecurityAgency/ghidra/pull/10167)
**Impact:** Metadata parsing aborts on a malformed or newer Swift binary

## Background

`SwiftTypeMetadata.parseTypeDescriptors()` reads a type context descriptor and switches on `descriptor.getFlags().getKind()`:

```java
ContextDescriptorKind kind = descriptor.getFlags().getKind();
descriptor = switch (kind) {
    case Class -> new TargetClassDescriptor(reader);
    ...
};
```

`ContextDescriptorKind.valueOf()` returns `null` for values it does not recognize. The switch called `kind.ordinal()` before reaching the default branch, so an unknown kind threw `NullPointerException` and aborted the entire metadata parse.

## Fix

Check for `null`, log the descriptor index, and skip that entry:

```java
if (kind == null) {
    log("Unrecognized type descriptor kind at index: 0x%x".formatted(origIndex));
    continue;
}
```

This matches the existing behavior for known-but-unhandled kinds.

## Verification

Compiled with `gradlew :Swift:compileJava` on JDK 25. A malformed descriptor no longer aborts the parse; the remaining entries are processed normally.

## Notes

Swift metadata is versioned and evolves between compiler releases. Treating an unknown kind as a fatal error turns a forward-compatibility problem into a denial of service on otherwise valid binaries.
