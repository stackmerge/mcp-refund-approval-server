# MCP Refund Approval Server

A **MuleSoft 4** application that exposes a **Model Context Protocol (MCP)** tool for AI agents to process customer refund requests with **human-in-the-loop approval** for high-value transactions. Built on the MuleSoft MCP Connector using Streamable HTTP transport.

---

## Overview

The **MCP Refund Approval Server** is a Mule application that bridges AI agent systems and human reviewers for refund processing. It exposes a single MCP tool (`issue_refund`) that:

1. **Validates** refund eligibility and checks for duplicates.
2. **Assesses risk** based on transaction amount and flags.
3. **Auto-approves** standard low-value refunds (≤ 50,000 units) and immediately calls the downstream Refund API.
4. **Pauses for human approval** on high-value (> 50,000 units) or high-risk refunds, sending an elicitation form to the MCP client.
5. **Executes or rejects** based on the human reviewer's decision.
6. **Fails closed** — if approval times out, reviewer declines, or an unexpected error occurs, no refund is processed.

## Prerequisites

| Requirement | Version |
|---|---|
| **Mule Runtime** | 4.11.0 or higher |
| **Java JDK** | 17 |
| **Maven** | 3.8+ |
| **MuleSoft MCP Connector** | 1.6.0 |
| **MuleSoft HTTP Connector** | 1.11.3 |

## Running the Application

```bash
cd mcp-refund-approval-server
mvn clean package
mvn mule:run
```

For full documentation see the [README](README.md).
