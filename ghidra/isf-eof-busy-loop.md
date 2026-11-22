# Ghidra ISF: a disconnect that pins a CPU core

**Component:** `Ghidra/Debug/Debugger-isf`
**Fix:** [NationalSecurityAgency/ghidra#10185](https://github.com/NationalSecurityAgency/ghidra/pull/10185)
**Impact:** CPU exhaustion from a client that connects and disconnects

## Background

The ISF debugger connection handler reads length-delimited protobuf messages from a socket:

```java
while (!socket.isClosed()) {
    RootMessage root = RootMessage.parseDelimitedFrom(inputStream);
    if (root != null) {
        AbstractMessage msg = handler.processMessage(root);
        ...
    }
}
```

`parseDelimitedFrom()` returns `null` when the stream reaches EOF. The loop only handled the non-null case, so a client that connected and then closed the socket left the handler calling `parseDelimitedFrom()` on an exhausted stream in a tight loop. The thread stayed at roughly 100% CPU for the lifetime of the server.

## Reproduction

1. Start the ISF server.
2. Connect a client and send a `PingRequest`.
3. Close the client socket.
4. Observe the handler thread at ~100% CPU.

## Fix

Break out of the loop when `parseDelimitedFrom()` returns `null`, and close the accepted socket in a `finally` block:

```java
RootMessage root = RootMessage.parseDelimitedFrom(inputStream);
if (root == null) {
    // EOF - the client closed the connection
    break;
}
```

## Notes

The condition is not a parser bug; it is a missing EOF check in the connection loop. The same pattern is worth checking in other socket handlers that use `parseDelimitedFrom()`.
