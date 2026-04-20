# System Design: LLM-powered Customer Support Bot

## Introduction

Traditional rule-based chatbots fail when queries deviate from predefined scripts, leading to generic responses, poor user experience, and costly escalations.

An LLM-powered customer support bot addresses this gap by leveraging large language models to interpret intent, maintain conversational context across multiple exchanges, and generate responses that feel natural. Modern production architectures go further by combining LLMs with Retrieval-Augmented Generation (RAG), which grounds every response in company-specific knowledge bases and real-time data rather than relying on the model's static training data. This dramatically reduces hallucinations and keeps answers accurate. In this chapter, we will design such a system.

## Requirements

### Functional Requirements

The following functional requirements define the system's core behaviour:

- **Dialogue management:** The system must maintain multi-turn conversation history so that follow-up questions like "What about the other item?" are interpreted correctly within the ongoing session, not treated as isolated queries.
- **Natural language understanding:** The system must interpret user intent, extract key entities such as order IDs and product names, and disambiguate vague queries using contextual cues from the conversation.
- **Response generation:** The system must fetch relevant documents from a company knowledge base or vector database and generate accurate, grounded answers instead of hallucinating information.
- **Function calling:** The system must invoke backend APIs when the user's intent maps to an actionable operation, such as checking order status or initiating a refund.
- **Human escalation:** The system must detect when it cannot confidently resolve a query and gracefully hand off the conversation to a human agent with full context preserved.
- **Feedback collection:** The system should collect user ratings on responses to build a signal for improving response quality over time.

<img width="799" height="317" alt="image" src="https://github.com/user-attachments/assets/d585eb48-235b-4d54-a631-d074feb28efa" />


These functional requirements ensure the bot is not just conversational but genuinely useful.

### Non-Functional Requirements

The following non-functional requirements are essential for a customer support bot serving millions of users.

- **Scalability:** The system must handle millions of concurrent users and scale horizontally, particularly during peak support hours such as holiday shopping seasons or product launches.
- **Low latency:** Responses should arrive within two to three seconds to maintain a natural conversational feel. This demands efficient inference pipelines, caching of frequent queries, and cost-aware routing.
- **Availability:** The system should target 99.9% uptime because customer support is a business-critical function where downtime directly impacts revenue and trust.
- **Reliability:** The system must produce consistent responses and degrade gracefully under heavy load rather than returning errors or nonsensical outputs.
- **Cost efficiency:** LLM inference is expensive at scale, so the architecture must incorporate tiered model selection, response caching, and batched inference to control operational costs.
- **Privacy and security:** User conversations often contain sensitive personal or financial data. The system must encrypt data in transit and at rest, comply with regulations like GDPR, and enforce strict access controls.

## Resource Estimation

Back-of-the-envelope calculations help us understand the magnitude of infrastructure required before committing to an architecture. Our estimates are based on the following assumptions:

- **Total registered users:** 500 million
- **Daily active users (DAUs):** 50 million
- **Average queries per user per day:** 5
- **Total requests per day:** 250 million
- **Average request size:** ~2 KB (metadata, headers, and user text)
- **Average response size:** ~4 KB (LLM-generated text is usually longer than input)

### Number of Servers Estimation

To estimate the number of servers, we first estimate the total number of requests per second. We assume that all requests from daily active users (50 million) arrive at once to estimate the peak load.

$$\text{Number of requests/second} = 50 \text{ Million}$$

The number of servers required at peak load:

$$\text{Servers required at peak load} = \frac{\text{Number of requests/second}}{\text{RPS of a server}}$$

Using 64,000 RPS as the estimated capacity per server:

$$\text{Servers required at peak load} = \frac{50 \text{ Million}}{64{,}000} \approx 782 \text{ servers}$$

However, GPU inference servers are the real bottleneck. Assuming each GPU server handles approximately 200 inference requests per second, the number of GPU servers required would be:

$$\text{GPU servers} = \frac{50 \text{ M}}{200} = 250 \text{ K servers}$$

> **Practical tip:** In practice, this number would be significantly lower. Real-world traffic is distributed over time rather than arriving simultaneously, which reduces the actual peak load. Additionally, inference optimizations such as batching, caching, and efficient model serving greatly improve throughput. As a result, the infrastructure requirement would likely scale to hundreds or a few thousand GPU servers, depending on the model size and latency requirements.

### Storage Estimation

In an LLM-powered customer support bot, the request size is typically around 2 KB, which includes the user query, metadata, and headers. The response size is usually larger, around 4 KB, since the LLM generates longer explanatory replies. The combined single request and response size becomes 6 KB.

$$\text{Storage required per day} = 6 \text{ KB} \times 50 \text{ M} \times 5 \approx 1.5 \text{ TB/day}$$

$$\text{Storage required per year} = 6 \text{ KB} \times 50 \text{ M} \times 5 \times 365 \approx 548 \text{ TB/year}$$

Model storage also contributes to the total storage requirements. A 1-trillion parameter model stored in FP16 precision requires about 2 TB of storage. This is a fixed requirement and remains constant unless the model size or precision changes (e.g., upgrading to a larger model or different data type).

### Bandwidth Estimation

Assuming a 2 KB request size and a 4 KB response size:

**Incoming bandwidth:**

$$\text{Incoming bandwidth} = \text{Requests/sec} \times \text{Request size} = 50 \text{ M/s} \times 2 \text{ KB} = 100 \text{ GBps}$$

**Outgoing bandwidth:**

$$\text{Outgoing bandwidth} = \text{Responses/sec} \times \text{Response size} = 50 \text{ M/s} \times 4 \text{ KB} = 200 \text{ GBps}$$

## Building Blocks

Designing this system requires a set of well-defined building blocks, each addressing a specific concern surfaced by our requirements and resource estimates.

<img width="737" height="231" alt="image" src="https://github.com/user-attachments/assets/0511b01d-428f-45ca-9de1-9703459decf2" />


The following components will form the backbone of the architecture:

- **Load balancers:** Distribute incoming user requests across application and inference servers to prevent hotspots.
- **Database:** Maintain session context across multi-turn dialogues so follow-up questions resolve correctly.
- **Cache layer:** Stores frequent query-response pairs to reduce redundant LLM inference calls, directly supporting cost-aware routing.
- **Pub-sub system:** Enables asynchronous event processing for logging, feedback collection, and analytics without blocking the main request path.
- **Rate limiter:** Controls the number of requests a user or client can send within a specific time window.

## High-Level Design

Production support bots quickly become outdated as product details and policies change, leading to inaccurate responses and declining user satisfaction. Systems must continuously use up-to-date knowledge while controlling LLM costs.

The following high-level design uses Retrieval-Augmented Generation (RAG) with cost-aware routing to deliver accurate, context-rich responses. The workflow is as follows: a user submits a query through a web or mobile interface, which is handled by the API gateway for authentication, rate limiting, and session tracking before being forwarded to the backend. The RAG pipeline retrieves relevant knowledge from a vector database and augments the prompt, which is then passed to the LLM to generate a response. The response goes through content moderation before being returned to the user, and the conversation is logged with feedback collected.

<img width="983" height="654" alt="image" src="https://github.com/user-attachments/assets/b4ed201d-6eb4-4f8d-b00f-e91478263fc3" />


This architecture ensures the system always references up-to-date product knowledge rather than relying solely on what the model learned during training.

## API Design

To support the functional requirements, we define a set of APIs that enable communication between system components and handle key operations within the LLM-powered customer support bot.

### sendMessage()

Handles incoming user queries, maintains session context, and returns a response by orchestrating NLU, RAG, and LLM components. It ensures multi-turn dialogue by attaching conversation history to each request.

> **`POST /v1/messages`**

```
sendMessage(session_id, user_id, message)
```

| Parameter | Description |
| --- | --- |
| `session_id` | A unique identifier for the conversation session. |
| `user_id` | A unique identifier for the user. |
| `message` | User's input text query. |

### parseQuery()

Extracts user intent and key entities (e.g., order ID, product name) from the input text. This API is used by the NLU service to structure unstructured queries for downstream processing.

```
parseQuery(text)
```

| Parameter | Description |
| --- | --- |
| `text` | Raw user query in natural language. |

### retrieveAndGenerate()

Fetches relevant documents from the vector database and generates a grounded response using the LLM. It ensures answers are based on up-to-date knowledge instead of model memory.

```
retrieveAndGenerate(query)
```

| Parameter | Description |
| --- | --- |
| `query` | User query (possibly enriched with context). |

### executeAction()

Invokes backend services when a query requires an action, such as checking order status or initiating a refund. It bridges the LLM with external systems through function calling.

> **`POST /v1/actions`**

```
executeAction(action, parameters)
```

| Parameter | Description |
| --- | --- |
| `action` | Name of the operation (e.g., "check_order_status", "initiate_refund"). |
| `parameters` | Key-value pairs required for the action (e.g., `{ "order_id": "1234" }`). |

### escalateToHuman()

Transfers the conversation to a human agent when the system detects low confidence or complex queries. It preserves full conversation context for seamless handoff.

> **`POST /v1/escalate`**

```
escalateToHuman(session_id, user_id, reason)
```

| Parameter | Description |
| --- | --- |
| `reason` | Cause for escalation (e.g., "low_confidence", "complex_query"). |

### submitFeedback()

Collects user ratings and feedback on responses to improve system performance over time. This data is used for monitoring, retraining, and quality evaluation.

> **`POST /v1/feedback`**

```
submitFeedback(session_id, user_id, message_id, rating, comment)
```

| Parameter | Description |
| --- | --- |
| `message_id` | Identifier of the specific response being rated. |
| `rating` | Feedback value (e.g., "thumbs_up", "thumbs_down"). |
| `comment` (optional) | Additional user feedback text. |

## Detailed Design

The high-level architecture shows what the system does. The detailed design reveals how it does it, and this is where the real engineering complexity lives. Each component must be carefully orchestrated to handle concerns that only surface at scale, including LLM performance degradation over time, cost management across model tiers, and seamless human-agent escalation when the bot reaches its limits.

<img width="984" height="823" alt="image" src="https://github.com/user-attachments/assets/46cb14e5-a719-436d-aeeb-e63f848566e7" />


### Components

The architecture comprises several interconnected services, each addressing a specific production concern. Here is how they work together.

- **Entry layer and app servers:** The entry layer handles incoming requests by managing authentication, rate limiting, and distributing traffic to ensure security and stability. The app servers process requests, manage business logic and user context, and prepare them for the pre-processing (NLU) unit.

- **Pre-processing (NLU service):** This service classifies user intent (billing inquiry, technical issue, order status) and extracts entities such as order IDs and product names. Think of it as the triage nurse in an emergency room, quickly categorizing the problem before routing it to the right specialist.

- **Cost-aware router:** This often-overlooked component analyzes query complexity and routes simple, high-confidence queries (store hours, return policy) to a lightweight, cheaper model while directing complex or ambiguous queries to the full LLM. Without this layer, every FAQ lookup burns the same compute budget as a multi-step troubleshooting session.

- **RAG pipeline:** The retrieval service converts the query into a vector embedding, searches the vector database for semantically similar product documentation chunks, ranks results by relevance, and injects the top-k results into the LLM prompt as grounding context. This ensures responses reflect current product knowledge rather than stale training data.

- **LLM core servers:** These host the large language model with function calling capabilities, enabling the bot to invoke external APIs for order status lookups, support ticket creation, and refund processing as structured tool calls.

- **Vector database:** This stores chunked and embedded product documentation, FAQs, troubleshooting guides, and policy documents. A knowledge ingestion pipeline updates these embeddings whenever source documents change, keeping the retrieval layer up to date.

- **Redis caching layer:** This stores recent conversation history per session, maintaining multi-turn context without repeated database lookups. Each new prompt includes relevant prior turns so the LLM can reference what the user said earlier.

- **Content moderation service:** This scans LLM outputs for policy violations, PII leakage, or harmful content before delivery to the user.

- **Human escalation module:** This uses confidence scoring to detect when the LLM cannot adequately resolve a query and routes the conversation to a live agent, preserving full conversation context for the agent.

- **Monitoring service:** This continuously tracks LLM response accuracy, latency, user satisfaction scores, and cost per query. It detects performance degradation early and can trigger knowledge base updates or model retraining.

- **MongoDB:** This stores conversation logs, user profiles, ticket metadata, and escalation records for audit and analytics purposes.

> **Practical tip:** When implementing the cost-aware router, start by logging query classifications for two weeks before activating routing decisions. This lets you validate the classifier's accuracy without risking degraded responses for real users.

The following table illustrates how the cost-aware routing strategy distributes queries across model tiers.

| Query Tier | Example Queries | Model Used | Avg Latency | Relative Cost | Accuracy Target |
| --- | --- | --- | --- | --- | --- |
| Simple/FAQ | "What are your return policies?", "Store hours" | Lightweight model (e.g., fine-tuned small LLM) | <200ms | Low | >95% |
| Moderate | "Help me troubleshoot my device not charging" | Full LLM with RAG | 500ms–1s | Medium | >90% |
| Complex/Sensitive | "I want to dispute a charge and escalate to a manager" | Full LLM with RAG + human escalation | 1–3s (before handoff) | High | >85% (with human fallback) |

With each component defined, the next section traces how a single user query flows through the entire system from submission to response.

### Workflow

Understanding the end-to-end workflow clarifies how these components coordinate in real time. Each step below maps directly to a component described above.

1. User submits a query through the web or mobile chat interface.

2. The API gateway authenticates the request, enforces rate limits, and forwards it to the load balancer.

3. The load balancer distributes the request to an available application server.

4. The pre-processing NLU service classifies intent and extracts entities (order IDs, product names).

5. The cost-aware router evaluates query complexity. Simple queries are sent to the lightweight model. Complex queries proceed to the full RAG-augmented LLM pipeline.

6. For RAG-routed queries, the retrieval service searches the vector database for relevant documentation chunks and ranks them by semantic similarity.

7. The top-k retrieved chunks are injected into the prompt alongside conversation history from Redis. The LLM generates a response, optionally invoking function calls such as an order lookup API.

8. The content moderation service scans the response for policy violations or PII before it leaves the system.

9. If the LLM's confidence score falls below a threshold, the human escalation module routes the conversation to a live agent via pub/sub, preserving full context.

10. The validated response is returned to the user through the API gateway.

11. The conversation is logged in MongoDB, and user feedback (thumbs up/down, comments) is collected and stored for continuous improvement and model retraining.

> **Attention:** Confidence scoring thresholds for human escalation require careful calibration. Setting the threshold too low floods agents with unnecessary escalations. Setting it too high means frustrated users never reach a human. Monitor escalation rates weekly and adjust.

## Requirements Compliance

A well-engineered system must demonstrate that every requirement traces back to specific architectural decisions. Functional requirements ensure the bot performs its intended tasks. Non-functional requirements define how well it performs under real-world conditions at scale. The following table maps each requirement to the components that fulfill it.

| Functional Requirement | How It Is Achieved |
| --- | --- |
| Accurate responses grounded in product knowledge | RAG pipeline retrieves current documentation from the vector database, preventing hallucination |
| Multi-turn conversation support | Redis caching layer maintains session-level conversation history injected into each prompt |
| Function calls, i.e., Order status and account actions | API calling enables the LLM to invoke external APIs for real-time data retrieval |
| Human escalation | Confidence scoring triggers routing to live agents via pub/sub with full context preservation |
| Feedback collection | Feedback module captures structured and unstructured feedback stored in MongoDB |

Non-functional requirements include scalability, low latency, availability, reliability, cost efficiency, and security & privacy. The table below highlights how the design satisfies these constraints.

| Non-functional Requirement | How It Is Achieved |
| --- | --- |
| Scalability | Load balancer distributes traffic across app servers, and horizontal scaling of stateless services (LLM servers, RAG pipeline, vector DB) handles millions of concurrent users. |
| Low latency | Cost-aware routing sends simple queries to lightweight models, while caching (Redis) and efficient RAG retrieval reduce response time to 2–3 seconds. |
| Availability | Redundant services (API gateway, load balancer, distributed servers) and monitoring ensure high uptime (99.9%). |
| Reliability | RAG grounding ensures consistent, accurate responses, while graceful degradation via routing and human escalation prevents failures under load. |
| Cost efficiency | Tiered model selection, caching frequent responses, and batched inference reduce expensive LLM usage. |
| Privacy and security | API gateway enforces authentication, encryption protects data in transit and at rest, and content moderation prevents PII leakage and ensures compliance (e.g., GDPR). |

## Conclusion

This lesson presented a production-ready architecture for an LLM-powered customer support bot that combines RAG for accurate, up-to-date responses with cost-aware routing to optimize efficiency at scale. By integrating components such as NLU, vector search, caching, function calling, and human escalation, the system balances accuracy, latency, and cost while maintaining a seamless user experience. Continuous monitoring and knowledge ingestion ensure the system adapts to changing data and avoids model drift, making it robust for real-world deployment. Ultimately, this design demonstrates how grounded retrieval, intelligent routing, and human-in-the-loop mechanisms form the foundation of reliable, scalable LLM applications.


