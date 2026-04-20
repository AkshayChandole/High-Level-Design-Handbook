# System Design: ChatGPT System

## [What is ChatGPT?](#what-is-chatgpt)

ChatGPT is an advanced conversational application built on a sophisticated AI language model developed by OpenAI. It understands natural language and generates human-like text to assist with various communication tasks. ChatGPT helps users by:

- Simplifying technical subjects.
- Proposing innovative ideas.
- Composing clear, well-structured text.

<img width="781" height="468" alt="image" src="https://github.com/user-attachments/assets/112fd5a1-666e-4362-b54f-823044aad82e" />


ChatGPT-like systems are notable for interpreting incomplete or ambiguous inputs to provide relevant, context-aware responses. Instead of relying on predefined answers, these systems use generative AI and large language models to analyze intent and patterns.

## [Requirements](#requirements)

The backend design must address the following functional and non-functional requirements:

### [Functional Requirements](#functional-requirements)

- **Dialogue management:** Interpret user input, maintain context, and generate relevant, human-like responses.
- **Natural language understanding (NLU):** Identify intent and extract key entities from user input.
- **Personalization:** Adapt to user preferences and history to provide tailored interactions.
- **Feedback:** Improve accuracy and performance over time using user feedback.

### [Non-Functional Requirements](#non-functional-requirements)

- **Scalability:** Handle increasing user traffic and workloads efficiently.
- **Availability:** Ensure consistent uptime for users.
- **Reliability:** Generate stable and predictable responses under varying conditions.
- **Low latency:** Minimize response times through efficient inference and caching.
- **Privacy and security:** Protect user data and ensure compliance with privacy standards.

<img width="907" height="376" alt="image" src="https://github.com/user-attachments/assets/9e0e68f8-d192-430c-a4c7-92e61747e88f" />


## [Resource Estimation](#resource-estimation)

We will use the following assumptions to estimate resources for a large user base:

- **Users:** 1 billion total users, with 150 million daily active users (DAU)
- **Requests:** Each active user sends ~10 requests daily, totaling 1.5 billion requests per day
- **Request size:** Approximately 2 KB (mostly short text prompts)
- **Response size:** Approximately 5 KB (AI-generated replies are typically longer)

### [Number of Servers Estimation](#number-of-servers-estimation)

First, we calculate the average requests per second (RPS) based on the daily volume:

$$\text{Servers needed at peak load} = \frac{\text{Number of requests/second}}{\text{RPS of server}}$$

Using a standard web server capacity of 64,000 RPS (from the back-of-the-envelope calculation section), the web tier requirements are minimal. However, this calculation applies to application servers handling connections, not the GPU servers required for inference.

$$\text{Servers needed at peak load} = \frac{150 \text{ million}}{64{,}000} \approx 2{,}343 \text{ servers}$$

### [Storage Estimation](#storage-estimation)

Storage requirements depend on the total number of requests and the data size per request:

$$\text{Total storage} = \text{Total daily requests} \times \text{Request size}$$

**Daily calculations:**

$$\text{Total daily requests} = \text{Total DAU} \times \text{Average daily requests per user} = 150 \text{ million} \times 10 = 1{,}500 \text{ million}$$

$$\text{Total daily storage} = 1{,}500 \text{ million} \times 2 \text{ KB} = 3 \text{ TB/day}$$

**Annual storage requirement:**

$$3 \text{ TB/day} \times 365 \text{ days/year} = 1{,}095 \text{ TB/year} \approx 1.1 \text{ PB/year}$$

> **Note:** This estimate covers text-based request data only. Actual storage will be higher when accounting for response data, conversation history, user metadata, and model artifacts.

### [Bandwidth Estimation](#bandwidth-estimation)

We calculate bandwidth separately for incoming and outgoing traffic.

#### [Incoming Bandwidth](#incoming-bandwidth)

Incoming bandwidth is determined by the number of incoming requests per second and the size of each request. Since most user requests to ChatGPT are relatively short, we assume an average request size of 2 KB.

$$\text{Total incoming bandwidth} = \text{Total requests/sec} \times \text{Request size}$$

$$\text{Requests/sec} = \frac{1{,}500 \text{ million}}{86{,}400 \text{ sec}} \approx 17{,}361 \text{ requests/sec}$$

$$\text{Incoming bandwidth} = 17{,}361 \times 2 \text{ KB} \approx 34.7 \text{ MB/s} \approx 277.8 \text{ Mbps}$$

#### [Outgoing Bandwidth](#outgoing-bandwidth)

Outgoing bandwidth depends on the number of responses per second and the size of each response. As responses are generated dynamically based on user input, response sizes vary. For estimation, assume an average response size of 5 KB per request.

$$\text{Total outgoing bandwidth} = \text{Total responses/sec} \times \text{Response size}$$

$$\text{Outgoing bandwidth} = 17{,}361 \times 5 \text{ KB} \approx 86.8 \text{ MB/s} \approx 694.4 \text{ Mbps}$$

Outgoing bandwidth consumption is significantly higher due to the larger size of generated responses.

## [Building Blocks](#building-blocks)

Based on our requirements and estimations, we will use the following core components:

- **Load balancers:** Distribute user requests across multiple servers and services.
- **Databases:** Store metadata and graph-structured data.
- **CDNs:** Deliver static content to end users, reducing latency and server load.
- **Pub-sub system:** Manage real-time messaging and events.
- **Cache:** Store frequently accessed data to improve performance.

Additional components, such as an API gateway and ZooKeeper, are also critical.

## [High-Level Design](#high-level-design)

The high-level design illustrates how the system handles real-time conversations. The following workflow outlines the component interactions:

<img width="971" height="530" alt="image" src="https://github.com/user-attachments/assets/f3712b73-88a5-42ff-a208-86b66d694e03" />


1. **User input:** The user submits a text prompt via the interface or API.
2. **Gateway processing:** The API gateway authenticates the request, applies rate limiting, manages the session, and forwards the prompt to the model server.
3. **Model inference:** The AI model processes the prompt using conversation history. Responses are cached for retrieval and logged in the database.
4. **Response delivery:** The generated response is returned to the user via the API gateway.
5. **Feedback loop:** User feedback is collected to improve system performance and fine-tune future models.

User feedback is essential for fine-tuning the system, enhancing its accuracy, and natural language generation.

> **Q: Is a typical cache (like LRU or TTL-based) sufficient for storing AI-generated responses?**
>
> While traditional caching strategies such as LRU or TTL are effective for static or frequently accessed web content, they are often inadequate for AI-generated responses, which are highly contextual and dependent on nuanced input factors such as prompt wording, user history, and personalization.
>
> To address this, ChatGPT-like systems benefit from a **semantic cache** that stores not only the response but also a vector embedding of the input prompt that captures its meaning. When a new prompt is received, the system can retrieve previous responses whose embeddings are semantically similar using a vector-based cache powered by approximate nearest neighbor (ANN) search. This enables the reuse of responses based on meaning rather than exact text matches.
>
> A hybrid approach proves to be more effective in maintaining performance, relevance, and scalability.

## [Detailed Design](#detailed-design)

Requests are routed through the API gateway to a load balancer, which distributes traffic across application servers. The system relies on the following core components:

<img width="1226" height="671" alt="image" src="https://github.com/user-attachments/assets/380f7af1-a84e-4176-b6e6-77ea5c740c73" />


- **Pre-processing (NLU service):** Extracts meaning, intent, and entities from user input. It converts text into numerical embeddings to capture semantic relationships.
- **Model server:** The core processing unit hosting AI models. It generates responses based on user queries.
- **Vector database:** Stores vector embeddings of conversation history and external knowledge. This enables the retrieval of semantically similar content for context-aware responses.
- **Redis (caching layer):** Temporarily stores recent conversation history to prevent redundant database lookups and improve latency.
- **Content moderation system:** Scans generated responses for policy violations and sensitive content before delivery.
- **User profile service:** Manages authentication, session tracking, and user preferences to personalize interactions.
- **ZooKeeper (orchestrator):** Coordinates model servers by tracking active instances, distributing workloads, and handling failovers.
- **Pub/sub:** Manages asynchronous communication (e.g., billing, logging) to ensure events propagate without blocking API calls.
- **Payment processing system:** Handles transaction logging, billing, and payments, coordinated via the pub/sub system.
- **MongoDB (user database):** Stores structured data, such as session metadata and billing history. MongoDB is selected for its flexible data models and horizontal scalability.

> **Q: How does the payment system determine how much to charge a user for a request in a system like ChatGPT, especially when billing is based on token usage or premium features?**
>
> The payment system calculates charges based on usage logs/data, which track details like the number of tokens used, the complexity of the request, and any premium features accessed.
>
> After processing the request, the billing service computes the appropriate cost based on factors like token count or other pricing metrics.
>
> This information is then sent to the payment system for user approval to ensure that the user is fully informed and consents to the charge before finalizing the transaction.

### [Workflow of the ChatGPT System](#workflow-of-the-chatgpt-system)

When a user enters a prompt through a web or mobile interface, the request first reaches the API gateway, which manages authentication, authorization, and rate limiting. From there, a load balancer distributes the request across multiple application servers to ensure efficient handling.

Before reaching the model, the prompt passes through the preprocessing NLU service. If the user has interacted before, a user profile service retrieves relevant historical data to personalize the response. Once preprocessed, the request is sent to the model servers, where the large language model (LLM) generates a response. When additional context is required, the system queries the vector database to retrieve semantically relevant information from past interactions.

Before the response is returned, it passes through the content moderation system to filter out harmful or inappropriate content. ZooKeeper coordinates the model servers by managing configuration, leader election, and health monitoring during scaling and failover.

If the request involves a paid feature, the payment processing system logs the transaction and updates records in MongoDB. Meanwhile, a CDN ensures static assets, such as UI elements, load quickly, but generated responses from the model are handled separately.

Finally, the response is sent back to the user. If the user provides feedback, it is collected via the feedback module and stored in the feedback database. Structured feedback (e.g., ratings or thumbs up/down) is stored in a relational database (e.g., PostgreSQL or MySQL). In contrast, unstructured feedback (such as comments or session metadata) is stored in a document database, such as MongoDB. This data helps improve the system over time through analysis and model updates.

## [Achieving Requirements](#achieving-requirements)

The following sections evaluate the design against the previously identified functional and non-functional requirements.

### [Achieving Functional Requirements](#achieving-functional-requirements)

| Functional Requirement | Techniques |
| --- | --- |
| **Dialogue management** | The pre-processing NLU service interprets user input. The model server generates context-aware responses. User profile service maintains the session context. |
| **Natural language understanding** | The pre-processing NLU service extracts intent and entities and understands user input. Vector DB enables semantic retrieval. Redis caches recent context for fast access. |
| **Personalization** | User profile stores preferences and history. Model server uses this data for tailored responses. |
| **Feedback** | Feedback module logs user ratings and comments, which are used to refine the system through offline training and model improvements. |

### [Achieving Non-Functional Requirements](#achieving-non-functional-requirements)

| Non-Functional Requirement | Techniques |
| --- | --- |
| **Scalability** | Load balancers distribute traffic across multiple servers. Model servers and vector database scale horizontally as demand grows. Redis caching reduces redundant processing. |
| **Reliability and availability** | ZooKeeper coordinates distributed components for fault tolerance. Redundant model servers and databases ensure continued operation in case of failures. Pub-sub enables decoupled communication for real-time updates and seamless failure recovery. |
| **Low latency** | Redis caching accelerates responses for frequently queried data. Vector database efficiently retrieves contextual information, reducing unnecessary recomputation. CDN reduces latency by caching and serving static content closer to users. |
| **Security and privacy** | API gateway enforces authentication, authorization, and rate limiting. Payment processing system ensures secure financial transactions. End-to-end encryption secures requests and responses from interception. MongoDB stores user data with encryption and access control. |

## [Conclusion](#conclusion)

In this chapter, we designed a ChatGPT-like system, covering the system architecture and detailed workflows. We examined how components like caching, content moderation, and distributed model hosting ensure scalability, reliability, and security.


