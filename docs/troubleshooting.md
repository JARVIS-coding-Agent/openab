# Troubleshooting

Diagnostic guide for OpenAB runtime issues. Most user-facing errors fall into one
of the categories below.

## `-32603 Internal Error` with no actionable detail

### Symptom

In Discord / Slack you see:

```
⚠️ **Internal Error** (code: -32603)
Internal error
```

…but no further context, and `kubectl logs` show the same opaque message.

### Why it happens

JSON-RPC 2.0 §5.1 reserves `-32603` as the "Internal error" catch-all. ACP
agents use it for wildly different failure modes (auth, model not supported,
context overflow, crash). When the agent doesn't populate `error.data`, the
client can only display the generic message.

| Agent | `error.data` shape | Diagnostic source |
|-------|-------------------|-------------------|
| codex-acp | `{"message": "<nested JSON>"}` | `data.message` extracted + nested JSON unwrapped |
| opencode | `{}` (empty) | **stderr tail** |
| hermes-agent | absent | **stderr tail** |
| claude-agent-acp | (varies) | `data.message` or stderr |

Since OpenAB 0.8.4 (PR #885) the broker extracts `error.data.message` when
present. Since 0.8.6, when `data` is empty/absent, the recent agent stderr is
shown as a "Recent agent output:" blockquote (with secrets redacted).

### Diagnostic path

1. **Read the user-facing error first.** The stderr blockquote (if any) usually
   points at the root cause directly.
2. **If still unclear**, fetch the broker logs:
   ```bash
   kubectl logs -l app.kubernetes.io/name=openab --since=10m | grep -A 5 "agent="
   ```
   The `tracing::warn!` path in `connection.rs` always emits every sanitized
   stderr line with `agent=<command name>`.
3. **Match the cause to the table below and apply the fix.**

### Common causes

| stderr / data.message pattern | Cause | Fix |
|------------------------------|-------|-----|
| `Missing API key`, `API key not set`, `unauthorized`, `401` | Missing / invalid credentials | Verify the `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `*_TOKEN` env var is set in `[agent].env` or `[agent].inherit_env`. See `config.toml.example`. |
| `model 'X' not supported`, `does not exist`, `deprecated` | Wrong / deprecated model | Switch `model` via `session/set_config_option`, or pin a supported model in agent CLI args. |
| `rate limit`, `429`, `quota` | Upstream rate limit | Wait, or reduce concurrency via `pool.max_sessions`. |
| `context length`, `too many tokens`, `maximum context` | Prompt too long | Start a new thread (auto-resets history), or `/cancel` then retry with a shorter prompt. |
| `connection refused`, `DNS`, `tls handshake` | Network / DNS | Check egress from broker pod; verify `cluster.egressProxy` if set. |
| Empty stderr + opaque `-32603` | Bug in agent | File an issue against the agent upstream with the `acp_recv` debug log line. |

### Verifying the fix landed

After deploying a build with the stderr-tail fallback:

```bash
# Confirm the new version is running
kubectl get pods -l app.kubernetes.io/name=openab -o jsonpath='{.items[0].spec.containers[0].image}'

# To force an auth failure for testing, update your config.toml to set an
# invalid API key for a test agent, then restart the pod and send a prompt.
# You should now see:
# ⚠️ **Internal Error** (code: -32603)
# Internal error
# > _Recent agent output:_
# > Error: ANTHROPIC_API_KEY not set
```

## Other categories

- **Connection Lost** — agent process crashed mid-prompt. `kubectl logs` for the
  broker will show `Agent process died` from the broker's liveness check.
- **Request Timeout** — agent didn't respond within 30s (or 120s for
  `session/new`). Either upstream is slow, agent is hung on a tool call, or
  the env config is wrong and initialization is looping.
- **Agent Not Found** — the configured `command` doesn't exist or isn't
  executable. Check `[agent].command` in `config.toml`.
- **Service Busy** — `pool.max_sessions` reached. Increase the limit or
  wait for existing sessions to TTL out (`pool.session_ttl_hours`).
