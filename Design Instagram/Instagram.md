# System Design: Instagram

## [What is Instagram?](#what-is-instagram)

Instagram is a social networking service for sharing photos and short videos. Users upload media with captions, hashtags, and geotags that improve discoverability in search and tag feeds. Posts are delivered to followers' feeds. Public posts can be discovered through hashtags and location feeds, while private profiles limit visibility to approved followers.

<img width="268" height="402" alt="image" src="https://github.com/user-attachments/assets/a23886f4-99d2-4750-b33a-209d9d5c7411" />


> **Note:** As the user base grows, Instagram must continuously optimize its backend to handle millions of daily active users. Efficiently managing growth and predicting resource requirements are essential for scalability.

Expanding user counts require increased server, database, and CDN capacity.

> **Did you know?** Instagram relies on technologies like Redis, Cassandra, and distributed file storage to maintain high availability and low latency for millions of users.

## [Requirements](#requirements)

### [Functional Requirements](#functional-requirements)

- **Post photos and videos:** Users can post photos and videos on Instagram.
- **Follow and unfollow users:** Users can follow and unfollow other users on Instagram.
- **Like or dislike posts:** Users can like or dislike posts of the accounts they follow.
- **Search photos and videos:** Users can search photos and videos based on captions and location.
- **Generate news feed:** Users can view the news feed consisting of photos and videos (in chronological order) from all the users they follow. Users can also view suggested and promoted photos in their news feed.

### [Non-Functional Requirements](#non-functional-requirements)

- **Scalability:** The system should be scalable to handle millions of users in terms of computational resources and storage.
- **Latency:** The latency to generate a news feed should be low.
- **Availability:** The system should be highly available.
- **Durability:** Any uploaded content (photos and videos) should never get lost.
- **Consistency:** We can compromise a little on consistency. It is acceptable if the content (photos or videos) takes time to show in followers' feeds located in a distant region.
- **Reliability:** The system must be able to tolerate hardware and software failures.

## [Resource Estimation](#resource-estimation)

The system is read-heavy because users browse feeds far more often than they create posts. Design priorities include storage efficiency and low-latency feed retrieval. Instagram supports roughly 1 billion users who share about 95 million posts per day.

We make the following assumptions:

- **Total users:** 1 billion (500 million daily active users)
- **Daily uploads:** 60 million photos and 35 million videos
- **File sizes:** Max 3 MB per photo, 150 MB per video
- **Traffic:** Average 20 requests per user/day

### [Storage Estimation](#storage-estimation)

Daily storage requirements:

$$60 \text{ million photos/day} \times 3 \text{ MB} = 180 \text{ TB/day}$$

$$35 \text{ million videos/day} \times 150 \text{ MB} = 5{,}250 \text{ TB/day}$$

$$\text{Total daily content size} = 180 + 5{,}250 = 5{,}430 \text{ TB/day}$$

Annual storage requirement:

$$5{,}430 \text{ TB/day} \times 365 \text{ days/year} = 1{,}981{,}950 \text{ TB} = 1{,}981.95 \text{ PB}$$

For simplicity, we omit metadata (comments, user info) and focus on the 5,430 TB/day media requirement.

### [Bandwidth Estimation](#bandwidth-estimation)

Incoming data (5,430 TB/day) translates to:

$$\frac{5{,}430 \text{ TB}}{86{,}400 \text{ sec}} \approx 62.84 \text{ GB/s} \approx 502.8 \text{ Gbps}$$

Assuming a read-to-write ratio of $100:1$, outgoing bandwidth is 100x incoming:

- **Incoming bandwidth** $\approx 502.8 \text{ Gbps}$
- **Required outgoing bandwidth** $\approx 100 \times 502.8 \text{ Gbps} \approx 50.28 \text{ Tbps}$

Outbound bandwidth usage is high due to frequent media delivery. Media compression can reduce file sizes before delivery. Content is cached near users through CDNs and edge caches located at IXPs and ISP networks to reduce latency during content delivery.

### [Server Count](#server-count)

Server capacity is estimated using two scenarios: a uniform request distribution (best case) and peak load conditions (worst case). These scenarios establish the lower and upper bounds for required server capacity. These bounds can be refined by adjusting traffic assumptions, for example, assuming 20% of daily requests occur during peak concurrency.

#### [Server Count Assuming Uniform Request Distribution](#server-count-assuming-uniform-request-distribution)

Assume 500 million DAUs and a server capacity of 64,000 requests per second (RPS).

- Requests per user/day = $20$
- Total requests/day = $500 \text{ million} \times 20$

$$\text{Requests/second} = \frac{500 \text{ million} \times 20}{24 \times 60 \times 60} \approx 116 \text{ K}$$

Server calculation:

$$\text{Servers needed} = \frac{\text{Number of requests/second}}{\text{RPS of server}} = \frac{116 \text{ K}}{64{,}000} \approx 2 \text{ servers}$$

Under uniform distribution, approximately **2 servers** are required.

#### [Server Count Assuming Peak Load](#server-count-assuming-peak-load)

Using our assumption of Daily Active Users as a proxy for peak concurrent requests (worst-case), we assume 500 million requests per second. The server count is calculated as:

$$\text{Servers needed at peak load} = \frac{\text{Number of requests/second}}{\text{RPS of server}} = \frac{500 \text{ million}}{64{,}000} = 7{,}812.5 \approx 8 \text{ K servers}$$

> **Note:** Concurrent requests drive server needs more than average load.

## [Building Blocks](#building-blocks)

We will use the following building blocks in our design:

<img width="791" height="258" alt="image" src="https://github.com/user-attachments/assets/e1d6480c-6fa5-4a61-aee1-0de5071990e5" />


- **Load balancer:** Distributes requests across available servers.
- **Database:** Stores user metadata and relationships.
- **Blob storage:** Stores media content like photos and videos.
- **Task scheduler:** Manages asynchronous tasks, such as cleaning up expired content.
- **Cache:** Stores frequently accessed content.
- **CDN:** Delivers content to end-users, reducing latency and server load.


# [High-Level Design](#high-level-design)

The system must support uploading, viewing, and searching media. Additionally, users can follow one another. This requires a robust storage layer to persist media and a retrieval mechanism to fetch data on demand.

<img width="682" height="560" alt="image" src="https://github.com/user-attachments/assets/c41382ac-5c65-4284-9364-eb5f2f01f181" />


## [API Design](#api-design)

We will implement REST APIs for the following features:

- Post photos and videos
- Follow and unfollow users
- Like or dislike posts
- Search photos and videos
- Generate a news feed

All calls require a `userID` to identify the requester. We will focus on unique parameters for each endpoint below.

### [Post Photos or Videos](#post-photos-or-videos)

> **`POST /v1/media`**

```
postMedia(userID, media_type, media_file, list_of_hashtags, caption)
```

| Parameter | Description |
| --- | --- |
| `media_type` | Indicates the type of media (photo or video) in a post. |
| `media_file` | Holds the media file (photo or video) of a post. |
| `list_of_hashtags` | Represents all hashtags (maximum limit 30) in a post. |
| `caption` | Text content (maximum limit 2,200 characters) in a user's post. |

**Response:**

| Status Code | Description |
| --- | --- |
| `201 Created` | Media posted successfully. |
| `400 Bad Request` | Invalid or missing parameters. |
| `401 Unauthorized` | Authentication failed. |
| `413 Payload Too Large` | Media file exceeds size limit. |

### [Follow and Unfollow Users](#follow-and-unfollow-users)

> **`POST /v1/users/{target_userID}/follow`**

```
followUser(userID, target_userID)
```

| Parameter | Description |
| --- | --- |
| `target_userID` | Indicates the user to be followed. |

> **`DELETE /v1/users/{target_userID}/follow`**

The `/unfollowUser` API accepts the same parameters.

**Response:**

| Status Code | Description |
| --- | --- |
| `200 OK` | Follow/unfollow action successful. |
| `401 Unauthorized` | Authentication failed. |
| `404 Not Found` | Target user does not exist. |

### [Like or Dislike Posts](#like-or-dislike-posts)

> **`POST /v1/posts/{post_id}/like`**

```
likePost(userID, post_id)
```

| Parameter | Description |
| --- | --- |
| `post_id` | Specifies the post's unique ID. |

> **`DELETE /v1/posts/{post_id}/like`**

Similarly, the `/dislike`  API removes a like.

**Response:**

| Status Code | Description |
| --- | --- |
| `200 OK` | Like/dislike action successful. |
| `401 Unauthorized` | Authentication failed. |
| `404 Not Found` | Post does not exist. |

### [Search Photos or Videos](#search-photos-or-videos)

> **`GET /v1/search?keyword={keyword}`**

```
searchPhotos(userID, keyword)
```

| Parameter | Description |
| --- | --- |
| `keyword` | The string (username, hashtag, or place) typed by the user in the search bar. |

**Response:**

| Status Code | Description |
| --- | --- |
| `200 OK` | Returns paginated search results. |
| `400 Bad Request` | Missing or invalid keyword. |

> **Note:** Instagram ranks search results by reach (likes and views). For example, a search for "London" displays posts in descending order of popularity. Results are paginated to support infinite scrolling.

### [Generate News Feed](#generate-news-feed)

> **`GET /v1/feed?generate_timeline={timestamp}`**

```
viewNewsfeed(userID, generate_timeline)
```

| Parameter | Description |
| --- | --- |
| `generate_timeline` | Indicates the time when a user requests news feed generation. Instagram shows posts not seen by the user between the last request and the current request. |

**Response:**

| Status Code | Description |
| --- | --- |
| `200 OK` | Returns the user's news feed. |
| `401 Unauthorized` | Authentication failed. |

## [Storage Schema](#storage-schema)

We must define a data model that supports our requirements.

### [Relational or Non-Relational Database?](#relational-or-non-relational-database)

Choosing the right database is critical. Our data is inherently relational, requiring strict order (chronological posts) and durability. SQL databases are ideal for these requirements, allowing us to efficiently query relationships, such as fetching followers or retrieving images by user ID.

Therefore, we will use a **relational database** for metadata storage.

### [Define Tables](#define-tables)

The schema requires the following tables:

- **Users:** Stores metadata including ID, name, email, bio, location, creation date, and login timestamps.
- **Followers:** Maps user relationships. Instagram uses a unidirectional model: if User A follows User B, User B does not automatically follow User A.
- **Photos:** Stores all photo-related information such as ID, location, caption, time of creation, and so on. Includes a `user_id` as a foreign key from the Users table.
- **Videos:** Stores video metadata (ID, location, caption, creation time). Includes a `user_id` as a foreign key.



> **Q: Where should we store the photos and videos?**
>
> We'll store the photos and videos in a blob store (like S3) and save the path to each photo or video in the table, as it's more efficient to store larger data in a distributed storage.

<img width="1218" height="420" alt="image" src="https://github.com/user-attachments/assets/1448fc09-453f-4f40-bced-12a657a14d1b" />


### [Data Estimation](#data-estimation)
Let’s estimate the amount of data stored in each table.

| Table | Estimated Row Size | Record Count | Total Storage |
| --- | --- | --- | --- |
| Users | ~222 bytes | 500 million | ~111,000 MB |
| Followers | ~8 bytes | 250 followers/user | ~2,000 MB |
| Photos | ~394 bytes | 60 million/day | ~23,640 MB |
| Videos | ~394 bytes | 35 million/day | ~13,790 MB |

> **Note:** Most modern services use both SQL and NoSQL stores. Instagram officially uses a combination of SQL (PostgreSQL) and NoSQL (Cassandra) databases. Loosely structured data, like timeline generation, is usually stored in NoSQL, while relational data is saved in SQL-based storage.

# [Detailed Design of Instagram](#detailed-design-of-instagram)

## [Additional Components](#additional-components)

We expand the design with the following components:

- **Load balancer:** Distributes incoming user requests.
- **Application servers:** Host the service logic for end users.
- **Relational database:** Stores metadata and user information.
- **Blob storage:** Stores media files (photos and videos).

<img width="809" height="468" alt="image" src="https://github.com/user-attachments/assets/ce16981a-a889-462c-8553-026f27e6cd66" />


## [Upload, View, and Search a Photo](#upload-view-and-search-a-photo)

- **Upload flow:** The client sends a photo to the load balancer, which routes it to an application server. The server stores the photo in the database (blob storage) and sends a success or error notification to the user.
- **View and search flow:** When a client requests to view or search for a photo, the application server fetches the matching content from the database and serves it to the user.
- **Service splitting:** Since read requests (views) significantly outnumber write requests (uploads), we split the architecture into separate read and write services. This allows us to scale the read infrastructure independently to handle high traffic.
- **Optimization:** We introduce caching to speed up reads for millions of users. We also implement lazy loading. This minimizes latency and bandwidth usage by only loading content as the user scrolls.

<img width="977" height="716" alt="image" src="https://github.com/user-attachments/assets/22e231db-ce02-4f83-ab6b-2da97f7f5701" />

## [Generate a Timeline](#generate-a-timeline)

Generating a user-specific timeline is a core feature. We analyze three strategies to determine the best approach.

### [The Pull Approach](#the-pull-approach)

This approach is known as **"fan-out on load."** When a user opens Instagram, the system generates the user's timeline. The service retrieves the accounts the user follows, fetches their recent posts, and aggregates them into a timeline for display.

**Drawback:** This approach creates high latency because the timeline is generated on-demand every time the app opens. We can mitigate this by pre-generating timelines offline for active users.

<img width="929" height="597" alt="image" src="https://github.com/user-attachments/assets/a25019e6-b386-4bc1-b9fa-ea845526d50d" />

<img width="948" height="592" alt="image" src="https://github.com/user-attachments/assets/77180a5a-0490-4cc5-8f91-197ee4e62239" />


> **Q: What are the shortcomings of the pull approach?**
>
> Instagram operates as a read-heavy system. Many users do not create posts and primarily consume content from others. As a result, many requests to fetch recent posts from followed accounts return no new content. The system should therefore be optimized for read-heavy workloads rather than write-heavy ones.

### [The Push Approach](#the-push-approach)

The push approach is also known as **"fan-out on write."** When a user creates a post, the system distributes it to the timelines of all followers. Unlike the pull approach, follower timelines are precomputed at write time.

**Benefit:** Read latency is minimal because the data is ready when the user opens the app. It also eliminates wasted read requests for users who haven't posted recently.

<img width="925" height="633" alt="image" src="https://github.com/user-attachments/assets/e203517e-8f29-40a0-98a4-b91bdcad44c3" />


> **Q: What are the shortcomings of the push approach?**
>
> Consider an account that belongs to a celebrity, like Cristiano Ronaldo, who has over 400 million followers. If he posts a photo or a video, we will push the links of the photo/video to 400 million+ users, which is inefficient.

### [Hybrid Approach](#hybrid-approach)

To balance the trade-offs, we categorize users based on follower count:

- **Push-based (standard users):** Users with a few hundred or thousand followers. Their posts are pushed to followers' timelines immediately.
- **Pull-based (celebrities):** Users with hundreds of thousands or millions of followers. Their posts are not pushed. Instead, followers fetch these posts on-demand when loading their timeline.

The timeline service aggregates data by pulling from celebrity accounts and reading the pre-pushed feed for standard accounts.

<img width="756" height="586" alt="image" src="https://github.com/user-attachments/assets/e2dc29a0-051f-4a80-9a0c-8118f5837d58" />


### [Timeline Storage](#timeline-storage)

We store the user's timeline in a key-value store. The key is the `userID`, and the value is the timeline content (a list of post links). If the value size exceeds the store's limit (e.g., a few MBs), we can store the actual timeline data in a blob store and keep a reference link in the key-value store.

## [Stories](#stories)

We can add a "Story" feature where photos expire after 24 hours. We implement this by storing a timestamp with the story entry. A task scheduler or TTL mechanism automatically deletes entries that exceed the 24-hour limit.

## [Finalized Design](#finalized-design)

We integrate a CDN (content delivery network) to serve static media like images and videos. This ensures high availability and low latency for millions of concurrent users.

The load balancer routes read requests to the nearest CDN edge server. If the content is not found, the request forwards to the Read Application Server.

<img width="1068" height="788" alt="image" src="https://github.com/user-attachments/assets/587b7603-828f-4bdf-aad2-deb8ae6d8233" />


> **Q: How can we count millions of interactions (like or view) on a celebrity post?**
>
> We can use sharded counters to count the number of multiple interactions for a particular user. Each counter has several shards distributed across various edge servers to reduce the load on the application server and latency. Users nearest to the edge server get the updated count frequently on a specific post compared to those in distant regions.

## [Requirements Compliance](#requirements-compliance)

We evaluate the design against our non-functional requirements:

- **Scalability:** Separating read/write services and sharding databases allows us to handle increasing request volumes and user data.
- **Latency:** Caching and CDNs significantly reduce content retrieval time.
- **Availability:** Replicating storage and databases across geographical regions ensures the system remains accessible even during outages.
- **Durability:** Persistent storage with automated backups ensures uploaded content is never lost.
- **Consistency:** Blob stores and databases maintain data integrity across the system.
- **Reliability:** Replication and load balancing prevent single points of failure.

## [Conclusion](#conclusion)

This design demonstrates how connecting scalable, fault-tolerant building blocks enables us to solve complex problems like efficient timeline generation. By offloading infrastructure concerns to these components, we can focus on optimizing specific use cases.
