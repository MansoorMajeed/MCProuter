# MCProuter Brainstorming

This document captures an AI-assisted (Cursor IDE - Claude-4-sonnet) architectural design session for solving MCP server horizontal scaling in Kubernetes.

## Project Overview
MCProuter is a reverse proxy/router for horizontally scaled MCP (Model Context Protocol) servers in Kubernetes environments.

## Problem Statement

### The Core Issue: Load Balancing & Routing
MCP assumes a **1:1 client-to-server relationship**, but Kubernetes creates:

- **Multiple server instances** behind a single service
- **No session affinity** by default - requests scatter across pods

```
Client → K8s Service → [Server Pod A, Server Pod B, Server Pod C]
         ↑
    This breaks MCP's session model
```

## Transport Decision
**Using**: Full Streamable HTTP (2025-03-26) protocol including SSE streaming support
**Not supporting**: Older HTTP+SSE specification (2024-11-05)

### Why Full Protocol Support:
- **Server-initiated messages**: Notifications, progress updates, live data
- **Streaming responses**: For long-running operations  
- **Interactive experiences**: Real-time tool execution feedback
- **Protocol compliance**: Clients expect complete Streamable HTTP support

## MCProuter Architecture

### Target Architecture
```
[Client] ←→ [Load Balancer] ←→ [MCProuter Instance A] ←→ [MCP Server Instance A]
                           ←→ [MCProuter Instance B] ←→ [MCP Server Instance B]  
                           ←→ [MCProuter Instance C] ←→ [MCP Server Instance C]
```

### Key Responsibilities

1. **Session Affinity**: Route all requests/streams with same session ID to same backend server

2. **Full Protocol Support**: Handle complete Streamable HTTP transport including:
   - HTTP POST requests with JSON-RPC messages
   - SSE stream proxying for real-time communication
   - Session management with `Mcp-Session-Id` headers

3. **Connection Management**: 
   - Proxy SSE streams between clients and backends
   - Manage HTTP request/response flows
   - Handle connection cleanup on session termination

4. **Load Balancing**: Intelligently distribute new client sessions across available backend servers

5. **Health Checking**: Monitor backend server health and route around failed instances

6. **Capability Routing**: Potentially route requests to servers based on their advertised capabilities

### Value Proposition
- **Clients** see a single, reliable MCP endpoint
- **Server operators** can horizontally scale MCP servers in K8s
- **DevOps** gets standard load balancing patterns for MCP workloads

## Horizontally Scaling MCProuter

### The Problem
MCProuter itself needs horizontal scaling, creating the same session affinity issue:

```
[Client] → [Load Balancer] → [MCProuter Instance A] → [Backend MCP Servers]
                          → [MCProuter Instance B] → [Backend MCP Servers]  
                          → [MCProuter Instance C] → [Backend MCP Servers]
```

### Solution: Consistent Hashing
Using `Mcp-Session-Id` headers enables consistent hashing for both HTTP requests and SSE streams:

#### Routing Strategy:
1. **New sessions** (no session ID): Route via round-robin
2. **Existing sessions** (with session ID): Route via consistent hashing on session ID
3. **Session mapping**: Each MCProuter instance maintains `session_id → backend_server` mappings

#### SSE Stream Handling:
- **Same session affinity**: SSE streams route to same MCProuter instance as HTTP requests
- **Stream proxying**: MCProuter maintains SSE connection between client and backend
- **Stream state**: Managed per session within each MCProuter instance

#### Failure Handling:
- If MCProuter instance dies, affected clients' HTTP requests and SSE streams fail
- Clients reinitialize and reconnect (acceptable trade-off)
- No shared state required between MCProuter instances

## Backend Service Discovery

### Dynamic Backend Management
MCProuter needs to discover and connect directly to individual MCP server pods instead of going through another Kubernetes load balancer layer.

### Kubernetes API Watch Strategy
**Approach**: MCProuter watches the Kubernetes API for MCP server pods with specific labels:

```go
// Watch for pods with specific labels
pods := clientset.CoreV1().Pods(namespace).Watch(ctx, metav1.ListOptions{
    LabelSelector: "app=mcp-server",
})

for event := range pods.ResultChan() {
    pod := event.Object.(*v1.Pod)
    if pod.Status.Phase == v1.PodRunning {
        // Add pod.Status.PodIP:port to backend pool
        backends.Add(pod.Status.PodIP + ":8080")
    }
}
```

### Benefits:
- **Dynamic scaling**: Automatically discovers new pods as they start
- **Health awareness**: Only routes to pods in `Running` state with readiness checks
- **Real-time updates**: Immediate backend pool updates on pod lifecycle events
- **Configuration-free**: No static backend configuration needed
- **Cloud-native**: Leverages Kubernetes primitives

### Requirements:
- **RBAC permissions**: MCProuter needs `get`, `list`, `watch` on pods
- **Pod labeling**: Backend MCP servers must have consistent labels
- **Health checks**: Integration with Kubernetes readiness probes

## Next Steps
- Design detailed architecture for session affinity
- Implement basic HTTP-only Streamable HTTP proxy
- Add consistent hashing for MCProuter horizontal scaling
- Add health checking and load balancing
- Consider capability-based routing 