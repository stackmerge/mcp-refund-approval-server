# MCP Refund Approval Server

A **MuleSoft 4** application that exposes a **Model Context Protocol (MCP)** tool for AI agents to process customer refund requests with **human-in-the-loop approval** for high-value transactions.

Built on the MuleSoft MCP Connector using Streamable HTTP transport.

---
## YouTube Tutorial

Watch the complete walkthrough here:

[![Watch the YouTube video](https://img.youtube.com/vi/b84tCxFR3N0/maxresdefault.jpg)](https://www.youtube.com/watch?v=b84tCxFR3N0)
---

## Overview

The **MCP Refund Approval Server** is a Mule application that bridges AI agent systems and human reviewers for refund processing.

It exposes a single MCP tool named `issue_refund` that:

1. **Validates** refund eligibility and checks for duplicate refunds.
2. **Assesses risk** based on transaction amount and risk flags.
3. **Auto-approves** standard low-value refunds of `≤ 50,000` units.
4. **Pauses for human approval** for high-value or high-risk refunds.
5. **Executes or rejects** the refund based on the reviewer decision.
6. **Fails closed** if approval times out, the reviewer declines, or an unexpected error occurs.

### Key Capabilities

| Feature                | Detail                                                      |
| ---------------------- | ----------------------------------------------------------- |
| **Protocol**           | Model Context Protocol (MCP) over Streamable HTTP           |
| **Transport**          | HTTP POST and Server-Sent Events (SSE)                      |
| **Human-in-the-loop**  | MCP Elicitation for high-value refunds                      |
| **Approval Threshold** | `50,000` units, configurable                                |
| **Risk Detection**     | Automatic high-risk flag for amounts greater than `100,000` |
| **Timeout**            | `120` seconds elicitation window                            |
| **Fail-safe**          | No refund is issued without explicit approval               |

---

## Architecture

```text
┌────────────────────────────────────────────────────────────────────┐
│                    AI Agent / MCP Client                           │
│           (Claude, Copilot, Custom AI, Postman)                    │
└───────────────────────────┬────────────────────────────────────────┘
                            │ MCP Streamable HTTP
                            │ POST http://localhost:8081/mcp
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│               MCP Refund Approval Server (Mule 4)                  │
│                                                                    │
│  ┌─────────────┐   ┌─────────────┐   ┌────────────────────────┐    │
│  │  Stage 1    │   │  Stage 2    │   │  Stage 3 (Conditional) │    │
│  │  MCP Tool   │──▶│  Validate & │──▶│  Human Approval        │    │
│  │  Listener   │   │  Risk Check │   │  Elicitation           │    │
│  │ issue_refund│   │             │   │  (Amount > 50,000)     │    │
│  └─────────────┘   └─────────────┘   └────────────────────────┘    │
│                                                 │                  │
│                    ┌─────────────┐              │                  │
│                    │  Stage 4    │◀─────────────┘                  │
│                    │  Execute or │                                 │
│                    │  Deny Refund│                                 │
│                    └──────┬──────┘                                 │
└───────────────────────────┼────────────────────────────────────────┘
                            │ HTTPS POST /api/v1/refunds
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│                    Downstream Refund API                           │
│              your-refund-api-host.example.com                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

| Requirement                  | Version                                  |
| ---------------------------- | ---------------------------------------- |
| **Mule Runtime**             | 4.11.0 or higher, 4.11.2 recommended     |
| **Java JDK**                 | 17                                       |
| **Maven**                    | 3.8 or higher                            |
| **Anypoint Studio**          | 7.x, optional                            |
| **MuleSoft MCP Connector**   | 1.6.0                                    |
| **MuleSoft HTTP Connector**  | 1.11.3                                   |
| **Anypoint Platform Access** | Required for Maven dependency resolution |

---

## Project Structure

```text
mcp-refund-approval-server/
├── src/
│   └── main/
│       ├── mule/
│       │   └── mcp-refund-approval-server.xml    ← Main Mule flow
│       └── resources/
│           └── config.yaml                       ← Application configuration
├── pom.xml                                       ← Maven build descriptor
├── mule-artifact.json                            ← Mule runtime metadata
└── README.md
```

---

## Configuration

All runtime configuration is externalized in:

```text
src/main/resources/config.yaml
```

```yaml
http:
  host: "0.0.0.0"
  port: "8081"

mcp:
  serverName: "Refund Approval MCP Server"
  serverVersion: "1.0.0"
  endpointPath: "/mcp"
  elicitationTimeoutSeconds: "120"

refundApi:
  host: "your-refund-api-host.example.com" # Update before running
  port: "443"
  protocol: "HTTPS"
  basePath: "/api/v1"
  refundsPath: "/refunds"

policy:
  approvalThresholdAmount: "50000"
```

> Update `refundApi.host` with the hostname of your downstream refund API before starting the application.

---

## Flow Design

The application contains a primary Mule flow:

```text
issue-refund-flow
```

### Stage-by-Stage Walkthrough

#### Stage 1 — Receive Tool Call

* The `mcp:tool-listener` named `issue_refund` receives calls from MCP clients.
* The MCP session ID is stored in `vars.mcpSessionId`.
* The incoming refund payload is stored in `vars.refundRequest`.
* The stored payload is available for use in error handlers and audit logs.

#### Stage 2 — Validate and Compute Approval Flag

A DataWeave 2.0 transformation computes the following fields:

| Field               | Logic                                                                 |
| ------------------- | --------------------------------------------------------------------- |
| `eligibilityCheck`  | Always `PASS` initially; replace with eligibility service integration |
| `duplicateCheck`    | Always `NONE` initially; replace with order management lookup         |
| `amountThreshold`   | `EXCEEDED` if `amount > 50000`, otherwise `WITHIN_LIMIT`              |
| `riskLevel`         | `HIGH-VALUE REFUND` if `amount > 100000`, otherwise `STANDARD`        |
| `riskFlag`          | `true` if `amount > 100000`                                           |
| `policyCheckResult` | `PASS`, `FAIL_INELIGIBLE`, or `FAIL_DUPLICATE`                        |
| `requiresApproval`  | `true` if `amount > 50000` or `riskFlag == true`                      |

#### Stage 3 — Human Approval Elicitation

This stage is triggered only when:

```text
requiresApproval == true
```

The `mcp:create-elicitation` component:

* Sends an approval form to the MCP client.
* Displays order details, amount, risk level, policy result, and refund reason.
* Requires a `decision` of `APPROVE` or `REJECT`.
* Requires a reviewer justification.
* Waits up to `120` seconds by default.
* Returns an `elicitationResult` containing the reviewer action and content.

#### Stage 4 — Execute or Deny Refund

| Elicitation Result                     | Outcome                     |
| -------------------------------------- | --------------------------- |
| `action=ACCEPT` and `decision=APPROVE` | Calls downstream Refund API |
| `action=ACCEPT` and `decision=REJECT`  | Refund is rejected          |
| `action=DECLINE`                       | Refund is not executed      |
| `action=CANCEL`                        | Refund is not executed      |
| `MCP:REQUEST_TIMEOUT`                  | Refund is not executed      |

For low-value refunds that do not require approval, the application directly invokes the downstream Refund API with:

```json
{
  "approvalDecision": "AUTO_APPROVED"
}
```

---

## Decision Logic

### Approval Routing Matrix

| Amount    | Risk Flag | Policy Result     | Path                             | Refund Executed?   |
| --------- | --------: | ----------------- | -------------------------------- | ------------------ |
| ≤ 50,000  |   `false` | `PASS`            | Auto-approve and call Refund API | ✅ Yes              |
| > 50,000  |   `false` | `PASS`            | Human approval required          | ✅ Only on approval |
| > 100,000 |    `true` | `PASS`            | High-value approval required     | ✅ Only on approval |
| Any       |       Any | `FAIL_INELIGIBLE` | Denied immediately               | ❌ No               |
| Any       |       Any | `FAIL_DUPLICATE`  | Denied immediately               | ❌ No               |

---

## MCP Tool Reference

### Tool: `issue_refund`

**Description:**
Issue a refund for a customer order. The tool validates eligibility, checks duplicates, assesses risk, and requires human approval for amounts above `50,000` units.

### Input Schema

```json
{
  "type": "object",
  "properties": {
    "orderId": {
      "type": "string",
      "description": "Order ID, for example ORD-10482"
    },
    "amount": {
      "type": "number",
      "description": "Amount in smallest unit, such as paise for INR",
      "minimum": 1
    },
    "currency": {
      "type": "string",
      "description": "ISO 4217 currency code, for example INR",
      "pattern": "^[A-Z]{3}$"
    },
    "reason": {
      "type": "string",
      "description": "Refund reason",
      "minLength": 5,
      "maxLength": 500
    }
  },
  "required": [
    "orderId",
    "amount",
    "currency",
    "reason"
  ]
}
```

### Input Parameter Details

| Parameter  | Type   | Required | Constraints                         | Example                |
| ---------- | ------ | -------: | ----------------------------------- | ---------------------- |
| `orderId`  | string |        ✅ | Non-empty string                    | `"ORD-10482"`          |
| `amount`   | number |        ✅ | Minimum value: `1`                  | `25000`                |
| `currency` | string |        ✅ | Three uppercase ISO 4217 characters | `"INR"`                |
| `reason`   | string |        ✅ | Between 5 and 500 characters        | `"Item not delivered"` |

> **Amount Note:** For INR, the amount is represented in paise. For example, `25000` represents ₹250.00 and `100000` represents ₹1,000.00.

### Tool Responses

#### Auto-Approved Refund

```json
{
  "status": "SUCCESS",
  "message": "Refund of INR 25000 was auto-approved and successfully issued.",
  "orderId": "ORD-10482",
  "refundId": "corr-abc-123",
  "correlationId": "corr-abc-123"
}
```

#### Human-Approved Refund

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

#### Policy Denial

```text
Refund denied. Policy: FAIL_INELIGIBLE. Order: ORD-99999
```

#### Reviewer Rejection

```text
Refund rejected by reviewer. Order: ORD-20891. Reviewer reason: Duplicate order detected by reviewer.
```

#### Reviewer Decline

```text
Refund not executed. Reviewer action: DECLINE. Order: ORD-20891. No refund was processed.
```

#### Approval Timeout

```text
Refund approval timed out. Order: ORD-20891. No refund was processed. Please retry or escalate.
```

---

## Elicitation: Human-in-the-Loop

When human approval is required, the application uses **MCP Elicitation** to pause the flow and request a decision through the MCP client interface.

### Elicitation Message Format

```text
Refund Approval Required | Order: ORD-20891 | Amount: INR 75000 |
Risk: STANDARD | Policy: PASS | Reason: Customer did not receive item
```

### Elicitation Form Fields

| Field            | Type   | Required | Options               |
| ---------------- | ------ | -------: | --------------------- |
| `decision`       | enum   |        ✅ | `APPROVE`, `REJECT`   |
| `reviewerReason` | string |        ✅ | Free-text explanation |

### Elicitation Actions

| Action    | Description                          | Refund Outcome         |
| --------- | ------------------------------------ | ---------------------- |
| `ACCEPT`  | Reviewer submits the approval form   | Depends on `decision`  |
| `DECLINE` | Reviewer dismisses the approval form | Refund is not executed |
| `CANCEL`  | Operation is cancelled               | Refund is not executed |

### Elicitation Timeout

When no response is received within `elicitationTimeoutSeconds`, the MCP Connector raises:

```text
MCP:REQUEST_TIMEOUT
```

The application fails closed and no refund is issued.

---

## Running the Application

### Option 1: Anypoint Studio

1. Open Anypoint Studio.
2. Select **File → Import → Anypoint Studio → Anypoint Studio project from File System**.
3. Select the `mcp-refund-approval-server` project folder.
4. Update `config.yaml` with your downstream Refund API details.
5. Right-click the project.
6. Select **Run As → Mule Application**.

### Option 2: Maven Command Line

```bash
cd mcp-refund-approval-server

mvn clean package

mvn mule:run
```

### Option 3: Deploy Packaged JAR

```bash
mvn clean package -DskipTests
```

The deployable JAR is created at:

```text
target/mcp-refund-approval-server-1.0.0-SNAPSHOT-mule-application.jar
```

### Verify the Server is Running

```bash
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
      "clientInfo": {
        "name": "TestClient",
        "version": "1.0"
      }
    }
  }'
```

The response should include:

```json
{
  "serverInfo": {
    "name": "Refund Approval MCP Server"
  }
}
```

---

## API Reference

### Endpoint

```text
POST http://localhost:8081/mcp
```

### Required Headers

| Header           | Value                                     | Description                          |
| ---------------- | ----------------------------------------- | ------------------------------------ |
| `Content-Type`   | `application/json`                        | Request body format                  |
| `Accept`         | `application/json` or `text/event-stream` | Use SSE for elicitation interactions |
| `mcp-session-id` | Session ID returned after initialization  | Required after `initialize`          |

### MCP Protocol Flow

```text
Client                                        Server
  │                                              │
  │── POST /mcp (method: initialize) ───────────▶│
  │◀── 200 OK {result: {serverInfo, ...}} ───────│
  │              [mcp-session-id header set]      │
  │                                              │
  │── POST /mcp (method: tools/list) ───────────▶│
  │◀── 200 OK {result: {tools: [issue_refund]}} ─│
  │                                              │
  │── POST /mcp (method: tools/call) ───────────▶│
  │   Accept: text/event-stream                  │
  │                                              │
  │◀── SSE event: elicitation request ───────────│
  │                                              │
  │── POST /mcp (elicitations/respond) ─────────▶│
  │◀── SSE event: tool result ───────────────────│
```

---

## Sample Payloads

### 1. Initialize Session

```http
POST http://localhost:8081/mcp
Content-Type: application/json
Accept: application/json
```

```json
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

**Response**

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

```http
POST http://localhost:8081/mcp
Content-Type: application/json
Accept: application/json
mcp-session-id: {{sessionId}}
```

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list",
  "params": {}
}
```

---

### 3. Standard Refund: Auto-Approved

```http
POST http://localhost:8081/mcp
Content-Type: application/json
Accept: application/json
mcp-session-id: {{sessionId}}
```

```json
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

**Response**

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

### 4. High-Value Refund: Requires Human Approval

```http
POST http://localhost:8081/mcp
Content-Type: application/json
Accept: text/event-stream
mcp-session-id: {{sessionId}}
```

```json
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

**SSE Elicitation Request**

```json
{
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

### 5. Submit Elicitation Response: Approve

```json
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

**Final Tool Result**

```json
{
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

### 6. Submit Elicitation Response: Reject

```json
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

**Final Tool Result**

```json
{
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

### 7. Submit Elicitation Response: Decline

```json
{
  "jsonrpc": "2.0",
  "id": "elicit-001",
  "result": {
    "action": "decline"
  }
}
```

**Final Tool Result**

```json
{
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

### 8. High-Risk Refund: Above Risk Threshold

```json
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

> This request is classified as a `HIGH-VALUE REFUND` and always requires human approval.

---

## Response Schemas

### Downstream Refund API Request: Auto-Approved

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

### Downstream Refund API Request: Human-Approved

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

### Internal Validation Payload

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

## Dependencies

### Runtime Dependencies

| Dependency              | Group ID                  | Artifact ID           | Version  | Type          |
| ----------------------- | ------------------------- | --------------------- | -------- | ------------- |
| MuleSoft HTTP Connector | `org.mule.connectors`     | `mule-http-connector` | `1.11.3` | `mule-plugin` |
| MuleSoft MCP Connector  | `com.mulesoft.connectors` | `mule-mcp-connector`  | `1.6.0`  | `mule-plugin` |

### Build Dependencies

| Dependency         | Group ID                   | Artifact ID          | Version |
| ------------------ | -------------------------- | -------------------- | ------- |
| Mule Maven Plugin  | `org.mule.tools.maven`     | `mule-maven-plugin`  | `4.7.0` |
| Maven Clean Plugin | `org.apache.maven.plugins` | `maven-clean-plugin` | `3.0.0` |

### Runtime Requirement

| Property                 | Value                                      |
| ------------------------ | ------------------------------------------ |
| **Mule Runtime**         | 4.11.0 or higher                           |
| **Java**                 | 17                                         |
| **Minimum Mule Version** | `4.11.0`, declared in `mule-artifact.json` |

---

## Use MCP Inspector as MCP Client

The current MCP Inspector release supports **elicitation**. Use the browser UI instead of `--cli`, because interactive elicitation flows require the browser-based interface.

### 1. Verify Node.js Version

```bash
node -v
```

Node.js version `22.7.5` or later is required.

### 2. Start MCP Inspector

```bash
MCP_SERVER_REQUEST_TIMEOUT=180000 \
MCP_REQUEST_MAX_TOTAL_TIMEOUT=180000 \
npx @modelcontextprotocol/inspector@0.22.0
```

### 3. Open MCP Inspector

Open the following URL in your browser:

```text
http://localhost:6274
```

### 4. Configure the Connection

| Setting    | Value                       |
| ---------- | --------------------------- |
| Transport  | `Streamable HTTP`           |
| Server URL | `http://localhost:8081/mcp` |

<img width="815" height="637" alt="MCP Inspector Streamable HTTP configuration" src="https://github.com/user-attachments/assets/9bf7f62e-db65-4da0-ad90-0f1ab616c9e1" />

### 5. Test the Elicitation Flow

1. Click **Connect**.
2. Open the **Tools** tab.
3. Run the `issue_refund` tool.
4. Enter a high-value refund payload.
5. The elicitation approval form should appear.

### MuleSoft Elicitation Schema

| Property         | Type   | Required Values / Rule |
| ---------------- | ------ | ---------------------- |
| `decision`       | Enum   | `APPROVE`, `REJECT`    |
| `reviewerReason` | String | Required               |

### Verify Elicitation Capability

After connecting, open the **History** tab and inspect the `initialize` request.

The request must include an `elicitation` capability.

If it does not:

1. Disconnect from the server.
2. Connect again.

> MCP capabilities are negotiated only during initialization.

---

## Extending the Application

### Integrate Real Eligibility Checks

Replace the placeholder logic in the Stage 2 DataWeave transformation:

```dataweave
var isEligible = true
var existingRefund = false
```

Replace these stubs with real API integrations:

```dataweave
var isEligible = true          // Call eligibility microservice
var existingRefund = false     // Query order management system
```

Use HTTP Request operations before the transformation to retrieve the actual eligibility and duplicate-refund results.

### Add More Currencies

The `currency` field accepts any three-character ISO 4217 currency code, such as:

```text
USD
EUR
GBP
INR
```

The current validation pattern checks formatting only:

```text
^[A-Z]{3}$
```

### Adjust the Approval Threshold

Use the configured threshold rather than a hard-coded value:

```dataweave
var threshold = ${policy.approvalThresholdAmount} as Number
var requiresApproval: Boolean = (amount > threshold) or riskFlag
```

### Add Audit Logging

Add an HTTP request, event publication, database write, or message queue publish after each decision point to send audit events to your governance or observability platform.

Recommended audit events include:

* Tool invocation received
* Validation completed
* Approval requested
* Reviewer decision received
* Refund executed
* Refund rejected
* Approval timeout
* Unexpected error

---

## Troubleshooting

| Issue                          | Likely Cause                       | Fix                                                       |
| ------------------------------ | ---------------------------------- | --------------------------------------------------------- |
| `HTTP:CONNECTIVITY` on startup | `refundApi.host` is unavailable    | Update `config.yaml` with a valid host and port           |
| Elicitation not received       | Incorrect `Accept` header          | Use `Accept: text/event-stream` for high-value tool calls |
| Session ID error               | `mcp-session-id` missing           | Copy the session ID returned after initialization         |
| Timeout after 120 seconds      | Reviewer did not submit a response | Increase `mcp.elicitationTimeoutSeconds`                  |
| Maven build fails              | Missing Anypoint credentials       | Add credentials to `~/.m2/settings.xml`                   |
| Port `8081` already in use     | Another process is using the port  | Change `http.port` in `config.yaml`                       |

---

## References

* [MuleSoft MCP Connector Documentation](https://docs.mulesoft.com/mcp-connector/latest/)
* [Model Context Protocol Specification](https://modelcontextprotocol.io/specification)
* [MuleSoft Anypoint Platform](https://anypoint.mulesoft.com)
* [DataWeave 2.0 Language Reference](https://docs.mulesoft.com/dataweave/latest/)
* [Mule 4 Error Handling](https://docs.mulesoft.com/mule-runtime/latest/error-handling)
* [MuleSoft General Documentation](https://docs.mulesoft.com/general/)

---

*MCP Refund Approval Server v1.0.0 — Built with MuleSoft 4 and MCP Connector 1.6.0.*
