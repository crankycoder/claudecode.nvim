# ADR-002: Agent-to-Agent Protocol as Separate Library

**Date**: 2025-01-15  
**Status**: Proposed  
**Authors**: vng  

## Context

With multi-instance Claude Code support, we have the opportunity to enable agent-to-agent (A2A) communication. This would allow multiple Claude instances to collaborate, share context, and coordinate on complex tasks across different worktrees.

## Decision

**Create a separate library** (`claudecode-a2a` or `mcp-a2a`) to handle agent-to-agent communication, rather than building it into claudecode.nvim.

## Rationale

### 1. Separation of Concerns
- **claudecode.nvim**: IDE integration (editor ↔ Claude)
- **claudecode-a2a**: Agent coordination (Claude ↔ Claude)
- Clean architectural boundaries

### 2. Protocol Independence
```lua
-- claudecode.nvim focuses on MCP-WebSocket
local ide_protocol = "mcp-over-websocket"

-- a2a library can use different transports
local a2a_protocols = {
  "grpc",
  "msgpack-rpc", 
  "nats",
  "redis-pubsub"
}
```

### 3. Broader Applicability
- A2A protocol could work with:
  - Multiple Claude Code instances
  - Different AI agents (GPT, Gemini, etc.)
  - Custom agents and tools
  - CI/CD automation agents

## Proposed Architecture

### A2A Library Design

```lua
-- claudecode-a2a/init.lua
local A2A = {
  -- Agent registry
  agents = {},
  
  -- Message broker
  broker = nil,
  
  -- Protocol handlers
  protocols = {}
}

-- Register an agent
function A2A:register_agent(agent_id, capabilities, endpoint)
  self.agents[agent_id] = {
    id = agent_id,
    capabilities = capabilities,
    endpoint = endpoint,
    status = "online"
  }
end

-- Send message between agents
function A2A:send_message(from_agent, to_agent, message)
  -- Route through broker
  return self.broker:route(from_agent, to_agent, message)
end
```

### Integration with claudecode.nvim

```lua
-- claudecode.nvim would expose hooks
local claudecode = require("claudecode")

-- Optional A2A integration
claudecode.setup({
  a2a = {
    enabled = true,
    library = "claudecode-a2a",
    
    -- Callback when A2A message received
    on_message = function(from_agent, message)
      -- Forward to Claude via existing WebSocket
      claudecode.send_to_claude({
        type = "a2a_message",
        from = from_agent,
        content = message
      })
    end
  }
})
```

### Message Flow Example

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Claude    │     │              │     │   Claude    │
│ Instance 1  │────▶│  A2A Broker  │────▶│ Instance 2  │
│ (main)      │     │              │     │ (feature)   │
└─────────────┘     └──────────────┘     └─────────────┘
      ▲                                          ▲
      │                                          │
      │                                          │
┌─────────────┐                          ┌─────────────┐
│claudecode.  │                          │claudecode.  │
│nvim (main)  │                          │nvim (feat)  │
└─────────────┘                          └─────────────┘
```

## A2A Protocol Considerations

### 1. Message Types
```json
{
  "id": "msg-123",
  "type": "request|response|notification|broadcast",
  "from": "agent-main-worktree",
  "to": "agent-feature-worktree",
  "timestamp": 1234567890,
  "payload": {
    "action": "share_context",
    "data": {
      "files": ["src/main.lua"],
      "summary": "Refactored the main loop"
    }
  }
}
```

### 2. Capabilities Discovery
```json
{
  "agent_id": "claude-main-worktree",
  "capabilities": [
    "code_analysis",
    "test_generation",
    "refactoring"
  ],
  "workspace": "/home/user/project",
  "status": "busy|idle|thinking"
}
```

### 3. Coordination Patterns
- **Request/Response**: Direct agent queries
- **Pub/Sub**: Broadcast updates to all agents
- **Task Queue**: Distribute work across agents
- **Consensus**: Multiple agents verify changes

## Implementation Options

### Option 1: Redis-based Broker
```lua
-- Simple, reliable, supports pub/sub
local redis = require("redis")
local broker = redis.connect("localhost", 6379)

-- Publish to channel
broker:publish("a2a:main-to-feature", message)
```

### Option 2: Built-in Lua Broker
```lua
-- No external dependencies
local broker = {
  channels = {},
  
  subscribe = function(self, channel, callback)
    self.channels[channel] = self.channels[channel] or {}
    table.insert(self.channels[channel], callback)
  end
}
```

### Option 3: Unix Domain Sockets
```lua
-- Fast, local-only communication
local socket = vim.loop.new_pipe()
socket:bind("/tmp/claudecode-a2a.sock")
```

## Benefits of Separate Library

1. **Independent Development**
   - A2A protocol can evolve separately
   - Different release cycles
   - Specialized maintainers

2. **Flexible Integration**
   - Other editors can use the A2A library
   - Works with non-Neovim Claude instances
   - Can bridge different AI systems

3. **Testing Isolation**
   - Test A2A logic without editor complexity
   - Mock agents for testing
   - Protocol compliance testing

4. **Performance Optimization**
   - A2A can use optimal transport (gRPC, etc.)
   - No vim.loop constraints
   - Can run in separate process

## Example Use Cases

### 1. Coordinated Refactoring
```lua
-- Agent 1 identifies refactoring opportunity
a2a:broadcast({
  type = "refactor_proposal",
  changes = "Extract login logic to auth module"
})

-- Agent 2 analyzes impact
a2a:respond({
  type = "impact_analysis",
  affected_files = 12,
  test_coverage = "85%"
})
```

### 2. Distributed Code Review
```lua
-- Multiple agents review different aspects
agent1: "Security review complete - 2 issues found"
agent2: "Performance review - suggests caching"
agent3: "Style review - formatting needed"
```

### 3. Knowledge Sharing
```lua
-- Agent learns something new
a2a:broadcast({
  type = "knowledge_update",
  learning = "Project uses custom error handling pattern"
})
```

## Consequences

### Positive
- ✅ Clean separation of concerns
- ✅ Reusable across different integrations
- ✅ Protocol flexibility
- ✅ Easier testing and maintenance
- ✅ Can evolve independently

### Negative
- ❌ Additional dependency for A2A features
- ❌ More complex deployment
- ❌ Need to maintain protocol compatibility

## Recommendation

1. **Phase 1**: Complete multi-instance support in claudecode.nvim
2. **Phase 2**: Design A2A protocol specification
3. **Phase 3**: Implement claudecode-a2a as separate library
4. **Phase 4**: Add optional A2A hooks to claudecode.nvim

This approach keeps claudecode.nvim focused on IDE integration while enabling powerful agent coordination through a dedicated library.