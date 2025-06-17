# ADR-001: Use WebSocket Protocol for Multi-Instance Support

**Date**: 2025-01-15  
**Status**: Accepted  
**Authors**: vng  

## Context

As we plan to support multiple Claude Code instances (one per git worktree), we need to decide how to handle multiple concurrent connections between Neovim and Claude processes. Two options were considered:

1. **Multiple WebSocket servers** (current approach) - Each instance gets its own port
2. **WebTransport with multiplexing** - Single server with multiplexed streams

## Decision

We will continue using **WebSocket protocol** with multiple server instances, one per Claude Code instance.

## Rationale

### 1. Claude Code Compatibility
- Claude Code CLI explicitly expects WebSocket connections
- Lock files specify `"transport": "ws"`
- No evidence of WebTransport support in Claude Code
- Changing protocols would break compatibility

### 2. Technical Constraints
```lua
-- Current implementation uses vim.loop (libuv)
local server = vim.loop.new_tcp()
server:bind("127.0.0.1", port)
```
- Neovim has no native HTTP/3 or QUIC support
- WebTransport would require external C dependencies
- Would significantly increase implementation complexity

### 3. Simplicity Wins
```lua
-- Multiple servers approach is straightforward
InstanceManager = {
  instances = {
    ["/worktree/main"] = { port = 10001, server = server1 },
    ["/worktree/feature"] = { port = 10002, server = server2 },
  }
}
```

### 4. Process Isolation Benefits
- Each Claude process connects to its dedicated server
- Natural security boundary between instances
- No risk of message routing errors
- Simple debugging (can monitor specific ports)

### 5. Resource Efficiency
- WebSocket servers are lightweight (~5MB per instance)
- Port range 10000-65535 provides 55,535 available ports
- vim.loop efficiently handles multiple TCP servers
- No practical limit for typical usage (5-10 instances)

## Consequences

### Positive
- ✅ Zero changes needed to Claude Code protocol
- ✅ Simple implementation (~300 lines vs ~1000+ for WebTransport)
- ✅ Easy to debug with standard WebSocket tools
- ✅ Each instance is fully isolated
- ✅ Works with existing Neovim capabilities

### Negative
- ❌ Uses one port per instance (not really an issue)
- ❌ Slightly more OS resources than multiplexing (negligible)
- ❌ Must manage port allocation (simple with random selection)

### Neutral
- Each instance creates a separate lock file in `~/.claude/ide/`
- Need to implement instance cleanup on Neovim exit
- Status tracking becomes slightly more complex

## Implementation Notes

```lua
-- Port allocation strategy
function find_port_for_instance(instance_id)
  local min, max = 10000, 65535
  for i = 1, 100 do  -- Try up to 100 times
    local port = math.random(min, max)
    if try_bind(port) then
      return port
    end
  end
  error("Could not find available port")
end

-- Instance cleanup
vim.api.nvim_create_autocmd("VimLeavePre", {
  callback = function()
    for _, instance in pairs(instances) do
      instance:stop()
    end
  end
})
```

## Alternatives Considered

### WebTransport
- **Pros**: Stream multiplexing, better performance, modern protocol
- **Cons**: Not supported by Claude, requires external deps, complex implementation
- **Verdict**: Over-engineering for this use case

### Single Server with Message Routing
- **Pros**: One port, centralized management
- **Cons**: Complex routing logic, breaks Claude's assumption of dedicated connection
- **Verdict**: Would require Claude-side changes

### Unix Domain Sockets
- **Pros**: No port management, faster than TCP
- **Cons**: Not supported by Claude WebSocket client
- **Verdict**: Incompatible with current protocol

## References

- [WebSocket Protocol RFC 6455](https://tools.ietf.org/html/rfc6455)
- [Claude Code Protocol Documentation](../PROTOCOL.md)
- [Current WebSocket Implementation](../../lua/claudecode/server/)
- [Multi-Instance Plan](../../MULTI_CC_WORKTREES.md)

## Review Notes

This decision can be revisited if:
- Claude Code adds WebTransport support
- Neovim adds native HTTP/3 support
- We encounter port exhaustion issues (unlikely)
- Performance becomes a bottleneck (current latency <1ms)