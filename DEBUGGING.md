# Debugging a test in VS Code

This repo is a large multi-module Gradle build with Android submodules (`cronet`,
`android`, `binder`, ...) and a codegen step (`grpc-compiler`). Both of these trip up
VS Code's Java tooling in non-obvious ways. This doc captures the setup that actually
works, and why, so you don't have to rediscover it.

## One-time setup

Create (or merge into) `.vscode/settings.json` at the repo root:

```json
{
  "java.import.gradle.arguments": "-PskipAndroid=true -PskipCodegen=true",
  "java.test.config": {
    "vmArgs": ["-Xdebug"]
  }
}
```

Why each key exists:

- **`java.import.gradle.arguments`**: VS Code's "Build Server for Gradle" extension
  (which resolves the project model for CodeLens, test discovery, etc.) runs its own
  Gradle invocation that does **not** reliably pick up the root `gradle.properties`
  (even though a plain `./gradlew` from a terminal does). Without this, project import
  tries to configure `:grpc-cronet`/`:grpc-android`/etc. and fails with `SDK location
  not found` (Android SDK isn't installed), which breaks project import entirely —
  no compile errors will show, but **no Run/Debug Test CodeLens will ever appear**
  above `@Test` methods, and "Debug Test" will hang on "Importing projects."
- **`java.test.config.vmArgs: ["-Xdebug"]`**: many tests in this repo guard their
  `@Rule` timeouts with JUnit's `DisableOnDebug`, e.g.:
  ```java
  @Rule public final TestRule globalTimeout = new DisableOnDebug(Timeout.seconds(10));
  ```
  `DisableOnDebug` disables the timeout only if it finds `-Xdebug` or an
  `-agentlib:jdwp...`-prefixed argument in the JVM's own startup arguments. VS Code's
  CodeLens-driven "Debug Test" launch does not happen to produce either of those by
  default, so the timeout stays active and you'll get a spurious
  `TestTimedOutException` the moment you pause at a breakpoint for more than the
  rule's timeout. Adding `-Xdebug` (a harmless no-op flag on modern JVMs — verified
  against the Temurin 21 JDK this repo uses) to every test launch satisfies that
  check without touching any test source or actually wiring up a second debug
  agent.

After creating/editing this file for the first time, force a clean re-import so the
new Gradle arguments actually take effect (settings changes to
`java.import.gradle.arguments` are not picked up by an already-running import):

1. Command Palette → **"Java: Clean the Java Language Server Workspace"**, or
2. Command Palette → **"Developer: Reload Window"**.

Confirm it worked by opening the **"Gradle for Java"** output channel (View → Output,
select it from the dropdown). You should see:

```
*** Skipping the build of codegen and compilation of proto files because skipCodegen=true
  * Skipping the build of Android projects because skipAndroid=true
...
CONFIGURE SUCCESSFUL in ...
Found N tasks
```

If instead you see `*** Android SDK is required...` and a `CONFIGURE FAILED`, the
settings change hasn't been picked up yet — reload the window again.

## Setting a breakpoint and debugging (primary flow)

Once the above is in place and import succeeds:

1. Open the test file, click in the gutter next to the line you want to break on (a
   red dot appears).
2. Above the `@Test` method (or the class declaration) a small CodeLens line reads
   **`Run Test | Debug Test`**. Click **Debug Test**.
3. Execution runs and stops at your breakpoint. Use the floating debug toolbar
   (Step Over / Step Into / Step Out / Continue) or `F10`/`F11`/`Shift+F11`/`F5`.
   Inspect state via the Variables/Watch/Call Stack panels in the Run and Debug
   sidebar, or the Debug Console to evaluate expressions in the current frame.

Thanks to the `-Xdebug` vmArg above, you can stay paused as long as you want without
tripping a `DisableOnDebug`-guarded test timeout.

## Fallback flow: manual attach (if CodeLens is unavailable or misbehaving)

If the Build Server for Gradle / Java Test Runner extensions are in a bad state
(crashed, mid-reimport, etc.), you can always bypass them entirely:

`.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "Attach to Gradle test",
      "request": "attach",
      "hostName": "localhost",
      "port": 5005
    }
  ]
}
```

Then, for a given test (example: `:grpc-core`'s `DnsNameResolverTest`):

```sh
./gradlew :grpc-core:test --tests "io.grpc.internal.DnsNameResolverTest.resolve_addressResolutionError" --debug-jvm
```

This prints `Listening for transport dt_socket at address: 5005` and suspends,
waiting for a debugger. **Set your breakpoints and select "Attach to Gradle test" in
the Run and Debug panel *before* you start this command** — Gradle only waits ~120
seconds for the debugger to connect before it tears down the test worker and fails
the build with `Unable to connect to the child process`. As soon as the port is
confirmed listening, attach immediately (`F5`).

This flow passes `-agentlib:jdwp=...` on the real JVM command line, so
`DisableOnDebug`-guarded timeouts are correctly disabled automatically — no
`-Xdebug` workaround needed here.

## Troubleshooting checklist

If debugging seems stuck or isn't hitting breakpoints, check in this order:

1. **Is this the first time importing in this window?** A multi-module project this
   size can take minutes; don't assume it's hung. Check the "Gradle for Java" output
   channel for progress.
2. **Did project import actually succeed?** Look for `CONFIGURE SUCCESSFUL` and a
   non-zero task count in the "Gradle for Java" output channel. A failed import
   (commonly the Android SDK issue above) silently breaks CodeLens without any
   visible error in the editor.
3. **Is the Java language server alive?** Run `jps -l` in a terminal and look for
   the `redhat.java-*/server/plugins/org.eclipse.equinox.launcher_*.jar` process. If
   it's missing, the server crashed; VS Code should auto-restart it, but you can also
   force it by deleting the cached workspace data under
   `~/Library/Application Support/Code/User/workspaceStorage/<hash>/redhat.java/ss_ws`
   (macOS) and reloading the window.
4. **For the manual attach flow**: confirm the port is actually accepting a
   connection, not just listening — `lsof -i -P -n | grep 5005` should show an
   `ESTABLISHED` line once you've attached, not just `LISTEN`. If it stays `LISTEN`
   forever, the debugger never actually connected (wrong config selected, stale
   `launch.json`, etc.) — this looks identical to a slow/hung JVM from the outside,
   so don't assume "it's just slow."
5. **Timeout failure right after a successful breakpoint hit**: this is almost
   certainly the `DisableOnDebug` issue above, not a real test failure — check
   whether the test class has a `@Rule ... DisableOnDebug(Timeout.seconds(N))` and
   whether `java.test.config.vmArgs` includes `-Xdebug`.
