# Investigation handoff: invalid port exception is not propagated by DNS resolution

## Objective

Fix a gRPC Java name-resolution liveness bug exposed by Apache Arrow Java. When a client channel is created with an invalid destination port such as `65537`, an `IllegalArgumentException` escapes from gRPC's resolver executor. The resolver listener is never notified, so the RPC waiting for name resolution can remain pending indefinitely.

This branch is intended to produce:

1. a focused gRPC regression test;
2. agreement on the correct gRPC contract and failure behavior;
3. a production fix in gRPC Java; and
4. an end-to-end check showing that Apache Arrow's JDBC connection attempt no longer hangs when it reaches this gRPC path.

## Repository and branch state

- Repository: `grpc/grpc-java` (the working fork remote is `ennuite/grpc-java`).
- Branch: `invalid-port-non-propagated-exception`.
- The branch currently contains a minimal failing test in:
  `core/src/test/java/io/grpc/internal/DnsNameResolverTest.java`.
- Test name: `resolve_invalidPort_reportsError`.
- No production fix has been implemented yet.

The test constructs a `RetryingNameResolver` for `addr.fake:65537`, starts it, and runs the queued resolution task. Its intended contract is that the task itself does not throw and that the listener receives a failed `ResolutionResult` whose cause is the original `IllegalArgumentException`. At present, the exception escapes from `fakeExecutor.runDueTasks()`, so the test fails before listener verification.

## Original Arrow Java context

The problem was exposed while working on Apache Arrow Java PR [#1027](https://github.com/apache/arrow-java/pull/1027), which makes Flight's TCP client respect JVM proxy settings.

Arrow changed the Flight client channel construction from:

```java
NettyChannelBuilder.forAddress(location.toSocketAddress())
```

to:

```java
NettyChannelBuilder.forAddress(location.getUri().getHost(), location.getUri().getPort())
```

The `String, int` overload is the normal proxy-aware path. It preserves the host name long enough for gRPC's `ProxyDetector`/`ProxySelector` path to run. The `SocketAddress` overload constructs an `InetSocketAddress` in Arrow before gRPC starts and therefore rejects port `65537` synchronously.

Arrow already had this JDBC test:

```text
flight/flight-sql-jdbc-core/src/test/java/
  org/apache/arrow/driver/jdbc/ConnectionTest.java

testUnencryptedConnectionProvidingInvalidPort
```

The test calls `DriverManager.getConnection()` with an Arrow Flight SQL JDBC URL containing port `65537` and expects `SQLException`. After switching to the proxy-aware overload, CI hung instead of completing the assertion.

Arrow's final PR added its own port validation in `flight/flight-core/src/main/java/org/apache/arrow/flight/grpc/NettyClientBuilder.java`. That is useful defensive validation but masks the lower-level gRPC behavior. The intermediate Arrow commit `30efec3dd` contains the proxy-aware overload and invalid-port JDBC test without the Arrow-side validation, so it is a useful reference point for end-to-end validation of the gRPC fix.

This is not evidence that proxy-aware channel construction is wrong. It exposed a previously hidden gRPC resolver failure path.

## Observed behavior and thread model

There are two independent activities:

1. Arrow's connection-establishment thread starts a Flight handshake and waits for its observer to complete.
2. gRPC's resolver executor performs name resolution and proxy detection.

In the observed Arrow run:

- The Arrow thread reaches `ClientHandshakeWrapper.doClientHandshake()` in
  `flight/flight-core/src/main/java/org/apache/arrow/flight/auth2/ClientHandshakeWrapper.java`.
- It calls `stub.handshake(observer)` and then waits on `observer.completed.get()`.
- A gRPC worker named `grpc-default-executor-0` runs `DnsNameResolver.Resolve.run()`.
- That worker throws `IllegalArgumentException: port out of range:65537` and dies.
- No gRPC callback completes the handshake observer exceptionally, so Arrow remains blocked in `completed.get()`.

`completed.get()` is not waiting on the resolver executor's `Runnable` or its exception. It waits on the Flight RPC observer. The only way the resolution failure can reach the RPC is through the gRPC name-resolver listener/channel machinery. An uncaught exception printed by the executor is therefore not sufficient propagation.

## Relevant gRPC classes and methods

### `io.grpc.netty.NettyChannelBuilder`

`forAddress(String, int)` converts the host and port into a target/authority rather than eagerly creating an `InetSocketAddress`. This is why proxy detection and port validation occur later than with `forAddress(SocketAddress)`.

Related helper: `io.grpc.internal.GrpcUtil.authorityFromHostAndPort(String, int)`. It uses `URI` construction, which does not reject port `65537`.

### `io.grpc.internal.ManagedChannelImpl`

Creates and wires the `NameResolver` used by the channel. The resolver is wrapped in `RetryingNameResolver`, so name-resolution attempts are expected to communicate their outcome through the listener.

### `io.grpc.internal.DnsNameResolver`

This is the central class for the bug.

- `resolve()` sets `resolving = true` and submits `new Resolve(listener)` to the resolver executor.
- `Resolve.run()` performs one asynchronous resolution attempt.
- `detectProxy()` creates the unresolved destination and asks the configured `ProxyDetector` for a proxy:

  ```java
  InetSocketAddress destination =
      InetSocketAddress.createUnresolved(host, port);
  ProxiedSocketAddress proxiedAddr = proxyDetector.proxyFor(destination);
  ```

- `InetSocketAddress.createUnresolved(host, 65537)` throws `IllegalArgumentException`.
- `Resolve.run()` catches `IOException`, but not `IllegalArgumentException`.
- Its `finally` block clears `resolving` and updates cache state. It does **not** notify the listener or schedule another attempt.
- `doResolve()` already converts address-resolution exceptions into a failed `ResolutionResult`, but this exception occurs earlier in `detectProxy()`, so that protection does not apply.

### `io.grpc.internal.RetryingNameResolver`

`RetryingNameResolver.RetryingListener` schedules another attempt when it receives a failed result through `onResult2()` or an error through `onError()`. It cannot retry an attempt whose `Runnable` terminates without invoking either callback.

There is no independent executor retry for a failed `Runnable`. Clearing `resolving` in `finally` only makes a later explicit refresh possible; it does not cause one.

### `io.grpc.NameResolver.Listener2`

The listener is the resolver-to-channel result path. The key contract question is not whether the executor may print an exception, but whether every started resolution attempt must report an outcome to the listener. The existing `DnsNameResolver` and `RetryingNameResolver` design strongly assumes that it does.

## Root-cause statement

An invalid port is accepted far enough into channel construction to reach asynchronous proxy detection. `DnsNameResolver.Resolve.run()` catches the declared `IOException` from proxy detection but does not handle the `IllegalArgumentException` produced while constructing the unresolved proxy destination. The exception escapes the executor task before `Listener2` receives a result. This bypasses both the managed channel's failure path and `RetryingNameResolver`, leaving the pending RPC without a terminal resolution outcome.

## Relevant implementation history

These commits help explain why the catch is narrow:

- `7df9ae975`: introduced proxy detection; unresolved destination construction and `proxyFor()` were not wrapped together by a general failure handler.
- `402c1740f`: changed `ProxyDetector.proxyFor()` to declare `IOException` because proxy resolution may perform network I/O, and added handling for that declared operational failure.
- `6b7c8694a9`: refactored the resolution runnable while preserving the `finally` cleanup of `resolving`.
- `6a7e47b8a5`: extracted `detectProxy()` but retained the `IOException`-only catch.

The history suggests that handling `IOException` was intentional, but it does not show an intentional decision that unchecked exceptions from destination construction should strand the resolution attempt. The invalid-port path appears to have been outside the cases considered during those changes.

## Proposed direction

The most direct fix is to make the proxy-detection portion of `Resolve.run()` convert this `IllegalArgumentException` into a failed `ResolutionResult` and deliver it through `Listener2.onResult2()`, just as the existing `IOException` path does.

Keep the catch narrow. Catching `RuntimeException` around all of `Resolve.run()` would also catch programming errors from unrelated resolution logic or even listener invocation, potentially turning bugs into repeated `UNAVAILABLE` results. A focused catch around `detectProxy()` avoids swallowing failures outside the operation being hardened.

One possible structure is:

```java
EquivalentAddressGroup proxiedAddr;
try {
  proxiedAddr = detectProxy();
} catch (IOException | IllegalArgumentException e) {
  // Build and deliver the same kind of failed ResolutionResult used by
  // the existing IOException path.
}
```

The exact status code and message should follow the convention maintainers prefer. Existing DNS/proxy failures use `Status.UNAVAILABLE` with the original exception as the cause.

An additional synchronous port-range check in `GrpcUtil.authorityFromHostAndPort()` (and the corresponding public `io.grpc.Grpc` helper) could make `forAddress(host, port)` fail earlier. That is complementary, not a complete substitute: callers can provide a target such as `dns:///host:65537` via `forTarget()`, bypassing the `forAddress()` helper and still reaching `DnsNameResolver`.

If synchronous validation is added, use the Java socket port range `0..65535` unless maintainers explicitly decide to change the public contract for client port `0`. Whether port `0` is meaningful as a destination is separate from whether existing APIs accept it.

## Open design questions for the gRPC discussion

1. Does gRPC want only the resolver liveness fix, or also synchronous validation in the public `forAddress(String, int)` path?
2. Should the resolver report this as `UNAVAILABLE` for consistency with existing resolution failures, or distinguish invalid target configuration?
3. Is `IllegalArgumentException` the intended narrow boundary, or do maintainers want a more general guarantee that a resolver attempt cannot exit without a listener outcome?
4. Because `RetryingNameResolver` retries any non-OK listener result, should deterministic invalid configuration be treated differently from transient DNS/proxy failure?

These questions should not block the regression test: the current hang is undesirable regardless of the final status/retry policy.

## Validation plan

### gRPC unit level

Use `DnsNameResolverTest.resolve_invalidPort_reportsError` as the primary regression test.

Before the production fix, it fails because `IllegalArgumentException` escapes from `fakeExecutor.runDueTasks()`. After the fix, it should prove:

- the resolution task returns normally;
- the listener receives exactly one result;
- the result is non-OK; and
- its cause is the original `IllegalArgumentException`.

Also run the rest of `DnsNameResolverTest` and `RetryingNameResolverTest` to check existing proxy failure and retry behavior.

### Arrow Java end to end

The purpose of this check is to prove that the gRPC failure reaches the Flight RPC and releases the JDBC caller, not merely that the resolver unit test passes.

1. Build the fixed gRPC Java artifacts locally using a distinct snapshot version.
2. Configure an Arrow Java checkout to use that local gRPC snapshot.
3. Use Arrow's proxy-aware channel code without its later synchronous port-validation guard. Intermediate commit `30efec3dd` is the known reference state for that behavior.
4. Run only `ConnectionTest.testUnencryptedConnectionProvidingInvalidPort`.
5. Confirm that `DriverManager.getConnection()` terminates with `SQLException` instead of hanging and that there is no uncaught `IllegalArgumentException` on `grpc-default-executor-0`.
6. Verify from Arrow's resolved dependency tree or debugger source mapping that the test actually loaded the locally built `grpc-core`; otherwise a passing or failing result is inconclusive.

Do not validate against the final Arrow PR state without accounting for its `NettyClientBuilder` port check. In that state Arrow rejects `65537` before invoking gRPC, so the test cannot demonstrate the gRPC fix.

## Useful debugging breakpoints

For the gRPC unit test:

- `DnsNameResolver.resolve()` where it submits `new Resolve(listener)`.
- `DnsNameResolver.Resolve.run()` before and after `detectProxy()`.
- `DnsNameResolver.detectProxy()` at `InetSocketAddress.createUnresolved(host, port)`.
- `RetryingNameResolver.RetryingListener.onResult2()`.

For the Arrow end-to-end test:

- `NettyClientBuilder.build()` where Arrow calls gRPC's proxy-aware overload.
- `ClientHandshakeWrapper.doClientHandshake()` at `stub.handshake(observer)` and `observer.completed.get()`.
- The gRPC breakpoints above.

The expected fixed flow is:

```text
invalid port
  -> resolver reports failed ResolutionResult
  -> managed channel/RPC receives name-resolution failure
  -> Flight handshake observer completes exceptionally
  -> observer.completed.get() stops waiting
  -> JDBC layer returns SQLException
```

## Immediate next steps

1. Refine the minimal test assertions if needed, keeping the test independent of Arrow.
2. Implement the narrow resolver fix around `detectProxy()`.
3. Run the resolver tests and inspect retry behavior for a deterministic invalid port.
4. Prepare a gRPC issue explaining the missing listener outcome and include the Arrow PR as the real-world reproducer.
5. Use the Arrow validation plan above before claiming the fix resolves the original JDBC hang.
