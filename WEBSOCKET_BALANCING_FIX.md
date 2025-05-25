# WebSocket Load Balancing Fix for Apache APISIX

## Problem Description

This fix addresses the WebSocket load balancing imbalance issue described in GitHub issue #12217. The problem occurs when using the `least_conn` load balancing algorithm with WebSocket connections and upstream node scaling.

### Root Cause

When upstream nodes are scaled (added or removed), APISIX recreates the load balancer instance, which causes the `least_conn` balancer to lose its connection state. This results in:

1. **Connection count reset**: All servers start with 0 connections, ignoring existing WebSocket connections
2. **Load imbalance**: New connections are distributed unevenly because the balancer doesn't know about existing connections
3. **Poor user experience**: WebSocket connections may be concentrated on fewer servers

## Solution Overview

The fix implements **persistent connection tracking** using nginx shared dictionaries to maintain connection counts across balancer recreations.

### Key Components

1. **Shared Dictionary**: `balancer-least-conn` stores connection counts persistently
2. **Connection Tracking**: Increment/decrement counts on connection start/end
3. **State Restoration**: Initialize new balancer instances with existing connection counts
4. **Cleanup Mechanism**: Remove stale connection counts for removed servers

## Implementation Details

### 1. Configuration Changes

**File**: `conf/config.yaml`
```yaml
nginx_config:
  http:
    lua_shared_dict:
      balancer-least-conn: 10m  # Shared dictionary for connection tracking
```

### 2. Enhanced least_conn Balancer

**File**: `apisix/balancer/least_conn.lua`

#### Key Functions Added:

- `init_conn_count_dict()`: Initialize shared dictionary access
- `get_conn_count_key()`: Generate unique keys for upstream/server combinations
- `get_server_conn_count()`: Retrieve persisted connection count
- `incr_server_conn_count()`: Atomically update connection counts
- `cleanup_stale_conn_counts()`: Remove counts for deleted servers

#### Enhanced Workflow:

1. **Balancer Creation**: 
   - Clean up stale connection counts
   - Initialize servers with persisted connection counts
   - Calculate scores including existing connections

2. **Connection Selection**:
   - Select server with lowest score (as before)
   - Increment connection count in shared dictionary
   - Update local heap score

3. **Connection Completion**:
   - Decrement connection count in shared dictionary
   - Update local heap score

### 3. Connection Count Persistence

The implementation uses a hierarchical key structure:
```
conn_count:{upstream_id}:{server_address}
```

Where:
- `upstream_id`: Unique identifier for the upstream configuration
- `server_address`: Server IP:port combination

### 4. Atomic Operations

All shared dictionary operations use atomic increment/decrement to ensure consistency across worker processes.

## Benefits

1. **Persistent State**: Connection counts survive upstream configuration changes
2. **Load Balance**: Maintains even distribution during scaling operations
3. **WebSocket Friendly**: Properly handles long-lived connections
4. **Performance**: Minimal overhead using efficient shared dictionary operations
5. **Reliability**: Graceful degradation when shared dictionary is unavailable

## Backward Compatibility

- **Fully backward compatible**: Works with existing configurations
- **Graceful fallback**: Functions normally even if shared dictionary is not configured
- **No breaking changes**: Existing behavior preserved when shared dict is unavailable

## Testing

The fix includes comprehensive test coverage:

1. **Connection persistence across upstream changes**
2. **Proper cleanup of stale connection counts**
3. **Load balancing accuracy with mixed connection states**
4. **Graceful handling of missing shared dictionary**

## Configuration Example

```yaml
# In apisix.yaml or via Admin API
upstreams:
  - id: websocket_upstream
    type: least_conn
    nodes:
      "192.168.1.10:8080": 1
      "192.168.1.11:8080": 1
      "192.168.1.12:8080": 1
    scheme: http

routes:
  - id: websocket_route
    uri: /websocket/*
    upstream_id: websocket_upstream
    plugins:
      proxy-rewrite:
        headers:
          Upgrade: websocket
          Connection: upgrade
```

## Monitoring

The fix includes detailed logging for debugging:

- Server initialization with connection counts
- Connection count updates
- Stale connection cleanup operations
- Shared dictionary access warnings

## Performance Impact

- **Memory**: ~10MB shared dictionary (configurable)
- **CPU**: Minimal overhead for atomic operations
- **Latency**: No measurable impact on request processing

## Future Enhancements

1. **TTL Support**: Automatic cleanup of stale connections
2. **Metrics Export**: Integration with monitoring systems
3. **Configuration Tuning**: Dynamic shared dictionary sizing
4. **Health Check Integration**: Reset counts on server health changes

## Related Issues

- GitHub Issue #12217: WebSocket load balancing imbalance
- GitHub Issue #7301: Related load balancing concerns

This fix provides a robust solution for WebSocket load balancing in dynamic environments while maintaining full backward compatibility and minimal performance impact.