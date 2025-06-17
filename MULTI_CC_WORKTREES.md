# Multiple Claude Code Instances with Git Worktrees Support

## Overview

This document outlines a comprehensive plan to upgrade claudecode.nvim to support multiple Claude Code instances, where each instance is connected to the Neovim instance that started it. This enables N git worktrees with N AI assistants working in parallel.

## Current Architecture Analysis

### Current Limitations

1. **Single Server Instance**: The plugin maintains a single WebSocket server state in `M.state.server`
2. **Single Port**: Only one port is tracked at a time in `M.state.port`
3. **Single Lock File**: Creates one lock file at `~/.claude/ide/[port].lock`
4. **Global State**: Uses module-level state that prevents multiple instances
5. **Single Terminal**: Terminal module manages one Claude instance at a time

### Key Components to Modify

1. **Server Management** (`lua/claudecode/server/init.lua`)
   - Currently uses singleton pattern with `M.state`
   - Needs instance-based architecture

2. **Lock File System** (`lua/claudecode/lockfile.lua`)
   - Creates discovery files for Claude
   - Needs to support multiple lock files with instance identification

3. **Terminal Management** (`lua/claudecode/terminal.lua`)
   - Manages Claude Code terminal windows
   - Needs to track multiple terminals per instance

4. **Main Plugin Module** (`lua/claudecode/init.lua`)
   - Central coordination point
   - Needs instance manager and routing logic

## Proposed Architecture

### 1. Instance-Based Design

```lua
-- New instance manager
local InstanceManager = {
  instances = {},  -- Map of instance_id -> instance_data
  active_instance = nil,  -- Currently active instance
}

-- Instance data structure
local Instance = {
  id = "unique-id",  -- Generated UUID or worktree path hash
  server = nil,      -- WebSocket server for this instance
  port = nil,        -- Port for this instance
  terminal = nil,    -- Terminal state for this instance
  worktree_path = "/path/to/worktree",
  neovim_pid = vim.fn.getpid(),
  created_at = os.time(),
}
```

### 2. Multi-Instance Server Architecture

#### Server Modifications

```lua
-- lua/claudecode/server/instance.lua (NEW)
local Instance = {}
Instance.__index = Instance

function Instance:new(config)
  local instance = {
    id = vim.fn.sha256(config.worktree_path .. os.time()),
    config = config,
    server = nil,
    port = nil,
    clients = {},
    handlers = {},
    ping_timer = nil,
  }
  setmetatable(instance, Instance)
  return instance
end

function Instance:start()
  -- Start WebSocket server for this instance
  -- Return port
end

function Instance:stop()
  -- Stop this instance's server
end
```

### 3. Enhanced Lock File System

#### Lock File Format Update

```json
{
  "pid": 12345,
  "neovimPid": 67890,  // NEW: Neovim process ID
  "instanceId": "abc123",  // NEW: Unique instance identifier
  "workspaceFolders": ["/path/to/worktree"],
  "worktreePath": "/path/to/worktree",  // NEW: Git worktree path
  "ideName": "Neovim",
  "transport": "ws",
  "createdAt": 1234567890,  // NEW: Timestamp
  "parentPort": 10001  // NEW: Optional parent instance port
}
```

### 4. Instance Discovery and Management

#### Instance Manager Implementation

```lua
-- lua/claudecode/instance_manager.lua (NEW)
local M = {}

-- Get instance for current buffer/worktree
function M.get_current_instance()
  local worktree_path = M.detect_worktree_path()
  return M.instances[worktree_path]
end

-- Create new instance for worktree
function M.create_instance(worktree_path, config)
  local instance = Instance:new({
    worktree_path = worktree_path,
    config = config,
  })
  
  M.instances[worktree_path] = instance
  return instance
end

-- Detect git worktree for current buffer
function M.detect_worktree_path()
  local current_file = vim.fn.expand("%:p")
  -- Use git rev-parse to find worktree root
  local handle = io.popen("git -C " .. vim.fn.shellescape(vim.fn.fnamemodify(current_file, ":h")) .. " rev-parse --show-toplevel 2>/dev/null")
  local worktree_path = handle:read("*l")
  handle:close()
  return worktree_path
end
```

### 5. Updated Command Structure

#### New Commands

```vim
" Instance-aware commands
:ClaudeCode                    " Opens Claude for current worktree
:ClaudeCodeList               " List all active instances
:ClaudeCodeSwitch <instance>  " Switch to specific instance
:ClaudeCodeKill <instance>    " Stop specific instance
:ClaudeCodeKillAll            " Stop all instances

" Instance-specific operations
:ClaudeCodeSend               " Sends to current worktree's Claude
:ClaudeCodeAdd <file>         " Adds to current worktree's Claude
```

### 6. Terminal Management Updates

#### Multi-Terminal Support

```lua
-- lua/claudecode/terminal/manager.lua (NEW)
local TerminalManager = {}

function TerminalManager:new()
  return {
    terminals = {},  -- Map of instance_id -> terminal
  }
end

function TerminalManager:get_or_create(instance_id, cmd, env)
  if not self.terminals[instance_id] then
    self.terminals[instance_id] = self:create_terminal(instance_id, cmd, env)
  end
  return self.terminals[instance_id]
end
```

### 7. Environment Variable Isolation

Each Claude instance needs isolated environment variables:

```lua
function get_instance_env(instance)
  return {
    CLAUDE_CODE_SSE_PORT = tostring(instance.port),
    ENABLE_IDE_INTEGRATION = "true",
    CLAUDE_INSTANCE_ID = instance.id,  -- NEW
    CLAUDE_WORKTREE_PATH = instance.worktree_path,  -- NEW
  }
end
```

## Implementation Plan

### Phase 1: Core Infrastructure (Week 1)

1. Create `Instance` class for managing individual Claude connections
2. Implement `InstanceManager` for tracking multiple instances
3. Update lock file format and creation logic
4. Add worktree detection utilities

### Phase 2: Server Modifications (Week 2)

1. Refactor server module to support instance-based operation
2. Update WebSocket handling for multiple concurrent servers
3. Implement port allocation strategy (avoid conflicts)
4. Add instance-specific message routing

### Phase 3: Terminal Integration (Week 3)

1. Create `TerminalManager` for multi-terminal support
2. Update terminal providers (native & snacks) for instances
3. Implement terminal-to-instance association
4. Add visual indicators for instance identification

### Phase 4: Command Updates (Week 4)

1. Update existing commands to be instance-aware
2. Implement new instance management commands
3. Add instance switcher UI (telescope integration?)
4. Create status indicators for active instances

### Phase 5: Testing & Polish (Week 5)

1. Comprehensive test suite for multi-instance scenarios
2. Performance testing with multiple instances
3. Memory leak detection and cleanup
4. Documentation updates

## Configuration Options

```lua
{
  "coder/claudecode.nvim",
  config = {
    multi_instance = {
      enabled = true,
      max_instances = 5,  -- Limit concurrent instances
      auto_start_for_worktree = true,  -- Auto-start when entering new worktree
      instance_indicator = true,  -- Show instance in statusline
      cleanup_on_exit = true,  -- Clean up instances on Neovim exit
    },
    instance_naming = {
      -- How to name instances in UI
      format = "worktree",  -- "worktree", "branch", "custom"
      custom_formatter = nil,  -- Function to generate instance names
    },
  },
}
```

## Usage Examples

### Basic Multi-Instance Workflow

```bash
# Terminal 1: Main project
cd ~/projects/myapp
nvim src/main.lua
:ClaudeCode  # Starts instance for main worktree

# Terminal 2: Feature branch worktree
cd ~/projects/myapp-feature-x
nvim src/feature.lua
:ClaudeCode  # Starts separate instance for this worktree

# Both Claude instances run independently
```

### Advanced Scenarios

```vim
" List all active Claude instances
:ClaudeCodeList
" Output:
" 1. [main] ~/projects/myapp (port: 10001)
" 2. [feature-x] ~/projects/myapp-feature-x (port: 10002) *active*

" Switch to different instance
:ClaudeCodeSwitch 1

" Send selection to specific instance
:'<,'>ClaudeCodeSendTo feature-x

" Kill instance for current worktree
:ClaudeCodeKill
```

## Benefits

1. **Parallel Development**: Work on multiple features simultaneously with separate AI assistants
2. **Context Isolation**: Each Claude instance only sees its worktree's context
3. **Resource Efficiency**: Start/stop instances as needed
4. **Workflow Integration**: Seamless integration with git worktree workflows

## Challenges & Solutions

### Challenge 1: Resource Management
- **Solution**: Implement instance limits and automatic cleanup
- Monitor memory usage per instance
- Provide cleanup commands and auto-cleanup options

### Challenge 2: Instance Discovery
- **Solution**: Use combination of worktree path and Neovim PID
- Implement robust instance matching algorithm
- Handle edge cases (submodules, nested worktrees)

### Challenge 3: User Experience
- **Solution**: Clear visual indicators for active instance
- Statusline integration showing current instance
- Telescope picker for instance switching

### Challenge 4: Backward Compatibility
- **Solution**: Default to single-instance mode
- Detect multi-instance need automatically
- Provide migration guide for existing users

## Testing Strategy

1. **Unit Tests**: Test each component in isolation
2. **Integration Tests**: Test multi-instance scenarios
3. **Performance Tests**: Measure overhead of multiple instances
4. **User Acceptance**: Beta testing with power users

## Migration Path

1. **Version 1.x**: Current single-instance behavior (default)
2. **Version 2.0**: Multi-instance support (opt-in)
3. **Version 2.1**: Multi-instance by default with auto-detection
4. **Version 3.0**: Full multi-instance with advanced features

## Conclusion

This upgrade will transform claudecode.nvim into a powerful multi-instance AI coding assistant, perfectly suited for modern development workflows using git worktrees. The architecture ensures scalability, maintainability, and excellent user experience while preserving backward compatibility.