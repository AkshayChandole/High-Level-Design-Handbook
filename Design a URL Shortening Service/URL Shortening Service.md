
# URL Shortening Service

## Introduction

A URL shortening service creates a short alias, or short link, for a long URL. When a user clicks the short link, they are redirected to the original, longer address.

### Advantages
The main advantages of a URL shortening service are:
- They are easier to share and type, and fit better where character counts are limited, such as in text messages or on social media.
- They can provide a cleaner, more professional appearance in communications.

### Disadvantages
URL shortening services also have some drawbacks:
- Branding can be diluted when many companies use the same short domain name from a third-party service.
- There is a dependency on the third-party service. If it shuts down, all associated links will break.
- The service’s reliability and security reflect on your brand. Popular custom URLs may also be unavailable if they are already taken.

## Requirements for URL shortening design


### Functional requirements
* Short URL generation: Generate a unique short alias for a given URL.

* Redirection: Redirect a short link to its original URL.

* Custom short links: Allow users to create custom short links for their URLs.

* Deletion: Allow users to delete a short link with proper authorization.

* Update: Allow users to update the long URL associated with a short link, with proper authorization.

* Expiry time: Assign a default expiration time for short links, but allow users to set a custom expiration.

> Note: The system deletes expired short URLs after five years, even though they are not reused. Retaining them indefinitely would cause the datastore’s search index to grow continuously, which increases query time and overall latency.

### Non-functional requirements
* Availability: The system must be highly available. Any downtime will cause URL redirections to fail. This requires significant fault tolerance, as the service’s domain is part of every generated URL.

* Scalability: The system must scale horizontally to handle increasing traffic.

* Readability: Generated short links must be easy to read and type.

* Latency: Redirection must happen with very low latency for a good user experience.

* Unpredictability: Generated short URLs should not be guessable. Using sequential IDs would create a security risk, allowing others to find and access links.

> Note: Predictable short URLs create a security risk because attackers can identify patterns and attempt to guess private links. To mitigate this risk, the system avoids sequential IDs. Instead, it generates random identifiers. This approach produces unique short URLs that are difficult to predict.

<img width="1075" height="687" alt="image" src="https://github.com/user-attachments/assets/ea619efd-6383-485d-8a2c-97d402cea146" />

## Resource Estimation

Before designing the system, we need to estimate the required resources. These estimations are based on the following assumptions.

### Assumptions

- The ratio of shortening requests (writes) to redirection requests (reads) is **1:100**.
- The system handles **200 million** new URL shortening requests per month.
- A URL shortening entry requires **500 bytes** of storage.
- Each entry expires after **five years** unless explicitly deleted.
- The service has **100 million** daily active users (DAU).

### Storage Estimation

With 200 million new entries per month stored for 5 years, the total will be **12 billion**.

$$200 \text{ Million/month} \times 12 \text{ months/year} \times 5 \text{ years} = 12 \text{ Billion URL shortening requests}$$

At 500 Bytes per entry, the total storage required is **6 TB**:

$$12 \text{ Billion} \times 500 \text{ Bytes} = 6 \text{ TB}$$

### Query Rate Estimation

With a 1:100 write-to-read ratio, 200 million new URLs per month will result in **20 billion** redirection requests.

$$200 \text{ Million} \times 100 = 20 \text{ Billion}$$

To calculate the Queries Per Second (QPS), we first find the number of seconds in an average month (30.42 days):

$$30.42 \text{ days} \times 24 \text{ hours} \times 60 \text{ minutes} \times 60 \text{ seconds} = 2{,}628{,}288 \text{ seconds}$$

This gives us the following rates:

**Write QPS** (new URLs per second):

$$\frac{200 \text{ Million}}{2{,}628{,}288 \text{ seconds}} = 76 \text{ URLs/s}$$

**Read QPS** (redirections per second):

$$100 \times 76 \text{ URLs/s} = 7.6 \text{ K URLs/s}$$

### Bandwidth Estimation

**Shortening requests (writes):** For 76 new URLs per second, the total incoming data is:

$$76 \times 500 \text{ Bytes} \times 8 \text{ bits} = 304 \text{ Kbps}$$

**Redirection requests (reads):** For 7.6K redirections per second, the total outgoing data is:

$$7.6 \text{ K} \times 500 \text{ Bytes} \times 8 \text{ bits} = 30.4 \text{ Mbps}$$

### Memory Estimation

To estimate memory requirements for our cache, we can apply the **80/20 rule**, assuming that 20% of URLs generate 80% of the traffic.

First, we calculate the total daily redirection requests:

$$7.6 \text{ K} \times 3600 \text{ sec} \times 24 \text{ hrs} = 0.66 \text{ billion}$$

We will cache the top 20% of these daily requests, so the total memory needed is **66 GB**.

$$0.2 \times 0.66 \text{ Billion} \times 500 \text{ Bytes} = 66 \text{ GB}$$

### Number of Servers Estimation

To estimate the number of servers, we use the number of daily active users (DAU) and the request handling capacity of a single server. Assuming a peak load of 100 million requests per second (using DAU as a proxy) and a server capacity of 64,000 requests per second (RPS):

$$\text{Number of Servers} = \frac{\text{Number of requests/second}}{\text{RPS of server}}$$

$$\text{Servers needed at peak load} = \frac{100 \text{ million requests/sec}}{64{,}000 \text{ requests/sec per server}} \approx 1562.5 \approx 1.6 \text{ K servers}$$

### Summary of Estimations

| Type of Operation | Time Estimates |
| --- | --- |
| New URLs | 76/s |
| URL redirections | 7.6 K/s |
| Incoming data | 304 Kbps |
| Outgoing data | 30.4 Mbps |
| Storage for 5 years | 6 TB |
| Memory for cache | 66 GB |
| Servers | 1600 |

These estimations give us a tangible sense of the system's size.

## Building Blocks

With the estimations done, we can identify the key building blocks in our design:

- **Database(s):** Store the mapping between short and long URLs.
- **Sequencer:** Generate unique IDs for new short URLs.
- **Load balancers:** Distribute requests across available servers.
- **Caches:** Store frequently accessed URL mappings to reduce latency.
- **Rate limiters:** Prevent system abuse by limiting requests from a single source.

In addition to these core services, our application will require:

- **Servers** to run application logic and handle requests.
- **A Base-58 encoder** to convert numeric IDs into readable alphanumeric strings.

## System APIs

We can define REST API endpoints to expose the service's core functionality:

- Shortening a URL
- Redirecting a short URL
- Deleting a short URL

### Shortening a URL

> **`POST /api/v1/urls`**

To create a short URL, we use the `shortURL()` endpoint:

```
shortURL(api_dev_key, original_url, custom_alias=None, expiry_date=None)
```

| Parameter | Description |
| --- | --- |
| `api_dev_key` | A registered user account's unique identifier. This is useful in tracking a user's activity and allows the system to control the associated services accordingly. |
| `original_url` | The original long URL that needs to be shortened. |
| `custom_alias` | The optional key that the user defines as a custom short URL. |
| `expiry_date` | The optional expiration date for the shortened URL. |

**Response:**

| Status Code | Description |
| --- | --- |
| `201 Created` | Returns the new short URL. |
| `400 Bad Request` | Invalid or missing `original_url`. |
| `409 Conflict` | The requested `custom_alias` is already in use. |
| `429 Too Many Requests` | Rate limit exceeded. |

### Redirecting a Short URL

> **`GET /api/v1/urls/{url_key}`**

To redirect a short URL:

```
redirectURL(api_dev_key, url_key)
```

| Parameter | Description |
| --- | --- |
| `api_dev_key` | The registered user account's unique identifier. |
| `url_key` | The shortened URL against which we need to fetch the long URL from the database. |

**Response:**

| Status Code | Description |
| --- | --- |
| `301 Moved Permanently` | Redirects the user to the original URL (cacheable by the browser). |
| `302 Found` | Redirects the user to the original URL (not cached, useful for analytics tracking). |
| `404 Not Found` | The short URL does not exist. |
| `410 Gone` | The short URL has expired. |

> **Note:** Use `301` if the mapping is permanent and caching is acceptable. Use `302` if you need to track click analytics, since the browser will call the service on every request.

### Deleting a Short URL

> **`DELETE /api/v1/urls/{url_key}`**

To delete a short URL:

```
deleteURL(api_dev_key, url_key)
```

| Parameter | Description |
| --- | --- |
| `api_dev_key` | The registered user account's unique identifier. |
| `url_key` | The shortened URL that should be deleted from the system. |

**Response:**

| Status Code | Description |
| --- | --- |
| `204 No Content` | The short URL was successfully deleted. |
| `401 Unauthorized` | The user is not authorized to delete this URL. |
| `404 Not Found` | The short URL does not exist. |

## Design

Let's discuss the main design components required for our URL shortening service. Our design depends on each part's functionality and progressively combines them to achieve the different workflows mentioned in the functional requirements.

### Components

#### Database

A URL shortening service is read-heavy and requires horizontally scalable storage. We need to store:

- User details
- Mappings of long URLs to their corresponding short URLs

As the URL mappings are independent and don't have complex relationships, a **NoSQL database** is a good fit. **MongoDB** is a strong candidate because:

- It uses a leader-follower protocol, enabling the use of replicas for heavy read workloads.
- It ensures atomicity in concurrent write operations and returns duplicate-key errors to prevent collisions.

#### Short URL Generator

The short URL generator consists of two main parts:

1. **A sequencer** to generate unique IDs
2. **A Base-58 encoder** to enhance the readability of the short URL

Our sequencer generates a unique 64-bit numeric ID (base-10). To create a more readable URL, we use a Base-58 encoder to convert this numeric ID into an alphanumeric string.

#### Other Building Blocks

- **Load balancing:** The system can use Global Server Load Balancing (GSLB) alongside local load balancers to improve availability. Because there is typically a delay between URL creation and the first access request, eventual consistency across geographically distributed databases is acceptable.

- **Cache:** Memcached is a good choice for our cache because it is simple, horizontally scalable, and meets our needs. To minimize latency, we will use a data-center-specific caching layer rather than a global one.

- **Rate limiter:** To prevent abuse, we will rate-limit users based on their unique `api_dev_key`. The fixed-window counter algorithm is sufficient for this purpose, allowing a fixed number of operations per user within a specific timeframe.

## Design diagram
The diagram below shows the system’s high-level design.

<img width="1171" height="590" alt="image" src="https://github.com/user-attachments/assets/27fa9957-1183-4953-8dbf-5e1f956be485" />


### Workflow

Let's analyze the system in-depth and how the individual pieces fit together to provide the overall functionality. Given the functional requirements, the workflow for the abstract design above would be as follows.

- **Shortening:** When a user requests a shortened URL, the application server forwards the request to the Short URL Generator (SUG). The SUG creates a short link, which is returned to the user and stored in the database.

- **Redirection:** When a redirection request is received, the application server first checks the cache. If the URL is not in the cache, it queries the database. If a record is found, the server redirects the user to the original long URL.

- **Deletion:** An authorized user can request the deletion of a short URL. The application server validates the request and removes the corresponding entry from the database. Deletion can also be triggered automatically when a URL expires.

- **Custom short links:** Users can request custom short links. The system first validates the format of the requested link (e.g., a maximum of 11 characters). The system then checks the database to determine whether the custom short URL is available. If the identifier is available, the system maps the custom short URL to the long URL. If not, the request returns an error.

### Managing Unique ID Allocation

When a custom short URL is created, the system must ensure its underlying unique ID is not reused.

The server decodes the custom short URL back to its base-10 equivalent and marks that ID as "used" in the database. This ensures no two long URLs can map to the same short URL.

This approach improves availability by storing the used/unused ID state in a scalable NoSQL database rather than in memory.

<img width="1032" height="436" alt="image" src="https://github.com/user-attachments/assets/1c6af8d2-bbb1-4fe8-991b-e5b42ce5b162" />


The process is simple:

1. Newly generated IDs are added to an "unused" list.
2. Once an ID is assigned to a URL, it is moved to the "used" list.

Because Base-58 encoding provides a one-to-one mapping to base-10 IDs, it prevents collisions.

### URL Encoding and Readability

Let's clarify two key aspects of the Short URL Generator (SUG):

- How does encoding improve the readability of the short URL?
- How are the sequencer and the Base-58 encoder related?

#### Why Use Encoding

Our sequencer generates a 64-bit ID in base-10, which can be converted to a base-64 short URL. Base-64 is the most common encoding for alphanumeric strings. However, there are some inherent issues with sticking to base-64 for this design problem: the generated short URL might be hard to read due to look-alike characters. Characters like `O` (capital o) and `0` (zero), `I` (capital I) and `l` (lowercase L) can be confused, while characters like `+` and `/` should be avoided because of other system-dependent encodings.

So, we remove six characters and use **Base-58** instead of base-64 (which includes A-Z, a-z, 0-9, + and /) for enhanced readability.

#### Base-58 Character Set

| Value | Char | Value | Char | Value | Char | Value | Char |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | 1 | 15 | G | 30 | X | 45 | n |
| 1 | 2 | 16 | H | 31 | Y | 46 | o |
| 2 | 3 | 17 | J | 32 | Z | 47 | p |
| 3 | 4 | 18 | K | 33 | a | 48 | q |
| 4 | 5 | 19 | L | 34 | b | 49 | r |
| 5 | 6 | 20 | M | 35 | c | 50 | s |
| 6 | 7 | 21 | N | 36 | d | 51 | t |
| 7 | 8 | 22 | P | 37 | e | 52 | u |
| 8 | 9 | 23 | Q | 38 | f | 53 | v |
| 9 | A | 24 | R | 39 | g | 54 | w |
| 10 | B | 25 | S | 40 | h | 55 | x |
| 11 | C | 26 | T | 41 | i | 56 | y |
| 12 | D | 27 | U | 42 | j | 57 | z |
| 13 | E | 28 | V | 43 | k | | |
| 14 | F | 29 | W | 44 | m | | |

> **Note:** The omitted characters are `0`, `O`, `I`, and `l` to avoid confusion with visually similar characters.

#### Converting Base-10 to Base-58

To convert a base-10 ID to a Base-58 string, we repeatedly use the modulus operator.

**Process:** Continuously divide the base-10 number by 58 and record the remainder. The sequence of remainders, mapped to our Base-58 character set, forms the new string. The last remainder becomes the first character of the string.

**Example:** Let's assume the unique ID is `2468135791013`.

`2468135791013 % 58 = 17`  
`42554065362 % 58 = 6`  
`733690782 % 58 = 4`  
`12649841 % 58 = 41`  
`218100 % 58 = 20`  
`3760 % 58 = 48`  
`64 % 58 = 6`  
`1 % 58 = 1`

Writing the remainders in reverse order of calculation gives us the following indexes:

```
Base-58 indices = [1] [6] [48] [20] [41] [4] [6] [17]
```

Using the character table, we can map these indexes to characters:

```
Base-58 = 27qMi57J
```

> **Note:** Both the base-10 numeric IDs and base-58 alphanumeric IDs represent the same 64-bit value.

#### Converting Base-58 to Base-10

Decoding a Base-58 string back to a base-10 number is also necessary, particularly for handling custom URLs.

**Process:** To convert from Base-58 to base-10, multiply each character's value by 58 raised to the power of its position (starting from 0 on the right). The sum of these products is the final base-10 number.

**Example:** Let's decode the previous example to see how it works.

Base-58: `27qMi57J`

$$2_{58} = 1 \times 58^7 = 2{,}207{,}984{,}167{,}552$$

$$7_{58} = 6 \times 58^6 = 228{,}412{,}155{,}264$$

$$q_{58} = 48 \times 58^5 = 31{,}505{,}124{,}864$$

$$M_{58} = 20 \times 58^4 = 226{,}329{,}920$$

$$i_{58} = 41 \times 58^3 = 7{,}999{,}592$$

$$5_{58} = 4 \times 58^2 = 13{,}456$$

$$7_{58} = 6 \times 58^1 = 348$$

$$J_{58} = 17 \times 58^0 = 17$$

Summing all values:

$$\text{Base-10} = 17 + 348 + 13{,}456 + 7{,}999{,}592 + 226{,}329{,}920 + 31{,}505{,}124{,}864 + 228{,}412{,}155{,}264 + 2{,}207{,}984{,}167{,}552$$

$$\text{Base-10} = 2{,}468{,}135{,}791{,}013$$

This is the same unique base-10 ID from the previous example.

### The Scope of the Short URL Generator

The design of our short URL generator is guided by the following constraints:

- The generated short URL must contain only alphanumeric characters.
- None of the characters should be visually ambiguous.
- The minimum default length of the generated short URL should be six characters.

These constraints determine the range of unique IDs we can use:

**Starting range:** Our 64-bit sequencer can generate numbers from $1 \to (2^{64} - 1)$. To ensure short URLs have a minimum length of six characters, we will only use IDs with at least 10 digits (i.e., starting from 1 billion).

**Ending point:** The maximum length of a short URL is determined by the largest 64-bit number. We can calculate the number of digits a 64-bit value can represent in a given base:

- The number of bits to represent one digit in base-n is given by $\log_2 n$.

$$\text{Number of digits} = \frac{\text{Total bits available}}{\text{Number of bits to represent one digit}}$$

Applying this to base-10 and Base-58:

**Base-10:**

$$\text{Bits per digit} = \log_2 10 = 3.32$$

$$\text{Total digits in a 64-bit ID} = \frac{64}{3.32} = 19.27 \approx 20 \text{ digits}$$

**Base-58:**

$$\text{Bits per digit} = \log_2 58 = 5.85$$

$$\text{Total digits in a 64-bit ID} = \frac{64}{5.85} = 10.9 \approx 11 \text{ characters}$$

**Maximum digits:** The calculations show that a 64-bit ID can have up to 20 decimal digits, which corresponds to a Base-58 encoded string of up to 11 characters.

### The Sequencer's Lifetime

We can estimate the lifetime of our sequencer based on two factors:

- Total numbers available in the sequencer = $2^{64} - 10^9$ (starting from 1 billion)
- Number of requests per year = $200 \text{ Million/month} \times 12 = 2.4 \text{ Billion}$ (from our earlier estimations)

Using these values:

$$\text{Lifetime} = \frac{\text{Total numbers available}}{\text{Yearly requests}} = \frac{2^{64} - 10^9}{2.4 \text{ Billion}} = 7{,}686{,}143{,}363.63 \text{ years}$$

Therefore, our service can run for a very long time before the range depletes.

## Requirements Compliance

Finally, we evaluate our design against the non-functional requirements.

### Availability

Our design must be highly available for both generating and redirecting URLs.

- Our core components (databases, caches, servers) support replication, which provides high availability and fault tolerance. The unique ID generation process also relies on a replicable database, which prevents a single point of failure.
- For disaster recovery, we can perform daily backups of our storage to a service like Amazon S3. In a worst-case scenario, this might result in the loss of URLs created since the last backup.
- Our design uses Global Server Load Balancing (GSLB) to intelligently distribute traffic across global servers, especially during regional failures.
- We also use rate limiters between clients and web servers to protect against DoS attacks and ensure fair usage. This protects system resources and ensures a smooth influx of traffic.

### Scalability

Our design is horizontally scalable. The database can be sharded, and we can use consistent hashing to evenly distribute load across both the application and database servers.

Our choice of a NoSQL database like MongoDB is key to this scalability for several reasons:

- NoSQL databases offer schema flexibility. This is useful because we don't always store a UserID for every URL, such as for anonymous users.
- NoSQL databases are generally designed for horizontal scaling and automatic data distribution, which is simpler to manage at a large scale than sharding a relational database.

Finally, the 64-bit sequencer provides a vast number of unique IDs, ensuring the system can scale for many years.

### Readability

Using Base-58 instead of Base-64 directly addresses the readability requirement in two ways:

- **Distinguishable characters:** It eliminates visually ambiguous characters like `0` (zero) and `O` (capital o), or `I` (capital I) and `l` (lowercase L).
- **URL-safe characters:** It avoids non-alphanumeric characters like `+` and `/`, which can cause encoding issues in URLs.

This makes the generated URLs less error-prone and easier for users to handle.

### Latency

Several design choices contribute to low latency:

- The URL generation process, including encoding, is computationally fast and adds negligible delay.
- The system is read-heavy. We chose MongoDB for its high throughput and low read latency.
- There is typically a delay before a newly created URL is used, which allows time for data to replicate across distributed databases without impacting the user experience.
- A distributed cache for popular URLs significantly reduces redirection time.

### Unpredictability

To make short URLs unpredictable and secure, we must avoid sequential IDs.

The sequencer assigns ranges of unique IDs to different servers. If a server issued IDs sequentially, the resulting short URLs would be predictable. To avoid this, the server selects an ID at random from its assigned range for each new URL. This makes the generated short URLs harder to predict.

### Summary of Non-Functional Requirements Compliance

| Requirement | Techniques |
| --- | --- |
| **Availability** | Daily backups to Amazon S3 for storage and cache servers. Global Server Load Balancing (GSLB) to handle traffic. Rate limiters to limit each user's resource allocation. |
| **Scalability** | Horizontal sharding of the database. Data distribution based on consistent hashing. MongoDB as the NoSQL database. |
| **Readability** | Base-58 encoder for short URL generation. Removal of non-alphanumeric characters. Removal of look-alike characters. |
| **Latency** | Unnoticeable delay in the overall operation. MongoDB for low latency and high throughput reads. Distributed cache to minimize service delays. |
| **Unpredictability** | Randomly selecting and associating an ID to each request from the pool of unused and readily available unique IDs. |

<img width="858" height="639" alt="image" src="https://github.com/user-attachments/assets/37d00294-bf17-4041-8e5c-45c5d0ca6aa7" />


## Conclusion

The proposed URL shortening service provides several core features: support for dynamic short URL ranges and short URLs that remain human-readable. Security can be improved by adding a salt to the unique ID before encoding. This makes the resulting short URLs harder to predict.
