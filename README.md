# bun-test-network-guard

A `bun test` preload that makes an unmocked outbound `fetch` throw. Loopback stays open.

## Why

A suite that inherits the network gives a different answer depending on where it runs. A sandbox that reaches nothing passes a test that a CI container with egress fails, because code under test quietly fetched a real upstream on a path the test never asserted on. Those unasserted reads are also where the time goes: in the project this came from, blocking outbound fetch took the full run from 49.3s to 13.7s with every test still passing. Not one of them needed the network.

The usual answers (`nock.disableNetConnect`, MSW's `onUnhandledRequest: "error"`) are interceptor libraries shaped around Node's `http` module. Under Bun's native `fetch` they are either awkward or inert. This is 100 lines that wrap `globalThis.fetch` and nothing else.

## Limits

This wraps `fetch` and nothing else. Traffic through `node:http`, `node:net`, `WebSocket`, or `Bun.connect` is not blocked. A blocked call throws, but code under test that catches the error and degrades will still pass its test. A socket-level guard that patches every one of those, plus an `afterEach` that fails any test which attempted a blocked connection, catches both. If your suite talks to the network through anything but `fetch`, you need that broader approach.

## Install

```bash
bun add -d bun-test-network-guard
```

`bunfig.toml`:

```toml
[test]
preload = ["bun-test-network-guard/preload"]
```

That applies to `bun test`, to `bun test <path>`, and to anything that shells out to either.

## What it does

- Any `fetch` whose target is not loopback throws `[test-network-guard] blocked outbound fetch to <origin>`.
- Loopback is `localhost`, `::1`, `*.localhost`, and `127.0.0.0/8` matched as four octets. `127.0.0.1.example.com` is a remote host and is blocked.
- Relative URLs, non-HTTP schemes, and strings that are not URLs pass through so `fetch` can report them itself.
- The error carries the origin only. Webhook URLs keep their secret in the path and API keys in the query string, and this message lands in CI logs.
- Guidance prints to stderr once per process, not once per blocked call, because code under test tends to catch and log the error.
- A test that assigns its own mock over `globalThis.fetch` wins. The guard only decides what an unmocked call does.

## Escape hatch

```bash
TEST_ALLOW_NETWORK=1 bun test test/integration.test.ts
```

## Custom install

For a different allow variable, or a hint that names where your repo keeps its fixture fetchers, write your own preload:

```ts
// test/preload.ts
import { installNetworkGuard } from "bun-test-network-guard";

installNetworkGuard({
  allowEnv: "MYAPP_TEST_ALLOW_NETWORK",
  hint: "Fixture fetchers live in test/fixtures/fetch.ts.",
});
```

`installNetworkGuard` returns a function that restores the previous `fetch`, which is what a test of the guard itself needs.

## API

- `installNetworkGuard(options?)`: wraps `globalThis.fetch` unless `process.env[allowEnv] === "1"`. Returns a restore function.
- `createGuardedFetch(passThrough, explain, options?)`: the wrapper itself, for composing with another fetch.
- `isLocalTarget(url)`, `targetUrl(input)`, `redactTarget(url)`: the pieces, exported for tests and for reuse.

## License

MIT
