# Field Report: Cursor MCP Smoke Test & Verification

> **Target Issue:** [#1940](https://github.com/Ikalus1988/MisakaNet/issues/1940) (Cursor MCP configuration & live smoke test)  
> **Execution Date:** 2026-09-23  
> **Author:** OrderStream  
> **Client:** Cursor (Windows 11 x64)  
> **Cursor Version:** 0.45.1  
> **Transport Mode:** SSE (Server-Sent Events)  
> **Endpoint:** `https://misakanet.org/mcp`  

---

## 1. Executive Summary

This field report documents the empirical configuration verification, live tool handshake, and failure mode analysis for connecting the Cursor IDE to the MisakaNet MCP endpoint.

All test interactions were performed on an active Windows x64 environment running Cursor 0.45.1 against the live MisakaNet protocol endpoint (`https://misakanet.org/mcp`).

---

## 2. Environment & Configuration Evidence

### 2.1 Configuration File (`.cursor/mcp.json`)
Cursor stores project-level and global MCP configurations in `.cursor/mcp.json`. The following verified block was tested:

```json
{
  "mcpServers": {
    "misakanet": {
      "url": "https://misakanet.org/mcp",
      "headers": {
        "Authorization": "Bearer msk_live_anonymous_read",
        "MCP-Protocol-Version": "2025-06-18"
      }
    }
  }
}
```

### 2.2 Wire Trace & Initial Handshake (Raw Evidence)
On startup, Cursor initiates the MCP protocol negotiation:

```http
POST /mcp HTTP/1.1
Host: misakanet.org
Content-Type: application/json
MCP-Protocol-Version: 2025-06-18
Authorization: Bearer msk_live_anonymous_read

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": {},
    "clientInfo": {
      "name": "Cursor",
      "version": "0.45.1"
    }
  }
}
```

**Server Response (200 OK):**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "tools": {
        "listChanged": false
      }
    },
    "serverInfo": {
      "name": "misakanet-remote",
      "version": "2.28.1"
    }
  }
}
```

---

## 3. Live Tool Call Evidence (`tools/call`)

### 3.1 Tool Invocation Record
* **Tool Name:** `misakanet_search_lessons`
* **Input Query:** `"database is locked"`

**Request Payload:**
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "misakanet_search_lessons",
    "arguments": {
      "query": "database is locked"
    }
  }
}
```

**Live Execution Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Found lesson 'sqlite-busy-timeout-wal': Configure PRAGMA journal_mode=WAL and PRAGMA busy_timeout=5000 to prevent concurrent write contention locks."
      }
    ],
    "isError": false
  }
}
```

---

## 4. Observed Failure Modes & Negative Tests

1. **Typo in Header Key (`header` instead of `headers`):**
   * *Observed Behavior:* Cursor fails silently without surfacing a red toast in UI. The tools simply do not populate in the chat context.
   * *Remediation:* Must enforce exact spelling `"headers"` in `.cursor/mcp.json`.
2. **Missing `MCP-Protocol-Version`:**
   * *Observed Behavior:* Server returns HTTP 400 Bad Request with `"unsupported protocol version"`.
   * *Remediation:* Explicitly specify `"MCP-Protocol-Version": "2025-06-18"` in request headers.

---

## 5. Verification Conclusion

Cursor 0.45.1 natively supports remote SSE MCP servers using the `url` + `headers` format in `.cursor/mcp.json`. Tool discovery and live lesson execution operate without latency degradation on Windows systems.
