
# Design WhatsApp

## WhatsApp
WhatsApp is the most popular, simple, secure, and reliable messaging system that allows users to send text, voice, and multimedia messages.

WhatsApp is a widely used messaging application used by billions of people worldwide. Many users rely on WhatsApp for daily communication.

## Requirements

Let's scope the WhatsApp design by identifying the following requirements.

### Functional Requirements

- **Conversation:** Support one-on-one and group conversations.
- **Acknowledgment:** Provide message delivery status (sent, delivered, and read).
- **Sharing:** Enable sharing of media files, including images, videos, and audio.
- **Chat storage:** Persist messages for offline users until they are successfully delivered.
- **Push notifications:** Notify offline users of new messages when they reconnect.

> **Note:** "Offline" refers to users disconnected from the internet. The system sends push notifications when they reconnect. "Online" users are actively connected and receive real-time messages.

### Non-Functional Requirements

- **Low latency:** Deliver messages with minimal delay.
- **Consistency:** Deliver messages in order and ensure chat history is consistent across all user devices.
- **Availability:** Prioritize high availability, though trade-offs may be made to ensure consistency.
- **Security:** Implement end-to-end encryption so only communicating parties can access message content.
- **Scalability:** Support an increasing number of users and daily messages.

## Resource Estimation

For capacity planning, assume more than 2 billion daily active users (DAUs) who send more than 100 billion messages per day. Estimate the storage, bandwidth, and server capacity required to support this load.

### Storage Estimation

We assume 100 billion messages per day with an average size of 100 bytes. Servers retain messages for 30 days; if a user does not reconnect within this window, messages are permanently deleted.

$$100 \text{ billion/day} \times 100 \text{ Bytes} = 10 \text{ TB/day}$$

The 30-day storage requirement is:

$$30 \times 10 \text{ TB/day} = 300 \text{ TB/month}$$

Real-world requirements exceed this figure due to media files (larger than 100 Bytes), user information, message metadata (timestamps, IDs), and encryption keys. For simplicity, we will use 300 TB per month as our baseline.

### Bandwidth Estimation

Based on a daily storage intake of 10 TB, the required incoming bandwidth is approximately 926 Mb/s.

$$\frac{10 \text{ TB}}{86{,}400 \text{ sec}} \approx 926 \text{ Mb/s}$$

> **Note:** This estimate ignores media content (images, videos, documents), so the actual bandwidth requirement will be higher.

Outgoing bandwidth must match incoming bandwidth, as every message received by the system is delivered to a recipient.

### High-Level Estimates

| Type | Estimates |
| --- | --- |
| Total messages per day | 100 billion |
| Storage required per day | 10 TB |
| Storage for 30 days | 300 TB |
| Incoming data per second | 926 Mb/s |
| Outgoing data per second | 926 Mb/s |

### Number of Servers Estimation

WhatsApp handles approximately 10 million connections per server. This high density is achieved through extensive performance engineering, which involves optimizing the server kernel, networking libraries, and infrastructure configuration.

> **Note:** General-purpose servers can be optimized for specific tasks by carefully tuning the full software stack.

We estimate the number of servers as follows:

$$\text{No. of servers} = \frac{\text{Total connections per day}}{\text{No. of connections per server}} = \frac{2 \text{ billion}}{10 \text{ million}} = 200 \text{ servers}$$

Based on these estimates, we require **200 chat servers**.

## Building Blocks

We will use the following building blocks in our design:

- **Databases:** Store user and group metadata.
- **Blob storage:** Store multimedia content shared in messages.
- **CDN:** Deliver frequently accessed multimedia content.
- **Load balancer:** Distribute incoming requests across available servers.
- **Caches:** Store frequently accessed data to reduce latency.
- **Messaging queue:** Temporarily hold messages for offline users.

## High-Level Design

At a high level, the system uses a chat server to handle message exchange. Users connect to the server to send and receive messages. The server routes messages to the intended recipient and stores the messages in a database for durability and offline retrieval.

The communication flow between clients proceeds as follows:

1. User A and User B establish connections with the chat server.
2. User A sends a message to the chat server.
3. The chat server acknowledges receipt to User A.
4. The chat server attempts to deliver the message to User B and stores it in the database to ensure delivery if the receiver is offline.
5. User B sends an acknowledgment to the chat server.
6. The chat server notifies User A that the message was successfully delivered.
7. When User B reads the message, the application notifies the chat server.
8. The chat server notifies User A that the message has been read.

## API Design

WhatsApp exposes APIs for various features, including:

- Sending messages
- Retrieving messages
- Uploading media or documents
- Downloading media or documents
- Sending a location
- Sending a contact
- Creating a status

We will focus on the APIs for the core messaging features.

### Send Message

> **`POST /v1/messages`**

```
sendMessage(message_ID, sender_ID, reciever_ID, type, text, media_object, document)
```

This API sends a text message via a POST request to the `/messages` endpoint. User IDs are typically phone numbers.

| Parameter | Description |
| --- | --- |
| `message_ID` | A unique identifier of the message sent. |
| `sender_ID` | A unique identifier of the user who sends the message. |
| `reciever_ID` | A unique identifier of the user who receives the message. |
| `type` | The default message type is `text`. Represents whether the sender sends a media file or a document. |
| `text` | The text content to be sent as a message. |
| `media_object` | Defined based on the `type` parameter. Represents the media file to be sent. |
| `document` | Represents the document file to be sent. |

**Response:**

| Status Code | Description |
| --- | --- |
| `200 OK` | Message sent successfully. |
| `400 Bad Request` | Invalid or missing parameters. |
| `401 Unauthorized` | Authentication failed. |
| `429 Too Many Requests` | Rate limit exceeded. |

### Get Message

> **`GET /v1/messages/{user_Id}`**

```
getMessage(user_Id)
```

This API retrieves all unread messages when a user comes online after a period of inactivity.

| Parameter | Description |
| --- | --- |
| `user_Id` | A unique identifier representing the user who has to fetch all unread messages. |

**Response:**

| Status Code | Description |
| --- | --- |
| `200 OK` | Returns a list of unread messages. |
| `401 Unauthorized` | Authentication failed. |
| `404 Not Found` | No unread messages found. |

### Upload Media or Document

> **`POST /v1/media`**

```
uploadFile(file_type, file)
```

This API uploads media via a POST request to the `/v1/media` endpoint. A successful upload returns a media ID, which is then forwarded to the receiver. The size limit is 16 MB for media and 100 MB for documents.

| Parameter | Description |
| --- | --- |
| `file_type` | The type of file uploaded via the API call. |
| `file` | The file being uploaded via the API call. |

**Response:**

| Status Code | Description |
| --- | --- |
| `200 OK` | Returns the media ID of the uploaded file. |
| `400 Bad Request` | Invalid file type or missing file. |
| `413 Payload Too Large` | File exceeds the size limit (16 MB for media, 100 MB for documents). |

### Download Media or Document

> **`GET /v1/media/{file_id}`**

```
downloadFile(user_id, file_id)
```

This API retrieves documents or media using the file ID.

| Parameter | Description |
| --- | --- |
| `user_id` | A unique identifier of the user who will download the file. |
| `file_id` | A unique identifier of the file, generated during the `uploadFile()` API call. The client can find the `file_id` by providing the file name to the server. |

**Response:**

| Status Code | Description |
| --- | --- |
| `200 OK` | Returns the requested file. |
| `401 Unauthorized` | Authentication failed. |
| `404 Not Found` | File does not exist. |

## Detailed Design

### Connection with a WebSocket Server

Each active device establishes a WebSocket connection to the messaging infrastructure. The WebSocket servers maintain persistent connections with online users. To support billions of users, connections are distributed across a cluster of WebSocket servers. Each server maintains a session for every active connection. A **WebSocket manager** maintains the mapping between users, active connections, and servers. This mapping is stored in a **Redis cluster**.

> **Q: Why is WebSocket preferred over HTTP(S) for client-server communication?**
>
> HTTP(S) doesn't keep the connection open for the server to send frequent data to a client. With HTTP(S), a client constantly requests updates from the server (commonly called polling), which is resource-intensive and causes latency. WebSocket maintains a persistent connection between the client and server. It transfers data to the client immediately whenever it becomes available, providing a bidirectional connection used as a common solution to send asynchronous updates from a server to a client.

### Send or Receive Messages

The WebSocket manager maintains a mapping between each active user and the server connection handling that user. When a user reconnects to a different WebSocket server, the mapping is updated in the data store.

The WebSocket server communicates with the **Message service**, which manages message storage using a **Mnesia database cluster**. The Message service acts as an interface for other services to store and retrieve messages. It deletes messages after a configurable period and provides APIs to filter messages by user ID or message ID.

Assume that User A wants to send a message to User B, and both users are connected to different WebSocket servers. The system performs the following steps:

1. User A sends the message to their connected WebSocket server.
2. User A's server queries the WebSocket manager to locate User B's server. If User B is online, the manager returns the connection details.
3. Simultaneously, the message is sent to the message service and stored in the Mnesia database for FIFO processing. Once delivered, the message is deleted from the database.
4. User A's server forwards the message to User B's server, establishing communication.
5. If User B is offline, the message remains in the Mnesia database. When User B comes online, the system delivers pending messages via push notifications. Undelivered messages are permanently deleted after 30 days.

Both the sender and receiver rely on the WebSocket manager to locate each other's WebSocket server. During an ongoing conversation, this can result in frequent lookups. To reduce latency and avoid repeated lookups, each WebSocket server caches the following metadata:

- If both users are connected to the same server, the call to the WebSocket manager is avoided.
- It caches information of recent conversations about which user is connected to which WebSocket server.

### Send or Receive Media Files

While WebSocket servers handle text, they are too lightweight for heavy media payloads. The **asset service** handles sending and receiving media files.

The media transfer process consists of the following steps:

1. The device compresses and encrypts the media file.
2. The file is sent to the asset service and stored in blob storage. The service assigns a unique ID and returns it to the sender. It also hashes files to prevent duplication; if the content already exists, the service returns the existing ID.
3. The asset service sends the media ID to the receiver via the message service. The receiver uses this ID to download the content.
4. If specific content receives high traffic, the asset service loads it onto a CDN.

### Support for Group Messages

WebSocket servers track active users, not groups. Since group members may have mixed online statuses, the system uses three components to deliver group messages:

- **Group message handler**
- **Group message service**
- **Kafka**

Suppose User A sends a message to Group/A. The flow is as follows:

1. User A sends the message to the message service via their WebSocket server.
2. The message service publishes the message and group details to Kafka. Here, the group acts as a topic, with senders as producers and receivers as consumers.
3. The group service manages group metadata (members, ID, status, icon) using a **MySQL cluster** with geographically distributed replicas. A **Redis cache** reduces latency for frequent lookups.
4. The group message handler retrieves Group/A member data from the group service.
5. The handler delivers the message to each member following the standard WebSocket delivery process.

### Putting Everything Together

We have designed the core features of WhatsApp, including connection management, text and media messaging, group chat, and encryption.

