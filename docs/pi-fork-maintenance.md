# Nebell fork: Pi cancellation and recovery

This is a modified copy of [mksglu/context-mode](https://github.com/mksglu/context-mode), maintained at [Nebell/context-mode](https://github.com/Nebell/context-mode). The original copyright and [Elastic License 2.0](../LICENSE) remain in force.

## Changes

The Pi bridge now accepts the tool's `AbortSignal`. Cancellation rejects the active request, resets this session's MCP server, and permits the next call to start a replacement server. Late stdout, exit, and stdin-error events from the old process cannot invalidate the replacement.

A watchdog runs in the Pi host, outside the MCP server. It bounds the whole tool call, including a respawn handshake. It can fire even when synchronous indexing blocks the server's event loop. The executor still receives the original `timeout` argument without changes.

The default watchdog budget is 900000 ms for calls without an explicit execution timeout. Set `CONTEXT_MODE_PI_CALL_TIMEOUT_MS` before starting Pi to change that default. Positive finite values are accepted and rounded up to milliseconds. Explicit execution budgets take precedence:

- `ctx_execute` and `ctx_execute_file`: execution timeout + 60000 ms initialization + 30000 ms completion.
- Rust adds a separate compile allowance of `min(timeout, 60000)` ms.
- Serial `ctx_batch_execute` shares one execution timeout. Parallel batches reserve `ceil(command count / concurrency)` execution timeouts. The command pool does not cap concurrency by CPU count.
- All watchdog values are capped at 2147483647 ms to prevent runtime timer overflow.

Thus an ordinary `timeout: 900000` call has a 990000 ms bridge budget. Rust has a 1050000 ms budget. These allowances do not extend the executor's own 900000 ms execution timeout.

## Cancellation effects and limits

**Warning:** cancellation or a bridge timeout resets the entire MCP server for this Pi session. Other active context-mode calls and background jobs owned by that server may also stop. Run persistent services outside this server.

On Linux, cleanup takes a `/proc` snapshot before the server exits. It records descendant PID, kernel starttime, and process group, including detached executor groups. Cleanup sends SIGTERM, then SIGKILL after five seconds to groups with a matching surviving process identity. It does not target Pi's own process group.

The snapshot cannot guarantee discovery of descendants that already daemonized and were reparented, or detached children created after the snapshot. Tests verify that executor processes stop running, not that the operating system has collected all zombies. Linux process-tree tests are skipped elsewhere. Windows and macOS have not been validated by this fork on physical or CI hosts; non-Linux cleanup cannot promise equivalent descendant discovery.

This change does not fix the underlying cause of every slow script or SQLite operation. It also cannot interrupt synchronous work already blocking the Pi host itself. Resetting the server does not intentionally delete database files, but in-memory server state is lost.

## Install from Git

The Pi package manifest points to the tracked TypeScript entry, `src/adapters/pi/extension.ts`. Pi's own loader handles TypeScript and relative `.js` imports that resolve to TypeScript source. The contributor entry under `.pi/extensions/context-mode/` uses the same source. The package includes `src/` for packed installations too.

No generated `build/` directory, TypeScript compiler, development dependencies, or install-time build is required for Pi. The extension starts the tracked `server.bundle.mjs` for MCP tools. Keep that bundle in sync when changing server code; the Pi bridge itself loads from source.

After verifying the fork, replace the official package:

```sh
pi remove npm:context-mode
pi install https://github.com/Nebell/context-mode
```

Do not keep both package sources enabled. Fully restart Pi afterward. Pushing the fork alone does not change an existing npm installation.

Use Pi's package updater for this Git source:

```sh
pi update https://github.com/Nebell/context-mode
# Or update all configured packages:
pi update --extensions
```

Pi can clean generated files and install only production dependencies without losing the source entry. Use Pi's updater rather than `ctx_upgrade`, whose upstream upgrade path is not a fork maintenance mechanism. Record the validated commit hash; specify a Git commit or tag in the package source if a fixed revision is required.

## Develop locally

```sh
git clone https://github.com/Nebell/context-mode.git
cd context-mode
npm install --ignore-scripts
npm rebuild better-sqlite3
npm run build
node node_modules/vitest/vitest.mjs run tests/adapters/pi-mcp-bridge.test.ts
```

Developers can also install the checkout with `pi install "$PWD"`, instead of the Git source. Local checkouts require manual pull and dependency management. Do not install both copies at once or update while a build is running.

## Maintenance

Keep changes in `src/adapters/pi/mcp-bridge.ts`, with regression coverage in the existing `tests/adapters/pi-mcp-bridge.test.ts`. Rebuild instead of editing generated JavaScript. The local version/hash-gated patch manager used before this fork is not part of the repository or its update path.

Before integrating upstream changes, compare the cancellation and timeout implementations. Preserve the long-running execution contract: do not restore the old fixed 120-second transport timeout. Test cancellation, subsequent recovery, parallel batches, Rust compilation, slow initialization, and process-group isolation after a merge.

Validation commands:

```sh
npm run typecheck
npm run build
node node_modules/vitest/vitest.mjs run tests/adapters/
```

On Linux, the focused Pi suite includes a real MCP-server integration test with isolated temporary storage. It covers execute, indexing, search, executor timeout, cancellation, and a subsequent successful call. This integration test is skipped on other operating systems because executor-tree cleanup there is not guaranteed. Long Rust and initialization budgets use simulated timers, not 15-minute wall-clock runs.
