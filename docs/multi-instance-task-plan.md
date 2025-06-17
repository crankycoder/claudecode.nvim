# Multi-Instance Claude Code Task Plan

## Overview
Transform the single-instance claudecode.nvim plugin to support multiple concurrent Claude Code instances with different working directories, enabling worktree support and instance switching.

## Time Estimates

### Overall Project Timeline
- **Total Development Hours**: 120-175 hours
- **Working Days** (8 hours/day): 15-22 days
- **Calendar Time**:
  - **Focused full-time development**: 3-5 weeks
  - **Part-time development**: 6-10 weeks

### Phase-by-Phase Estimates

| Phase | Description | Raw Hours | With Overhead (30-50%) |
|-------|-------------|-----------|------------------------|
| Phase 1 | Core Architecture Refactoring | 14-20 hours | 18-30 hours |
| Phase 2 | Multi-Port Server Management | 10-13 hours | 13-20 hours |
| Phase 3 | Terminal Management Enhancement | 14-18 hours | 18-27 hours |
| Phase 4 | Command System Enhancement | 13-17 hours | 17-26 hours |
| Phase 5 | UI and Notifications | 9-12 hours | 12-18 hours |
| Phase 6 | Advanced Features | 15-19 hours | 20-29 hours |
| Phase 7 | Testing and Documentation | 15-19 hours | 20-29 hours |
| **Total** | | **90-118 hours** | **120-175 hours** |

### Priority-Based Timeline

#### High Priority (Core Functionality)
- Phases 1, 2, 4.1-4.2
- **Estimated time**: 37-50 hours → 5-8 working days
- **Recommended timeline**: 2 weeks with buffer

#### Medium Priority (Enhanced UX)
- Phases 3, 5, 6.1
- **Estimated time**: 28-37 hours → 4-6 working days
- **Recommended timeline**: 1.5-2 weeks with buffer

#### Low Priority (Nice to Have)
- Phases 6.2, 6.3, 5.1
- **Estimated time**: 25-32 hours → 3-5 working days
- **Recommended timeline**: 1-1.5 weeks with buffer

### Development Recommendations

1. **Staged Deployment**: Implement and deploy high-priority features first to get early feedback
2. **Buffer Time**: Include 25-35% buffer for unexpected complexity and edge cases
3. **Parallel Development**: Some phases can be developed in parallel by multiple developers
4. **Testing Time**: Allocate additional 2-3 weeks for production hardening and edge case handling
5. **Code Review**: Add 10-15% additional time for thorough code reviews

## Phase 1: Core Architecture Refactoring (Foundation)

### 1.1 Instance Registry Module
- [ ] Create `lua/claudecode/instance_registry.lua`
  - [ ] Define instance data structure with fields: `id`, `port`, `cwd`, `name`, `state`, `created_at`
  - [ ] Implement registry storage using table with instance IDs as keys
  - [ ] Add methods: `register()`, `unregister()`, `get()`, `get_all()`, `get_active()`
  - [ ] Add instance lifecycle tracking (starting, running, stopped, error)
  - [ ] Write unit tests for all registry methods

### 1.2 Instance Manager Module  
- [ ] Create `lua/claudecode/instance_manager.lua`
  - [ ] Implement `create_instance(opts)` that returns instance ID
  - [ ] Implement `destroy_instance(id)` with cleanup
  - [ ] Add `switch_active_instance(id)` method
  - [ ] Add `get_instance_by_cwd(cwd)` for worktree detection
  - [ ] Implement instance state validation
  - [ ] Write unit tests for instance lifecycle

### 1.3 Refactor Global State
- [ ] Modify `lua/claudecode/init.lua` to remove singleton `M.state`
- [ ] Replace with `M.instances = {}` table
- [ ] Update all state access to use instance-specific state
- [ ] Add `M.active_instance_id` for tracking current instance
- [ ] Ensure backward compatibility with single-instance usage
- [ ] Update all existing tests to work with new structure

## Phase 2: Multi-Port Server Management

### 2.1 Server Factory Pattern
- [ ] Create `lua/claudecode/server/factory.lua`
  - [ ] Implement `create_server(instance_id, port)` method
  - [ ] Add server-to-instance mapping
  - [ ] Handle port allocation per instance
  - [ ] Implement server cleanup on instance destruction
  - [ ] Write tests for concurrent server creation

### 2.2 Port Management Enhancement
- [ ] Modify `lua/claudecode/server/tcp.lua`
  - [ ] Add port reservation system to prevent conflicts
  - [ ] Track used ports across all instances
  - [ ] Implement `release_port(port)` method
  - [ ] Add port range partitioning for instances
  - [ ] Test concurrent port allocation

### 2.3 Lock File Per Instance
- [ ] Update `lua/claudecode/lockfile.lua`
  - [ ] Generate unique lock file per instance: `~/.claude/ide/[port]-[instance_id].lock`
  - [ ] Add instance metadata to lock file content
  - [ ] Implement lock file cleanup on instance destruction
  - [ ] Add lock file discovery for existing instances
  - [ ] Test multiple lock file scenarios

## Phase 3: Terminal Management Enhancement

### 3.1 Multi-Terminal Support
- [ ] Refactor `lua/claudecode/terminal.lua`
  - [ ] Remove global terminal state
  - [ ] Add terminal-to-instance mapping
  - [ ] Implement `get_terminal_for_instance(id)`
  - [ ] Update toggle logic to work with active instance
  - [ ] Preserve terminal state per instance

### 3.2 Terminal Provider Updates
- [ ] Update `lua/claudecode/terminal/native.lua`
  - [ ] Support multiple terminal buffers
  - [ ] Add instance ID to terminal buffer variables
  - [ ] Implement terminal switching logic
  - [ ] Handle concurrent terminal operations
  
- [ ] Update `lua/claudecode/terminal/snacks.lua`
  - [ ] Adapt snacks integration for multiple terminals
  - [ ] Ensure proper terminal identification
  - [ ] Test with multiple snacks terminals

### 3.3 Window Management
- [ ] Create `lua/claudecode/windows.lua`
  - [ ] Implement window layout for multiple instances
  - [ ] Add tabbed interface option
  - [ ] Add split window arrangement option
  - [ ] Implement instance switcher UI
  - [ ] Handle window cleanup on instance close

## Phase 4: Command System Enhancement

### 4.1 Instance-Aware Commands
- [ ] Update all existing commands in `lua/claudecode/init.lua`
  - [ ] Add optional `--instance-id` parameter to all commands
  - [ ] Default to active instance when no ID specified
  - [ ] Add instance validation to each command
  - [ ] Update command completion to suggest instance IDs

### 4.2 New Multi-Instance Commands
- [ ] Implement `:ClaudeCodeNew [--cwd path] [--name name]`
  - [ ] Create new instance with specified working directory
  - [ ] Auto-generate instance name if not provided
  - [ ] Return instance ID on creation
  
- [ ] Implement `:ClaudeCodeList`
  - [ ] Show all instances with status, port, cwd
  - [ ] Highlight active instance
  - [ ] Add instance health indicators
  
- [ ] Implement `:ClaudeCodeSwitch <instance-id|name>`
  - [ ] Switch active instance
  - [ ] Update UI to reflect switch
  - [ ] Trigger switch notification
  
- [ ] Implement `:ClaudeCodeKill <instance-id|name|all>`
  - [ ] Gracefully shutdown specified instance(s)
  - [ ] Clean up all resources
  - [ ] Handle 'all' keyword for bulk operations

### 4.3 Selection and Context Handling
- [ ] Update `lua/claudecode/selection.lua`
  - [ ] Track selections per instance
  - [ ] Update selection sync to use active instance
  - [ ] Handle instance switching during selection
  
- [ ] Update file sending logic
  - [ ] Route file additions to correct instance
  - [ ] Support sending files to specific instance
  - [ ] Add cross-instance file sharing option

## Phase 5: UI and Notifications

### 5.1 Instance Switcher UI
- [ ] Create `lua/claudecode/ui/switcher.lua`
  - [ ] Implement floating window instance picker
  - [ ] Show instance details (name, cwd, status)
  - [ ] Add keyboard navigation
  - [ ] Support mouse clicks
  - [ ] Add instance preview

### 5.2 Status Line Integration
- [ ] Create `lua/claudecode/ui/statusline.lua`
  - [ ] Provide statusline component showing active instance
  - [ ] Add instance count indicator
  - [ ] Show instance health status
  - [ ] Support popular statusline plugins

### 5.3 Notifications System
- [ ] Enhance `lua/claudecode/logger.lua`
  - [ ] Add instance-specific notifications
  - [ ] Implement instance switch notifications
  - [ ] Add instance error notifications
  - [ ] Support notification routing to active instance

## Phase 6: Advanced Features

### 6.1 Worktree Integration
- [ ] Create `lua/claudecode/worktree.lua`
  - [ ] Auto-detect git worktrees
  - [ ] Create instance per worktree automatically
  - [ ] Map worktree to instance
  - [ ] Handle worktree switching
  - [ ] Clean up on worktree deletion

### 6.2 Instance Persistence
- [ ] Create `lua/claudecode/persistence.lua`
  - [ ] Save instance state to disk
  - [ ] Restore instances on Neovim restart
  - [ ] Handle stale instance cleanup
  - [ ] Implement session management

### 6.3 Network Features
- [ ] Implement fetch() proxy support
  - [ ] Add fetch command routing through Claude
  - [ ] Support both web and intranet (when available)
  - [ ] Handle CORS and security
  - [ ] Add request/response logging

## Phase 7: Testing and Documentation

### 7.1 Comprehensive Testing
- [ ] Update all existing tests for multi-instance
- [ ] Add integration tests for multiple instances
- [ ] Test instance switching scenarios
- [ ] Test resource cleanup
- [ ] Add stress tests for many instances
- [ ] Test worktree integration

### 7.2 Documentation Updates
- [ ] Update README.md with multi-instance usage
- [ ] Document all new commands
- [ ] Add multi-instance configuration examples
- [ ] Create troubleshooting guide
- [ ] Add architecture diagrams

### 7.3 Migration Guide
- [ ] Create migration guide for existing users
- [ ] Document breaking changes
- [ ] Provide configuration migration tool
- [ ] Add compatibility layer for old commands

## Success Metrics

### Measurable Outcomes
1. **Instance Management**
   - Support minimum 10 concurrent instances
   - Instance creation time < 500ms
   - Instance switching time < 100ms
   - Zero port conflicts in 1000 iterations

2. **Resource Usage**
   - Memory per instance < 50MB
   - No memory leaks after 100 create/destroy cycles
   - Clean process termination 100% of time

3. **User Experience**
   - All commands respond < 200ms
   - Status updates within 50ms
   - Smooth UI transitions
   - No blocking operations

4. **Compatibility**
   - 100% backward compatibility for single instance
   - All existing tests pass
   - Works with Neovim 0.8+
   - Works with both terminal providers

## Implementation Priority

1. **High Priority** (Core Functionality)
   - Phase 1: Architecture Refactoring
   - Phase 2: Multi-Port Management
   - Phase 4.1-4.2: Basic Commands

2. **Medium Priority** (Enhanced UX)
   - Phase 3: Terminal Management
   - Phase 5: UI and Notifications
   - Phase 6.1: Worktree Integration

3. **Low Priority** (Nice to Have)
   - Phase 6.2: Persistence
   - Phase 6.3: Network Features
   - Phase 5.1: Advanced UI

## Risk Mitigation

1. **Breaking Changes**
   - Maintain compatibility layer
   - Deprecate gradually
   - Provide migration tools

2. **Performance Impact**
   - Profile before/after
   - Lazy load instances
   - Implement resource limits

3. **Complexity Growth**
   - Modular architecture
   - Clear separation of concerns
   - Comprehensive documentation