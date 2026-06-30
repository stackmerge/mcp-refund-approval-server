# MCP Refund Approval Server

A **MuleSoft 4** application that exposes a **Model Context Protocol (MCP)** tool for AI agents to process customer refund requests with **human-in-the-loop approval** for high-value transactions. Built on the MuleSoft MCP Connector using Streamable HTTP transport.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Configuration](#configuration)
- [Flow Design](#flow-design)
- [Decision Logic](#decision-logic)
- [MCP Tool Reference](#mcp-tool-reference)
- [Elicitation (Human-in-the-Loop)](#elicitation-human-in-the-loop)
- [Error Handling](#error-handling)
- [Running the Application](#running-the-application)
- [API Reference](#api-reference)
- [Sample Payloads](#sample-payloads)
- [Response Schemas](#response-schemas)
- [Deployment](#deployment)
- [Dependencies](#dependencies)

---

## Overview

The **MCP Refund Approval Server** is a Mule application that bridges AI agent systems and human reviewers for refund processing. It exposes a single MCP tool (`issue_refund`) that:

1. **Validates** refund eligibility and checks for duplicates.
2. **Assesses risk** based on transaction amount and flags.
3. **Auto-approves** standard low-value refunds (≤ 50,000 units) and immediately calls the downstream Refund API.
4. **Pauses for human approval** on high-value (> 50,000 units) or high-risk refunds, sending an elicitation form to the MCP client.
5. **Executes or rejects** based on the human reviewer's decision.
6. **Fails closed** — if approval times out, reviewer declines, or an unexpected error occurs, no refund is processed.

### Key Capabilities

| Feature | Detail |
|---|---|
| **Protocol** | Model Context Protocol (MCP) over Streamable HTTP |
| **Transport** | HTTP POST + Server-Sent Events (SSE) |
| **Human-in-the-loop** | MCP Elicitation for high-value refunds |
| **Approval Threshold** | 50,000 units (configurable) |
| **Risk Detection** | Automatic flag for amounts > 100,000 |
| **Timeout** | 120 seconds elicitation window |
| **Fail-safe** | Fails closed — no refund without explicit APPROVE |

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                    AI Agent / MCP Client                           │
│           (Claude, Copilot, Custom AI, Postman)                    │
└───────────────────────────┬────────────────────────────────────────┘
                            │  MCP Streamable HTTP
                            │  POST http://localhost:8081/mcp
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│               MCP Refund Approval Server (Mule 4)                  │
│                                                                    │
│  ┌─────────────┐   ┌─────────────┐   ┌────────────────────────┐    │
│  │  Stage 1    │   │  Stage 2    │   │  Stage 3 (conditional) │    │
│  │  MCP Tool   │──▶│  Validate & │──▶│  Human Approval        │    │
│  │  Listener   │   │  Risk Check │   │  Elicitation           │    │
│  │ issue_refund│   │             │   │  (amount > 50,000)     │    │
│  └─────────────┘   └─────────────┘   └────────────────────────┘    │
│                                                 │                  │
│                    ┌─────────────┐              │                  │
│                    │  Stage 4    │◀─────────────┘                  │
│                    │  Execute or │                                 │
│                    │  Deny Refund│                                 │
│                    └──────┬──────┘                                 │
└───────────────────────────┼────────────────────────────────────────┘
                            │  HTTPS POST /api/v1/refunds
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│                    Downstream Refund API                           │
│              your-refund-api-host.example.com                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

| Requirement | Version |
|---|---|
| **Mule Runtime** | 4.11.0 or higher (4.11.2 recommended) |
| **Java JDK** | 17 |
| **Maven** | 3.8+ |
| **Anypoint Studio** | 7.x (optional, for IDE development) |
| **MuleSoft MCP Connector** | 1.6.0 |
| **MuleSoft HTTP Connector** | 1.11.3 |
| **Anypoint Platform Access** | Required for Maven dependency resolution |

### Maven Settings

Ensure your `~/.m2/settings.xml` includes credentials for the MuleSoft repositories:

```xml
<servers>
  <server>
    <id>anypoint-exchange-v2</id>
    <username>YOUR_ANYPOINT_USERNAME</username>
    <password>YOUR_ANYPOINT_PASSWORD</password>
  </server>
  <server>
    <id>mulesoft-releases</id>
    <username>YOUR_ANYPOINT_USERNAME</username>
    <password>YOUR_ANYPOINT_PASSWORD</password>
  </server>
</servers>
```

---

## Project Structure

```
mcp-refund-approval-server/
├── src/
│   └── main/
│       ├── mule/
│       │   └── mcp-refund-approval-server.xml    ← Main Mule flow
│       └── resources/
│           └── config.yaml                        ← Application configuration
├── pom.xml                                        ← Maven build descriptor
├── mule-artifact.json                             ← Mule runtime metadata
└── README.md
```

---

## Configuration

All configuration is externalized in `src/main/resources/config.yaml`:

```yaml
http:
  host: "0.0.0.0"          # Bind address (0.0.0.0 = all interfaces)
  port: "8081"              # HTTP listener port

mcp:
  serverName: "Refund Approval MCP Server"
  serverVersion: "1.0.0"
  endpointPath: "/mcp"                    # MCP endpoint path
  elicitationTimeoutSeconds: "120"        # Human approval timeout (seconds)

refundApi:
  host: "your-refund-api-host.example.com"  # ← UPDATE BEFORE RUNNING
  port: "443"
  protocol: "HTTPS"
  basePath: "/api/v1"
  refundsPath: "/refunds"

policy:
  approvalThresholdAmount: "50000"       # Threshold for human approval
```

### Configuration Reference

| Property | Description | Default |
|---|---|---|
| `http.host` | HTTP listener bind address | `0.0.0.0` |
| `http.port` | HTTP listener port | `8081` |
| `mcp.serverName` | MCP server display name | `Refund Approval MCP Server` |
| `mcp.serverVersion` | MCP server version string | `1.0.0` |
| `mcp.endpointPath` | URL path for the MCP endpoint | `/mcp` |
| `mcp.elicitationTimeoutSeconds` | Seconds to wait for human decision | `120` |
| `refundApi.host` | Downstream refund API hostname | *(must be set)* |
| `refundApi.port` | Downstream refund API port | `443` |
| `refundApi.protocol` | HTTP or HTTPS | `HTTPS` |
| `refundApi.basePath` | Base path for refund API | `/api/v1` |
| `refundApi.refundsPath` | Path for refund endpoint | `/refunds` |
| `policy.approvalThresholdAmount` | Amount above which human approval is required | `50000` |

> **Note**: `policy.approvalThresholdAmount` is stored in config.yaml for reference but the threshold is currently hardcoded in the DataWeave transform (`amount > 50000`). To make it fully dynamic, reference `${policy.approvalThresholdAmount}` inside the transform.

---

## Flow Design

The application contains a single Mule flow: **`issue-refund-flow`**

### Stage-by-Stage Walkthrough

#### Stage 1 — Receive Tool Call
- The `mcp:tool-listener` named `issue_refund` receives calls from MCP clients.
- The MCP session ID is stored in `vars.mcpSessionId` for later elicitation routing.
- The raw payload is stored in `vars.refundRequest` for error-handler fallback.

#### Stage 2 — Validate and Compute Approval Flag
A DataWeave 2.0 transform computes the following fields:

| Field | Logic |
|---|---|
| `eligibilityCheck` | Always `PASS` (stub — connect to eligibility service) |
| `duplicateCheck` | Always `NONE` (stub — connect to order service) |
| `amountThreshold` | `EXCEEDED` if `amount > 50000`, else `WITHIN_LIMIT` |
| `riskLevel` | `HIGH-VALUE REFUND` if `amount > 100000`, else `STANDARD` |
| `riskFlag` | `true` if `amount > 100000` |
| `policyCheckResult` | `PASS`, `FAIL_INELIGIBLE`, or `FAIL_DUPLICATE` |
| `requiresApproval` | `true` if `amount > 50000` OR `riskFlag == true` |

#### Stage 3 — Human Approval Elicitation (Conditional)
Triggered only when `requiresApproval == true`. The `mcp:create-elicitation` component:
- Sends a form to the MCP client with order details, amount, risk level, and policy result.
- Presents two fields: `decision` (enum: APPROVE / REJECT) and `reviewerReason` (string).
- Blocks the flow for up to `elicitationTimeoutSeconds` (120s) waiting for a response.
- Returns an `elicitationResult` with `action` (ACCEPT, DECLINE, CANCEL) and `content`.

#### Stage 4 — Execute or Deny
| Elicitation Result | Outcome |
|---|---|
| `action=ACCEPT` + `decision=APPROVE` | Calls downstream Refund API — refund **executed** |
| `action=ACCEPT` + `decision=REJECT` | Records rejection — refund **NOT executed** |
| `action=DECLINE` or other | Fails closed — refund **NOT executed** |
| Timeout (`MCP:REQUEST_TIMEOUT`) | Fails closed — refund **NOT executed** |

For low-value refunds (no elicitation path):
- Directly calls the downstream Refund API with `approvalDecision: "AUTO_APPROVED"`.

---

## Decision Logic

### Approval Routing Matrix

| Amount | Risk Flag | Policy Result | Path | Refund Executed? |
|---|---|---|---|---|
| ≤ 50,000 | false | PASS | Auto-approve (direct API call) | ✅ Yes |
| > 50,000 | false | PASS | Human elicitation required | ✅ Only on APPROVE |
| > 100,000 | true | PASS | Human elicitation required (HIGH-VALUE) | ✅ Only on APPROVE |
| any | any | FAIL_INELIGIBLE | Denied immediately | ❌ No |
| any | any | FAIL_DUPLICATE | Denied immediately | ❌ No |

### Fail-Closed Principle

This application **always fails closed**. A refund is executed **only** if:
- Policy check = `PASS`, AND
- Either `requiresApproval == false` (auto-approve), OR
- `requiresApproval == true` AND reviewer submits `action=ACCEPT` AND `decision=APPROVE`

Any other outcome — timeout, decline, cancel, REJECT, or error — results in **no refund**.

---

## MCP Tool Reference

### Tool: `issue_refund`

**Description**: Issue a refund for a customer order. Validates eligibility, checks duplicates, assesses risk, and requires human approval for amounts over 50,000 units.

#### Input Schema

```json
{
  "type": "object",
  "properties": {
    "orderId": {
      "type": "string",
      "description": "Order ID (e.g. ORD-10482)"
    },
    "amount": {
      "type": "number",
      "description": "Amount in smallest unit (paise for INR)",
      "minimum": 1
    },
    "currency": {
      "type": "string",
      "description": "ISO 4217 currency code e.g. INR",
      "pattern": "^[A-Z]{3}$"
    },
    "reason": {
      "type": "string",
      "description": "Refund reason",
      "minLength": 5,
      "maxLength": 500
    }
  },
  "required": ["orderId", "amount", "currency", "reason"]
}
```

#### Input Parameter Details

| Parameter | Type | Required | Constraints | Example |
|---|---|---|---|---|
| `orderId` | string | ✅ | Any non-empty string | `"ORD-10482"` |
| `amount` | number | ✅ | Minimum: 1 (smallest unit) | `25000` (₹250.00) |
| `currency` | string | ✅ | ISO 4217, exactly 3 uppercase letters | `"INR"` |
| `reason` | string | ✅ | 5–500 characters | `"Item not delivered"` |

> **Amount Note**: For INR, amounts are in paise. `25000` = ₹250.00. `100000` = ₹1,000.00.

#### Success Response (Auto-Approve — amount ≤ 50,000)

```json
{
  "status": "SUCCESS",
  "message": "Refund of INR 25000 was auto-approved and successfully issued.",
  "orderId": "ORD-10482",
  "refundId": "corr-abc-123",
  "correlationId": "corr-abc-123"
}
```

#### Success Response (Human-Approved — amount > 50,000)

```json
{
  "status": "SUCCESS",
  "message": "Refund of INR 75000 was approved and successfully issued.",
  "orderId": "ORD-20891",
  "refundId": "corr-def-456",
  "reviewerReason": "Verified by finance team. Customer escalation.",
  "correlationId": "corr-def-456"
}
```

#### Denial Response (Policy Failure)

```text
Refund denied. Policy: FAIL_INELIGIBLE. Order: ORD-99999
```

#### Rejection Response (Reviewer Rejected)

```text
Refund rejected by reviewer. Order: ORD-20891. Reviewer reason: Duplicate order detected by reviewer.
```

#### Decline Response (Reviewer Declined Form)

```text
Refund not executed. Reviewer action: DECLINE. Order: ORD-20891. No refund was processed.
```

#### Timeout Response

```text
Refund approval timed out. Order: ORD-20891. No refund was processed. Please retry or escalate.
```

---

## Elicitation (Human-in-the-Loop)

When a refund requires human approval, the server uses **MCP Elicitation** to pause the flow and request a decision from the human reviewer via the MCP client interface.

### Elicitation Message Format

The elicitation message shown to the reviewer:

```
Refund Approval Required | Order: ORD-20891 | Amount: INR 75000 | 
Risk: STANDARD | Policy: PASS | Reason: Customer did not receive item
```

### Elicitation Form Fields

| Field | Type | Required | Options |
|---|---|---|---|
| `decision` | enum | ✅ | `APPROVE` or `REJECT` |
| `reviewerReason` | string | ✅ | Free-text explanation |

### Elicitation Actions

The MCP client can respond with one of three actions:

| Action | Description | Refund Outcome |
|---|---|---|
| `ACCEPT` | Reviewer submitted the form | Depends on `decision` field |
| `DECLINE` | Reviewer explicitly dismissed the form | Refund NOT executed (fail closed) |
| `CANCEL` | Operation cancelled (triggers `MCP:OPERATION_CANCELLED`) | Refund NOT executed (error handler) |

### Elicitation Timeout

If no response is received within `elicitationTimeoutSeconds` (default: 120 seconds), an `MCP:REQUEST_TIMEOUT` error is raised and the flow fails closed — **no refund is issued**.

---

## Error Handling

The flow includes a comprehensive error handler covering all failure scenarios:

| Error Type | Trigger | Outcome |
|---|---|---|
| `MCP:REQUEST_TIMEOUT` | Elicitation not responded to within 120s | Fail closed, no refund. Message: "Refund approval timed out." |
| `HTTP:CONNECTIVITY` | Downstream Refund API unreachable | Fail closed, no refund. Message includes error details. |
| `MULE:ANY` | Any other error, including `MCP:OPERATION_CANCELLED` | Fail closed, no refund. Full error description returned. |

All errors are:
- Logged at `ERROR` level with the order ID and error description.
- Returned to the MCP client as a descriptive text message via `mcp:on-error-responses`.
- **Never** result in a refund being executed.

---

## Running the Application

### Option 1: Anypoint Studio

1. Open Anypoint Studio.
2. Go to **File → Import → Anypoint Studio → Anypoint Studio project from File System**.
3. Select the `mcp-refund-approval-server` folder.
4. Update `config.yaml` with your downstream Refund API details.
5. Right-click the project → **Run As → Mule Application**.

### Option 2: Maven Command Line

```bash
# Clone/navigate to project directory
cd mcp-refund-approval-server

# Build the project
mvn clean package

# Run locally (requires local Mule runtime)
mvn mule:run
```

### Option 3: Deploy Packaged JAR

```bash
# Build the deployable JAR
mvn clean package -DskipTests

# The JAR is created at:
# target/mcp-refund-approval-server-1.0.0-SNAPSHOT-mule-application.jar
```

### Verify the Server is Running

```bash
# Check HTTP listener (should return MCP protocol response)
curl -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": { "name": "TestClient", "version": "1.0" }
    }
  }'
```

Expected response contains `"result"` with `serverInfo.name: "Refund Approval MCP Server"`.

---

## API Reference

### Endpoint

```
POST http://localhost:8081/mcp
```

### Required Headers

| Header | Value | Description |
|---|---|---|
| `Content-Type` | `application/json` | Request body format |
| `Accept` | `application/json` or `text/event-stream` | Use SSE for tool calls with elicitation |
| `mcp-session-id` | `<session-id>` | Required after initialize; returned in response headers |

### MCP Protocol Flow

The MCP Streamable HTTP transport follows this sequence:

```
Client                                        Server
  │                                              │
  │── POST /mcp  (method: initialize) ──────────▶│
  │◀── 200 OK  {result: {serverInfo, ...}}  ─────│
  │              [mcp-session-id header set]      │
  │                                              │
  │── POST /mcp  (method: tools/list)  ──────────▶│
  │◀── 200 OK  {result: {tools: [issue_refund]}} ─│
  │                                              │
  │── POST /mcp  (method: tools/call)  ──────────▶│
  │   Accept: text/event-stream                  │
  │   [For high-value: server sends elicitation] │
  │◀── SSE event: elicitation request ───────────│
  │                                              │
  │── POST /mcp  (elicitations/respond)  ────────▶│
  │◀── SSE event: tool result ───────────────────│
```

---

## Sample Payloads

### 1. Initialize Session

```json
POST http://localhost:8081/mcp
Content-Type: application/json
Accept: application/json

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "elicitation": {}
    },
    "clientInfo": {
      "name": "PostmanMCPClient",
      "version": "1.0.0"
    }
  }
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {},
      "elicitation": {}
    },
    "serverInfo": {
      "name": "Refund Approval MCP Server",
      "version": "1.0.0"
    }
  }
}
```

---

### 2. List Available Tools

```json
POST http://localhost:8081/mcp
Content-Type: application/json
Accept: application/json
mcp-session-id: {{sessionId}}

{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list",
  "params": {}
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "issue_refund",
        "description": "Issue a refund for a customer order. Validates eligibility, checks duplicates, assesses risk, and requires human approval for amounts over 50,000 units.",
        "inputSchema": {
          "type": "object",
          "properties": {
            "orderId": { "type": "string", "description": "Order ID (e.g. ORD-10482)" },
            "amount": { "type": "number", "description": "Amount in smallest unit (paise for INR)", "minimum": 1 },
            "currency": { "type": "string", "description": "ISO 4217 code e.g. INR", "pattern": "^[A-Z]{3}$" },
            "reason": { "type": "string", "description": "Refund reason", "minLength": 5, "maxLength": 500 }
          },
          "required": ["orderId", "amount", "currency", "reason"]
        }
      }
    ]
  }
}
```

---

### 3. Standard Refund — Auto-Approved (amount ≤ 50,000)

```json
POST http://localhost:8081/mcp
Content-Type: application/json
Accept: application/json
mcp-session-id: {{sessionId}}

{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "issue_refund",
    "arguments": {
      "orderId": "ORD-10482",
      "amount": 25000,
      "currency": "INR",
      "reason": "Item was not delivered to the customer address"
    }
  }
}
```

**Response (Success — Auto-Approved):**
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"status\":\"SUCCESS\",\"message\":\"Refund of INR 25000 was auto-approved and successfully issued.\",\"orderId\":\"ORD-10482\",\"refundId\":\"corr-abc-123\",\"correlationId\":\"corr-abc-123\"}"
      }
    ],
    "isError": false
  }
}
```

---

### 4. High-Value Refund — Requires Human Approval (amount > 50,000)

```json
POST http://localhost:8081/mcp
Content-Type: application/json
Accept: text/event-stream
mcp-session-id: {{sessionId}}

{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "tools/call",
  "params": {
    "name": "issue_refund",
    "arguments": {
      "orderId": "ORD-20891",
      "amount": 75000,
      "currency": "INR",
      "reason": "Customer received a damaged product and requested full refund after escalation"
    }
  }
}
```

**SSE Event Response (Elicitation Request):**
```
event: message
data: {
  "jsonrpc": "2.0",
  "id": "elicit-001",
  "method": "elicitations/create",
  "params": {
    "message": "Refund Approval Required | Order: ORD-20891 | Amount: INR 75000 | Risk: STANDARD | Policy: PASS | Reason: Customer received a damaged product and requested full refund after escalation",
    "requestedSchema": {
      "type": "object",
      "properties": {
        "decision": {
          "type": "string",
          "enum": ["APPROVE", "REJECT"],
          "description": "Approve or reject this refund"
        },
        "reviewerReason": {
          "type": "string",
          "description": "Reason for your decision"
        }
      },
      "required": ["decision", "reviewerReason"]
    }
  }
}
```

---

### 5. Submit Elicitation Response — APPROVE

```json
POST http://localhost:8081/mcp
Content-Type: application/json
Accept: text/event-stream
mcp-session-id: {{sessionId}}

{
  "jsonrpc": "2.0",
  "id": "elicit-001",
  "result": {
    "action": "accept",
    "content": {
      "decision": "APPROVE",
      "reviewerReason": "Verified with warehouse team. Product confirmed damaged in transit. Refund authorized."
    }
  }
}
```

**Final SSE Tool Result:**
```
event: message
data: {
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"status\":\"SUCCESS\",\"message\":\"Refund of INR 75000 was approved and successfully issued.\",\"orderId\":\"ORD-20891\",\"refundId\":\"corr-def-456\",\"reviewerReason\":\"Verified with warehouse team. Product confirmed damaged in transit. Refund authorized.\",\"correlationId\":\"corr-def-456\"}"
      }
    ],
    "isError": false
  }
}
```

---

### 6. Submit Elicitation Response — REJECT

```json
POST http://localhost:8081/mcp
Content-Type: application/json
Accept: text/event-stream
mcp-session-id: {{sessionId}}

{
  "jsonrpc": "2.0",
  "id": "elicit-001",
  "result": {
    "action": "accept",
    "content": {
      "decision": "REJECT",
      "reviewerReason": "Order already refunded via separate channel. Duplicate refund request rejected."
    }
  }
}
```

**Final SSE Tool Result:**
```
event: message
data: {
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Refund rejected by reviewer. Order: ORD-20891. Reviewer reason: Order already refunded via separate channel. Duplicate refund request rejected."
      }
    ],
    "isError": false
  }
}
```

---

### 7. Submit Elicitation Response — DECLINE (Reviewer Dismissed Form)

```json
POST http://localhost:8081/mcp
Content-Type: application/json
Accept: text/event-stream
mcp-session-id: {{sessionId}}

{
  "jsonrpc": "2.0",
  "id": "elicit-001",
  "result": {
    "action": "decline"
  }
}
```

**Final SSE Tool Result:**
```
event: message
data: {
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Refund not executed. Reviewer action: DECLINE. Order: ORD-20891. No refund was processed."
      }
    ],
    "isError": false
  }
}
```

---

### 8. High-Risk Refund — Above Risk Threshold (amount > 100,000)

```json
POST http://localhost:8081/mcp
Content-Type: application/json
Accept: text/event-stream
mcp-session-id: {{sessionId}}

{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "tools/call",
  "params": {
    "name": "issue_refund",
    "arguments": {
      "orderId": "ORD-55001",
      "amount": 150000,
      "currency": "INR",
      "reason": "Enterprise customer requesting bulk order cancellation and full refund"
    }
  }
}
```

> This triggers the `HIGH-VALUE REFUND` risk level (`riskFlag=true`) and always requires human approval via elicitation, regardless of any other condition.

---

## Response Schemas

### Downstream Refund API — Request Body (Auto-Approved)

```json
{
  "orderId": "ORD-10482",
  "amount": 25000,
  "currency": "INR",
  "reason": "Item was not delivered to the customer address",
  "approvalDecision": "AUTO_APPROVED",
  "correlationId": "mule-corr-id-abc123"
}
```

### Downstream Refund API — Request Body (Human-Approved)

```json
{
  "orderId": "ORD-20891",
  "amount": 75000,
  "currency": "INR",
  "reason": "Customer received a damaged product",
  "approvalDecision": "APPROVE",
  "reviewerReason": "Verified with warehouse team. Product confirmed damaged.",
  "correlationId": "mule-corr-id-def456"
}
```

### Validation Payload Structure (internal, `vars.validatedRefund`)

```json
{
  "orderId": "ORD-20891",
  "amount": 75000,
  "currency": "INR",
  "reason": "Customer received a damaged product",
  "validation": {
    "eligibilityCheck": "PASS",
    "duplicateCheck": "NONE",
    "amountThreshold": "EXCEEDED",
    "riskLevel": "STANDARD",
    "riskFlag": false,
    "policyCheckResult": "PASS"
  },
  "requiresApproval": true
}
```

---

## Deployment

### Deploy to CloudHub 2.0

1. Log in to [Anypoint Platform](https://anypoint.mulesoft.com).
2. Navigate to **Runtime Manager → Deploy Application**.
3. Upload `target/mcp-refund-approval-server-1.0.0-SNAPSHOT-mule-application.jar`.
4. Set environment properties:
   - `refundApi.host` = your production refund API hostname
   - `refundApi.port` = production port
   - `refundApi.protocol` = `HTTPS`
5. Set vCores and replicas as needed.

### Deploy to Runtime Fabric (RTF)

```bash
mvn deploy -DmuleDeploy \
  -Danypoint.username=YOUR_USERNAME \
  -Danypoint.password=YOUR_PASSWORD \
  -Dcloudhub.application.name=mcp-refund-approval-server \
  -Dcloudhub.environment=Production
```

### Docker (Local Testing)

```bash
# Build
docker build -t mcp-refund-approval-server:1.0.0 .

# Run with overridden config
docker run -p 8081:8081 \
  -e REFUND_API_HOST=localhost \
  -e REFUND_API_PORT=8080 \
  -e REFUND_API_PROTOCOL=HTTP \
  mcp-refund-approval-server:1.0.0
```

---

## Dependencies

### Runtime Dependencies

| Dependency | Group ID | Artifact ID | Version | Type |
|---|---|---|---|---|
| MuleSoft HTTP Connector | `org.mule.connectors` | `mule-http-connector` | `1.11.3` | mule-plugin |
| MuleSoft MCP Connector | `com.mulesoft.connectors` | `mule-mcp-connector` | `1.6.0` | mule-plugin |

### Build Dependencies

| Dependency | Group ID | Artifact ID | Version |
|---|---|---|---|
| Mule Maven Plugin | `org.mule.tools.maven` | `mule-maven-plugin` | `4.7.0` |
| Maven Clean Plugin | `org.apache.maven.plugins` | `maven-clean-plugin` | `3.0.0` |

### Runtime Requirement

| Property | Value |
|---|---|
| **Mule Runtime** | 4.11.0+ (tested on 4.11.2) |
| **Java** | 17 |
| **Min Mule Version** | `4.11.0` (declared in `mule-artifact.json`) |

### Maven Repositories

| Repository ID | URL |
|---|---|
| `anypoint-exchange-v2` | `https://maven.anypoint.mulesoft.com/api/v2/maven` |
| `mulesoft-releases` | `https://repository.mulesoft.org/releases/` |

---

## Extending the Application

### Integrating Real Eligibility Checks

Replace the stub variables in the DataWeave transform in **Stage 2**:

```dataweave
// Replace these stubs with real API calls:
var isEligible = true          // → call eligibility microservice
var existingRefund = false     // → query order management system
```

Add HTTP request calls before the transform to fetch live data and pass results into the transform via flow variables.

### Adding More Currencies

The `currency` field accepts any valid ISO 4217 3-letter code (e.g., `USD`, `EUR`, `GBP`, `INR`). No code changes needed — the regex pattern `^[A-Z]{3}$` validates format only.

### Adjusting the Approval Threshold

Change `amount > 50000` in the DataWeave transform to reference the config property:

```dataweave
var threshold = ${policy.approvalThresholdAmount} as Number
var requiresApproval: Boolean = (amount > threshold) or riskFlag
```

### Adding Audit Logging

The application already logs at `INFO`/`WARN`/`ERROR` levels. To integrate with an audit system, add an HTTP request or publish to a queue after each decision point.

---

## Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| `HTTP:CONNECTIVITY` on startup | `refundApi.host` not reachable | Update `config.yaml` with correct host/port |
| Elicitation not received in client | `Accept` header missing or wrong | Set `Accept: text/event-stream` for tool calls |
| Session ID error | `mcp-session-id` header not sent | Copy session ID from `initialize` response header |
| 120s timeout | Reviewer did not respond in time | Increase `mcp.elicitationTimeoutSeconds` in config.yaml |
| Maven build fails | Missing Anypoint credentials | Add credentials to `~/.m2/settings.xml` |
| Port 8081 already in use | Another process on same port | Change `http.port` in config.yaml |

---

## References

- [MuleSoft MCP Connector Documentation](https://docs.mulesoft.com/mcp-connector/latest/)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/specification)
- [MuleSoft Anypoint Platform](https://anypoint.mulesoft.com)
- [DataWeave 2.0 Language Reference](https://docs.mulesoft.com/dataweave/latest/)
- [Mule 4 Error Handling](https://docs.mulesoft.com/mule-runtime/latest/error-handling)
- [MuleSoft General Documentation](https://docs.mulesoft.com/general/)

---

*MCP Refund Approval Server v1.0.0 — Built with MuleSoft 4 and MCP Connector 1.6.0*
