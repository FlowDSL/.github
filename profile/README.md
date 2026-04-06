<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://flowdsl.com/logo/flowdsl-light.png">
  <source media="(prefers-color-scheme: light)" srcset="https://flowdsl.com/logo/flowdsl-dark.png">
  <img alt="FlowDSL" height="72" src="https://flowdsl.com/logo/flowdsl-dark.png">
</picture>

<br />
<br />

**The open specification for executable event-driven flows.**

_Nodes define business logic. Edges define delivery semantics. The runtime enforces guarantees._

<br />

[![Spec v1.0.0](https://img.shields.io/badge/Spec-v1.0.0-6366f1?style=flat-square&logo=gitbook&logoColor=white)](https://github.com/flowdsl/spec)
[![Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-22c55e?style=flat-square)](https://github.com/flowdsl/spec/blob/main/LICENSE)
[![Discord](https://img.shields.io/badge/Discord-Join_Community-5865F2?style=flat-square&logo=discord&logoColor=white)](https://discord.gg/flowdsl)
[![Website](https://img.shields.io/badge/flowdsl.com-000?style=flat-square&logo=vercel&logoColor=white)](https://flowdsl.com)

</div>

---

## Where FlowDSL fits

Each specification describes a different layer of your system. FlowDSL completes the picture:

| Spec | Describes |
|------|-----------|
| **OpenAPI** | HTTP interfaces — endpoints, request/response schemas |
| **AsyncAPI** | Event contracts — channels, message payloads, brokers |
| **FlowDSL** | Flow graphs — nodes, edges, delivery semantics, runtime guarantees |

FlowDSL is **fully self-contained**. Events and packets are defined natively in `components.events` and `components.packets`. OpenAPI and AsyncAPI are optional integrations — your FlowDSL documents work without them.

---

## What it looks like

A complete LLM email triage pipeline — classified, routed, and delivered with the right guarantees at each step:

```yaml
flowdsl: "1.0.0"
info:
  title: Smart Email Triage
  version: "1.0.0"

flows:
  email_triage:
    nodes:
      email_fetcher:   { $ref: "#/components/nodes/EmailFetcherNode" }
      llm_classifier:  { $ref: "#/components/nodes/LlmClassifierNode" }
      intent_router:   { $ref: "#/components/nodes/IntentRouterNode" }
      email_sender:    { $ref: "#/components/nodes/EmailSenderNode" }
      slack_notifier:  { $ref: "#/components/nodes/SlackNotifierNode" }
    edges:
      - from: email_fetcher
        to: llm_classifier
        packet: "#/components/packets/IncomingEmail"
        delivery:
          mode: direct                # in-process, zero overhead

      - from: intent_router
        to: email_sender
        when: "output.category == 'support' || output.category == 'inquiry'"
        delivery:
          mode: durable          # business-critical — persisted to MongoDB
          store: mongo
          retryPolicy:
            $ref: "#/components/policies/emailRetry"

      - from: intent_router
        to: slack_notifier
        when: "output.priority == 'high'"
        delivery:
          mode: ephemeral        # burst smoothing via Redis
          backend: redis

components:
  events:
    EmailReceived:
      summary: A new email has been fetched from the inbox
      version: "1.0.0"
      entityType: email
      action: received
      payload: { $ref: "#/components/packets/IncomingEmail" }
```

The JSON document is always the source of truth — not the canvas, not the runtime config.

---

## Five delivery modes. One document.

Stop hardcoding Kafka everywhere. Each edge in your flow declares exactly the delivery guarantee it needs:

| Mode | Transport | Durability | Best for |
|------|-----------|------------|----------|
| `direct` | In-process | None | Fast transforms, cheap steps |
| `ephemeral` | Redis / NATS / RabbitMQ | Low | Burst smoothing, worker pools |
| `checkpoint` | Mongo / Redis / Postgres | Stage-level | High-throughput replay |
| `durable` | Mongo / Postgres | Packet-level | Business-critical transitions |
| `stream` | Kafka / Redis / NATS | Durable stream | External integration, fan-out |

---

## Ten node kinds

Every node in the registry has a semantic kind — the runtime uses it for scheduling, Studio uses it for color-coding:

| Kind | What it does |
|------|-------------|
| `source` | Ingests external events — email, webhook, Kafka, Postgres |
| `transform` | Reshapes and maps payloads |
| `router` | Conditional fan-out to multiple downstream nodes |
| `llm` | LLM inference — prompt → structured output |
| `action` | External side effect — write, notify, publish |
| `checkpoint` | Durable state persistence for replay |
| `publish` | Event bus publisher |
| `terminal` | Flow terminator / data sink |
| `integration` | Third-party platform connector |
| `subworkflow` | Delegates execution to a child FlowDSL workflow |

---

## The ecosystem

<table>
<tr>
<td width="50%" valign="top">

### 📐 [flowdsl/spec](https://github.com/flowdsl/spec)
The canonical FlowDSL JSON Schema, `node.proto` gRPC contract, node manifest format, and 8 reference examples. Single source of truth — Studio and website consume spec artefacts directly.

</td>
<td width="50%" valign="top">

### 🎨 [flowdsl/studio](https://github.com/flowdsl/studio)
Visual drag-and-drop editor built with React Flow. Bidirectional — the canvas and JSON panel stay in sync at all times. Validate, export, and configure delivery semantics visually.

</td>
</tr>
<tr>
<td width="50%" valign="top">

### ⚙️ [flowdsl/flowdsl-go](https://github.com/flowdsl/flowdsl-go)
The execution engine. Reads FlowDSL documents, invokes nodes via gRPC, and manages all five delivery guarantees across MongoDB, Redis, and Kafka.

</td>
<td width="50%" valign="top">

### 🐍 [flowdsl/flowdsl-py](https://github.com/flowdsl/flowdsl-py)
Write nodes in Python. Includes `BaseNode`, a gRPC server scaffold, and first-class [redelay](https://redelay.com) framework integration.

</td>
</tr>
<tr>
<td width="50%" valign="top">

### 🟡 [flowdsl/flowdsl-js](https://github.com/flowdsl/flowdsl-js)
Write nodes in TypeScript. Full `NodeService` gRPC implementation with streaming support.

</td>
<td width="50%" valign="top">

### 📦 [flowdsl/examples](https://github.com/flowdsl/examples)
Real-world flows — email triage, webhook alerts, Kafka stream filtering, DB anomaly detection, HTTP-to-Mongo sync, and domain enrichment pipelines.

</td>
</tr>
</table>

---

## Multi-language nodes via gRPC

Nodes can be written in **Go**, **Python**, or **TypeScript**. Cross-language invocation uses **gRPC + Protobuf** — no HTTP between nodes, no serialization overhead.

```
FlowDSL Runtime (Go)
  │
  ├── Go nodes      ── in-process (direct function call, zero overhead)
  ├── Python nodes  ── gRPC → flowdsl-nodes-py:50052
  └── JS/TS nodes   ── gRPC → flowdsl-nodes-js:50053
```

Every node implements the same `NodeService` interface — defined once in [`schemas/node.proto`](https://github.com/flowdsl/spec/blob/main/schemas/node.proto), consumed by all three language SDKs.

---

## Node registry

Pre-built nodes are published at **[repo.flowdsl.com](https://repo.flowdsl.com)** — the open registry for FlowDSL nodes.

```json
{
  "id": "flowdsl/llm-analyzer",
  "kind": "llm",
  "language": "python",
  "runtime": {
    "invocation": "grpc",
    "grpc": { "port": 50052, "streaming": true }
  },
  "settingsSchema": {
    "properties": {
      "prompt":        { "type": "string" },
      "model":         { "enum": ["claude-sonnet-4-5", "gpt-4o", "gemini-1.5-pro"] },
      "outputFormat":  { "enum": ["text", "json"] }
    }
  }
}
```

Drag a node from the Studio catalog → fill in settings → connect edges. No boilerplate. No transport code.

---

## Get started

```bash
# Validate an example against the schema
git clone https://github.com/flowdsl/spec
cd spec
npm install -g ajv-cli
ajv validate -s schemas/flowdsl.schema.json -d examples/email-triage.flowdsl.json

# Or clone the examples repo and spin up infrastructure
git clone https://github.com/flowdsl/examples
cd examples
make up-infra          # MongoDB · Redis · Kafka · ClickHouse
open http://localhost:5173   # FlowDSL Studio
```

Full tutorial → [flowdsl.com/docs/getting-started](https://flowdsl.com/docs/getting-started)

---

## AI-native

FlowDSL docs ship with:

- [`/llms.txt`](https://flowdsl.com/llms.txt) and [`/llms-full.txt`](https://flowdsl.com/llms-full.txt) for LLM consumption
- Native **MCP server** at `https://flowdsl.com/mcp`

Connect your IDE to the FlowDSL knowledge base:

```bash
# Claude Code
claude mcp add --transport http flowdsl https://flowdsl.com/mcp

# Cursor — .cursor/mcp.json
{ "mcpServers": { "flowdsl": { "type": "http", "url": "https://flowdsl.com/mcp" } } }
```

---

## Commercial layer

FlowDSL is **Apache 2.0**. The commercial products build on top:

| Product | What it is |
|---------|------------|
| **Node Catalog** _(coming soon)_ | Node marketplace — build, publish, and sell FlowDSL nodes |
| **Cloud Infrastructure** _(coming soon)_ | Managed FlowDSL workflow hosting for businesses |

---

## Contributing

Contributions are welcome across all repos. The best place to start is the spec:

```bash
git clone https://github.com/flowdsl/spec
cd spec

# Add a node manifest, write an example flow, improve the schema
# Then validate:
ajv validate -s schemas/flowdsl.schema.json -d examples/your-flow.flowdsl.json
```

→ [CONTRIBUTING.md](https://github.com/flowdsl/spec/blob/main/CONTRIBUTING.md)
→ [Good first issues](https://github.com/flowdsl/spec/labels/good%20first%20issue)
→ [Discord](https://discord.gg/flowdsl)

---

<div align="center">

[Website](https://flowdsl.com) · [Docs](https://flowdsl.com/docs) · [Studio](https://flowdsl.com/studio) · [Node Registry](https://repo.flowdsl.com) · [Discord](https://discord.gg/flowdsl)

<br />

Apache 2.0 · Built with love by the FlowDSL community

</div>
