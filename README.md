> **Moved.** This plugin now lives in [heliowap/opencode-plugins](https://github.com/heliowap/opencode-plugins/tree/main/plugins/model-policy). Install: `opencode plugin add 'github:heliowap/opencode-plugins#main::path:plugins/model-policy'`. This repository is archived.

# opencode-v2-model-policy

OpenCode 2 plugin that enforces which model a subagent runs on, per agent, with
provider fallback.

The parent session can pass any `model` to the `subagent` tool. For agents with a
policy, an `execute.before` hook replaces it with the policy route, so a delegated
agent always runs where you decided it should. The plugin does not wrap the
`subagent` tool itself: OpenCode 2.0.25 reported the first failure's error for
every parallel call that failed once a plugin wrapped the tool. That bug is fixed
in 2.1.4. The regression canary in `host/upstream` stays in the suite and fails if
it comes back.

## Policy

| Agent | Routes, in order |
| --- | --- |
| `implementer` | `cli_proxy/cpa-claude-haiku-5-5#medium` → `cli_proxy/cpa-claude-sonnet-5-5#medium` → `cli_proxy_openai/gpt-6.1-sol#medium` |
| `researcher`, `explore` | `cli_proxy/cpa-claude-sonnet-5-5#medium` → `cli_proxy/cpa-claude-opus-5-5#low` → `cli_proxy_openai/gpt-6.1-sol#medium` |
| `planner` | `cli_proxy/cpa-claude-opus-5-5#high` → `cli_proxy_openai/gpt-6.1-sol#high` |
| `general`, `poteto-agent` | the parent's model (no policy) |

Every other agent also keeps the model the parent chose. Routes are defined in
[`src/policy.ts`](src/policy.ts).

- A route missing from the model catalog, or lacking the effort variant, is
  skipped. If no route is left, the dispatch fails instead of using another model.

## Fallback

A provider failure (rate limit, quota, auth, transport, timeout, missing route,
provider error) moves the failed route to the back for 10 minutes. The failure
is returned to the parent, and its next dispatch of that agent starts on the
next route. This works the same for foreground and background dispatches.

- Rate-limit and quota failures skip OpenCode's own retries when another route
  is available, so the failure reaches the parent in seconds instead of minutes.
- Other provider errors (5xx, transport) keep OpenCode's retries, about 80
  seconds, before the route is demoted.
- Task failures (invalid request, content filter, tool errors) and cancellation
  do not demote the route.

Each successful result carries `metadata.modelPolicy`: the agent, the requested
model, the chosen route, the skipped routes and the demoted routes. Prompts are
never recorded.

## Quota-aware routing

Before it dispatches a policied agent, the plugin reads Claude quota and skips
the Claude routes when the account pool is exhausted. The Sol route stays
available, so the dispatch moves there.

- **Rule.** A Claude account is exhausted when less than 5% is left in its 5h or
  7d window. The pool is judged by the best enabled Claude account: it is
  exhausted only when every enabled account is. Accounts of other providers, and
  disabled accounts, are ignored.
- **Scope.** Only Claude routes (`cli_proxy/cpa-claude-*`) are skipped. Sol is
  never skipped for quota.
- **Unknown quota.** Missing configuration, an unreachable proxy, an error or
  malformed response, or no enabled Claude account all count as not exhausted.
  Unknown quota never skips a route.
- **All routes exhausted.** The declared order is kept, so the dispatch still
  runs somewhere.
- **Skips are visible.** Each skipped route appears in `metadata.modelPolicy.skipped`
  with the reason `account quota below 5%`.
- **Retry agrees with dispatch.** A provider failure is judged against the same
  exhausted set the dispatch used, so a route with no fallback is retried in place
  instead of failing back to the parent. A child that started before a quota read
  and is rate-limited after it fails fast the same way, when another route is
  available.

Configuration comes from two environment variables, read by the OpenCode service:

- `CLIPROXY_URL`: the CLIProxyAPI base URL. Default `http://127.0.0.1:8317`. An
  empty value uses the default.
- `CLIPROXY_MANAGEMENT_KEY`: the management key for the credentials endpoint.

Both must be set in the **OpenCode service's** environment, not only in an
interactive shell. A key exported only in `~/.zshrc` is not seen by a service
started outside an interactive shell. In that case quota stays unknown and the
feature silently does nothing: nothing is skipped and nothing is reported as an
error.

Timing and access:

- The plugin reads `GET /v8/management/credentials` on the proxy and never calls
  Anthropic directly.
- A result is cached for 60 seconds. A read that takes longer than 2 seconds
  counts as unknown.
- After a 401, 403 or 429 from the credentials endpoint, the plugin does not call
  it again for 10 minutes. Quota reads in that window are unknown.
- Agents without a policy never read quota.

## Coverage

The plugin intercepts `subagent` via `execute.before`. These paths create child
sessions without it and are not enforced:

- Commands with `subagent: true` (or whose agent has `mode: subagent`). They use
  the command's `model`, then the agent's default model.
- Background subagents resumed after a server restart.

Set each agent's default `model` to its first route so these paths start on the
right model.

## Install

```sh
git clone https://github.com/heliowap/opencode-v2-model-policy
cd opencode-v2-model-policy
bun install
ln -s "$PWD/src" ~/.config/opencode/plugins/model-policy
```

Check that it is active:

```sh
opencode api get /api/plugin
```

## Development

```sh
bun test            # policy, failure classification and quota unit tests
bun run typecheck
bun test ./host     # starts real `opencode serve` instances against fixture providers
```

The host tests need the `opencode` binary on `PATH` (or `OPENCODE_BIN`). They
were written against OpenCode 2.0.25 and pass on 2.1.4. Tests that load your
global plugins or the `code-review` skill are skipped when those files are
missing. The host harness clears `CLIPROXY_URL` and `CLIPROXY_MANAGEMENT_KEY`, so
the tests never reach your real proxy.

## License

MIT
