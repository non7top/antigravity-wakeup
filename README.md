# antigravity-wakeup

Wake an idle Antigravity CLI (`agy`) conversation from another process.

An MCP server cannot start a turn in `agy`. The only way is `agy agentapi send-message`, which needs `ANTIGRAVITY_LS_ADDRESS` and `ANTIGRAVITY_CSRF_TOKEN`. `agy` puts those (and `ANTIGRAVITY_CONVERSATION_ID`) only into its agent's shell tool, never into MCP servers or hook processes. This library works around that with a one-time "prime" step run from the agent's own shell.

## How it works

1. A host process (for example an MCP server running under `agy`) creates a `Prime`, builds a poker with `prime.new_poker()` and serves its control socket (`poker.serve_control_socket(path)`, a `0600` Unix socket).
2. The agent runs the command from `prime.command()` in its shell tool. That runs `antigravity-wakeup --prime`, which reads the `ANTIGRAVITY_*` environment, finds the host process under the same `agy` process and sends the endpoint to its control socket.
3. The host calls `poker.notify(sender)` whenever something arrives. Notifications are debounced, then the poker runs `agy agentapi send-message` with text from your `wake_text` callback. A send counts as delivered only when agy's reply names the conversation (`response.sendMessage.recipientId`).
4. A marker file records which host is primed for a conversation. The `PreInvocation` hook (`antigravity-wakeup --prime-hook`) reads it and, for an unprimed conversation, injects an `ephemeralMessage` telling the agent to run the prime command. After a failed attempt the hook waits 10 minutes before asking again.

The token is held in memory only and reaches the `agy` child through its environment, never argv, logs or disk. Only loopback addresses are accepted. Optional `pin_agy_process=True` also checks that the primed `agy` process (pid plus start time) is still the same one before each send.

## Use

```python
from antigravity_wakeup.prime import Prime

prime = Prime(
    script_name="my-tool",              # console script that runs --prime / --prime-hook
    module="my_tool.cli",               # fallback for `python -m` when the script is not found
    notice="...",                       # text injected by the hook for an unprimed conversation
    wake_text=lambda senders: "...",    # text sent on each wake; must not carry peer-written content
)
poker = prime.new_poker()
# in the host's event loop:  asyncio.create_task(poker.serve_control_socket(path))
# when something arrives:    poker.notify(sender)
# to tell the agent what to run: prime.command()
```

The `antigravity-wakeup` command exposes `--prime` and `--prime-hook` for a host that has no CLI of its own; `antigravity_wakeup.cli.main(prime)` takes a configured `Prime`.

## Current limits

Several defaults still come from the project this was split out of: marker and stable-link directories under `~/.claude/chatter-*`, and host discovery through Claude Code's `~/.claude/sessions` files (`messagingSocketPath`). Making those pluggable is open work.

## Development

```
make build
make lint
make test
make package
```

Everything runs in a throwaway container with its own `$HOME`.

## License

MIT
