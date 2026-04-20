# [System Design: Retrieval-Augmented Generation (RAG) systems](#system-design-retrieval-augmented-generation-rag-systems)

## [Introduction to RAG](#introduction-to-rag)

Retrieval-augmented generation (RAG) is an advanced AI technique that enhances the capabilities of large language models (LLMs) by integrating external information retrieval mechanisms. Traditional LLM-based systems are limited to the data they were trained on, which can lead to outdated or incomplete responses. RAG addresses this limitation by allowing models to fetch and incorporate up-to-date information from external sources during the response generation.

LLMs rely on patterns learned from vast datasets but do not have real-time access to up-to-date information. This can result in responses not aligned with the most current data or context. RAG mitigates this issue by retrieving relevant information at query time, ensuring the generated content is accurate and contextually relevant.

RAG-based systems have found applications across various domains, including:

- **Chatbots and virtual assistants:** Enhancing interactions by responding based on the latest information.

- **Question answering systems:** Delivering precise answers by consulting external knowledge bases.

- **Document summarization:** Generating concise summaries that incorporate the given documents.

RAG-based systems integrate retrieval mechanisms with generative models to connect static knowledge with dynamic, real-time information, which improves the reliability and contextual relevance of AI applications.

## [Requirements](#requirements)

The following are essential requirements for a RAG system considered during system design.

### [Functional requirements](#functional-requirements)

- **Understanding user query intent:** The system should accurately interpret the semantic meaning of a user's query, even if it is phrased ambiguously or informally.

- **Ingesting and indexing documents:** The system should accept documents in multiple formats, extract their content, and store them in a searchable index for retrieval.

- **Retrieving relevant knowledge:** The system should find and return the most contextually relevant document passages from the knowledge base in response to a user's query.

- **Generating grounded, accurate responses:** The system should produce natural language responses that are directly based on retrieved documents, rather than relying solely on the model's trained knowledge.

### [Nonfunctional requirements](#nonfunctional-requirements)

- **Low latency:** The system should return a response to the user within an acceptable time frame, even under high query load.

- **Scalability:** The system should handle a growing number of users and documents without degrading in performance or accuracy.

- **High availability:** The system should remain operational at all times, with no single point of failure across its subsystems.

- **Data freshness:** The system should allow new documents to be ingested and made searchable without requiring a full re-indexing of the knowledge base.

- **Content safety:** The system should detect and filter out harmful, inaccurate, or policy-violating content before responses reach the user.

- **Fault tolerance:** The system should gracefully handle failures in individual components, such as the retrieval service or LLM, without bringing down the entire pipeline.

## [Core Components of a RAG System](#core-components-of-a-rag-system)

A RAG system enhances LLMs by integrating external information retrieval mechanisms, enabling more accurate and contextually relevant responses. The core components of a RAG system include:

- **Data indexing:** Converting data into vector embeddings and storing them in a vector database for efficient retrieval.

- **Retriever mechanism:** Fetching relevant documents using similarity metrics and retrieval algorithms.

- **Augmentation process:** Combining retrieved documents with user queries to form augmented prompts.

- **Generator module:** Utilizing the augmented prompts to generate informed responses.

These components enable the RAG-based system to retrieve and integrate external information, which improves LLM performance.

## [Detailed Workflow of a RAG System](#detailed-workflow-of-a-rag-system)

In a RAG-based system, several key processes work together to integrate external information retrieval with an LLM. This integration enables the system to generate accurate, contextually relevant responses using up-to-date data. The system includes the following core processes: data indexing, retrieval, augmentation, and response generation.

### [Data Indexing](#data-indexing)

Data indexing is the foundational step where raw data is transformed into a structured format that facilitates efficient retrieval. This process involves:

- **Data preprocessing:** Raw data is cleaned and normalized to ensure consistency and quality.

- **Embedding generation:** Preprocessed data is converted into vector embeddings using embedding models. These embeddings capture the semantic essence of the data, allowing for meaningful comparisons.

- **Vector storage:** The generated embeddings are stored in a vector database, which is optimized for rapid similarity searches.

Efficient data indexing ensures that the retriever can quickly access pertinent information, thereby enhancing the system's responsiveness.

### [Retriever Mechanism](#retriever-mechanism)

The retriever mechanism is responsible for fetching relevant documents or data points in response to a user's query. This involves:

- **Query embedding:** The user's query is transformed into a vector embedding using the same embedding model employed during data indexing.

- **Similarity search:** The query embedding is compared against the stored embeddings in the vector database to identify the most relevant documents. Techniques such as cosine similarity are commonly used for this purpose.

- **Document ranking:** The retrieved documents are ranked based on their relevance scores to prioritize the most pertinent information.

An effective retrieval mechanism enables the system to access and use the most relevant external information, which improves the quality of the generated responses.

### [Augmentation Process](#augmentation-process)

Once relevant documents are retrieved, they are combined with the user's original query to form an augmented input. This process includes:

- **Contextual integration:** The content of the retrieved documents is integrated with the user's query to provide comprehensive context.

- **Prompt engineering:** The combined input is structured into a prompt format that effectively guides the LLM in generating a response.

The augmentation process ensures that the LLM has access to external knowledge, enabling it to produce accurate and contextually enriched responses.

### [Generator Module](#generator-module)

The generator module utilizes the augmented prompt to produce the final response. This involves:

- **Language modeling:** The LLM (text-to-text generation system) processes the augmented prompt, leveraging its internal knowledge and the provided context to generate coherent and contextually appropriate text.

- **Response generation:** The model crafts a response that addresses the user's query, incorporating information from both its training data and the retrieved external documents.

## [Technical Considerations for RAG-based Systems](#technical-considerations-for-rag-based-systems)

Several technical aspects must be meticulously addressed to ensure efficiency, accuracy, and scalability. Key considerations include parsing techniques, chunking strategies, embedding models, indexing approaches, and retrieval methods.

### [Parsing Techniques](#parsing-techniques)

Parsing is the initial step in processing documents within a RAG-based system. It involves extracting text from various file formats, such as PDFs. Common formats include Word documents and HTML files. Each format presents unique challenges. For example, PDFs may contain complex layouts, images, or embedded fonts that make text extraction more difficult.

Scanned documents introduce additional challenges because they require Optical Character Recognition (OCR) to convert text images into machine-readable formats. However, OCR processes can introduce errors due to factors like poor image quality or unusual fonts. Implementing preprocessing steps such as noise reduction, skew correction, and layout analysis can enhance OCR accuracy. Human-in-the-loop verification helps reduce OCR errors and improves the reliability of the extracted text for downstream processing.

### [Chunking Strategies](#chunking-strategies)

Once text is extracted, it must be segmented into manageable units or chunks to facilitate efficient retrieval and processing. Effective chunking balances context retention with system performance. Common strategies include:

- **Fixed-length chunking:** Divides text into equal-sized segments, which simplifies implementation but may disrupt semantic units, leading to a potential loss of meaning.

- **Semantic chunking:** Segments text based on natural language structures such as sentences or paragraphs, preserving semantic integrity. This approach often yields better retrieval results, as it maintains the context within each chunk.

- **Overlapping chunking:** Introduces overlaps between consecutive chunks to ensure that contextual information is retained across boundaries. This technique is particularly useful when the exact query context is unpredictable, as it increases the likelihood that relevant information is captured within at least one chunk.

Selecting an appropriate chunking strategy is crucial, as it directly impacts the system's ability to retrieve relevant information and generate accurate responses.

> **Q. How does chunking impact retrieval performance in RAG?**
>
> Chunking affects both retrieval accuracy and latency:
> - **Smaller chunks:** More retrieval calls, better granularity, but risks losing global context.
> - **Larger chunks:** Fewer retrieval calls, better context retention, but may introduce irrelevant information.
>
> An optimal strategy involves hybrid chunking, where chunk size varies based on document structure and query intent. For example, if a query needs fine-grained details (e.g., a specific formula in a research paper), smaller semantic chunks are preferable. If the query is broad (e.g., "explain deep learning"), larger contextual chunks work better.

### [Embedding Generation](#embedding-generation)

Embedding models transform textual chunks into vector representations that capture semantic meanings, enabling efficient similarity searches. Choosing the right embedding model is vital and depends on factors such as the domain specificity of the data and the desired balance between performance and computational resources.

Models like BERT or GPT generate embeddings applicable across various domains. However, they may lack the nuance required for specialized fields. The choice of embedding model affects the system's ability to represent polysemy and context-dependent meanings, which directly impacts the accuracy of retrieval and generation.

These embeddings are then stored in data stores. Vector databases, designed to handle high-dimensional data, are commonly used in RAG systems. When selecting a vector database, considerations include:

- **Scalability:** The database should accommodate growth in data volume without significant performance degradation.

- **Query performance:** Low-latency retrieval is critical for real-time applications.

Popular vector databases like Pinecone, Milvus, and Weaviate offer diverse features catering to different scalability and performance requirements. For instance, Pinecone provides a fully managed service with robust scalability, while Milvus offers open-source flexibility with high-performance retrieval capabilities.

> **Q. What happens if embeddings are generated using different models for retrieval and query processing?**
>
> This leads to embedding mismatch, reducing retrieval accuracy.
>
> If the query embedding model differs from the document embedding model, similarity calculations become unreliable.
>
> Even slight differences (e.g., BERT vs. RoBERTa embeddings) can degrade performance.
>
> The best practice is to use a consistent embedding model for both indexing and query processing.

### [Retrieval Methods](#retrieval-methods)

In RAG-based systems, the retrieval component is critical in identifying and fetching documents that are most relevant to a user's query. The efficacy of this component significantly influences the overall performance and accuracy of the system. Key aspects of retrieval methods include the selection of similarity metrics, the implementation of search algorithms, and the integration of advanced retrieval techniques such as hybrid search strategies.

Similarity metrics are fundamental in determining the closeness between the query embeddings and document embeddings. Commonly employed metrics include:

- **Cosine similarity:** Measures the cosine of the angle between two non-zero vectors, effectively capturing the orientation and, thus, the similarity in context between the query and document embeddings. This metric is particularly useful in high-dimensional spaces where the magnitude of vectors is less informative than their direction.

- **Euclidean distance:** Calculates the straight-line distance between two points in a multidimensional space. While intuitive, Euclidean distance can be less effective in high-dimensional settings due to the "curse of dimensionality," where distances between points become less discriminative.

- **Dot product:** Assesses similarity by computing the dot product of two vectors. This metric is computationally efficient and often used in large-scale retrieval tasks.

The choice of similarity metric can significantly impact retrieval performance. For instance, cosine similarity is often preferred in text retrieval tasks due to its effectiveness in capturing semantic similarities, whereas Euclidean distance might be more applicable in other contexts where spatial proximity is a priority.

RAG systems are vulnerable to adversarial attacks that involve injecting misleading or manipulated documents into the retrieval index. Common attack types include:

- **Keyword stuffing:** Flooding indexed documents with popular query keywords to manipulate rankings.

- **Embedding collisions:** Crafting text with embeddings that intentionally resemble high-quality documents but contain false information.

- **Contextual poisoning:** Adding documents that are partially correct but subtly misleading, making fact-checking harder.

We need to defend against them with anomaly detection mechanisms during retrieval.

To further enhance retrieval accuracy and robustness, hybrid search strategies combine the strengths of sparse (keyword-based) and dense (vector-based) retrieval methods:

- **Keyword-based search (sparse retrieval):** Relies on traditional information retrieval techniques, such as term frequency-inverse document frequency (TF-IDF) and BM25, to match query terms with documents. While effective for exact term matching, this approach may struggle with capturing semantic similarities.

- **Vector-based search (dense retrieval):** This method utilizes embeddings to capture the semantic content of queries and documents, enabling the retrieval of conceptually related information even when exact terms do not match. This method excels in understanding context but may retrieve less relevant documents if not properly tuned.

> **Q. What are the trade-offs between dense and sparse embeddings for retrieval?**
>
> When choosing between dense and sparse embeddings, we need to consider the following trade-offs:
>
> | Feature | Dense Embeddings (e.g., BERT, GPT) | Sparse Embeddings (e.g., TF-IDF, BM25) |
> | --- | --- | --- |
> | Semantic Understanding | High (captures meaning) | Low (relies on word matching) |
> | Exact Match Performance | Low (may retrieve loosely related docs) | High (retrieves exact terms) |
> | Computational Cost | High (requires vector search) | Low (simple term matching) |
> | Interpretability | Low (hard to explain results) | High (easy to trace matches) |
>
> A common approach is to use hybrid retrieval (dense and sparse embeddings) because it combines the strengths of both approaches.

Incorporating these advanced retrieval methods into RAG-based systems improves the retrieved documents' relevance and accuracy and contributes to the generation of more informed and contextually appropriate responses. As the field evolves, ongoing research and development in similarity metrics, search algorithms, and hybrid strategies continue to refine and enhance the capabilities of retrieval components in RAG-based systems.

## [Evaluation in RAG-based Systems](#evaluation-in-rag-based-systems)

Evaluating RAG-based systems is essential to ensure their effectiveness in retrieving relevant information and generating accurate, coherent responses. A comprehensive evaluation encompasses both the retrieval and generation components, utilizing specific metrics and frameworks to assess performance.

### [Evaluating the Retrieval Component](#evaluating-the-retrieval-component)

The retrieval component's primary function is to fetch pertinent documents or data segments that inform the generation process. Key metrics for assessing retrieval performance include:

- **Precision:** Measures the proportion of retrieved documents that are relevant to the query.

- **Recall:** Assesses the proportion of relevant documents successfully retrieved from the total available relevant documents.

- **Mean reciprocal rank (MRR):** Evaluates the rank position of the first relevant document within the retrieved results, emphasizing the importance of placing relevant documents higher in the list.

- **Mean average precision (MAP):** Calculates the mean of the average precision scores for all queries, providing a single-figure measure of quality across multiple queries.

Implementing these metrics allows for a detailed analysis of the retrieval system's ability to select and rank relevant documents effectively.

### [Evaluating the Generation Component](#evaluating-the-generation-component)

The generation component produces responses using the retrieved information. Evaluating this component involves assessing:

- **Faithfulness:** Determines whether the generated content accurately reflects the information found in the retrieved documents, ensuring that the response is based on factual data.

- **Answer relevance:** Measures how well the generated response addresses the user's query, focusing on the pertinence and usefulness of the information provided.

Metrics, such as BLEU, ROUGE, and others like METEOR (metric for evaluation of translation with explicit ordering), are commonly used to assess the quality of the generated text.

## [Implementing a RAG-based System with File Inputs](#implementing-a-rag-based-system-with-file-inputs)

Consider a RAG-powered document assistant designed to help users query a repository of corporate PDF files containing product information, specifications, and usage guidelines. Users can input natural language questions, and the system retrieves relevant information from these documents to generate accurate and contextually appropriate responses. This application is particularly beneficial for customer support, enabling quick access to detailed product information. Let's take a brief look at the technical considerations for this system:

1. We will be importing documents into the system and using text extraction and OCR to extract data from the PDF files. Using OCR increases processing time and may introduce errors due to misinterpretation of characters. However, it enables the inclusion of scanned documents that would otherwise be inaccessible.

2. We can then use fixed-length chunking for the sake of simplicity. The trade-off there is that smaller chunks enhance retrieval precision but may lose context, while larger chunks retain context but can reduce retrieval accuracy.

3. We can pick any publicly available and open-source embedding model to generate our vector embeddings and store them in a vector database.

4. When a user submits a query, we will convert it into an embedding and perform a similarity search to retrieve relevant document chunks.

5. We will then combine retrieved document chunks with the user query to form comprehensive context and design prompts that effectively guide the language model to generate accurate and contextually relevant responses.

## [Resource Estimation](#resource-estimation)

Before designing a system, it is important to understand how much storage, compute, and network capacity it will actually need. The estimates below are based on a system serving 100 million daily active users (DAUs), each submitting 10 queries per day.

### [Storage Estimation](#storage-estimation)

Storage in a RAG system falls into four categories:

- **Model weights:** A 3-billion-parameter LLM stored in FP16 precision occupies approximately 6 GB. This is a fixed, one-time cost.

- **Retrieval corpus:** The document store that the retrieval system searches over is estimated at 10 TB. This grows only when new documents are added.

- **User profile data:** At roughly 10 KB per user profile, 100 million users require 1 TB of storage.

- **Interaction data:** This is the dominant and growing cost. Each query produces around 100 KB of data (query history, retrieved passages, generated response, logs, and metadata). With 10 queries per user per day:

$$\text{Daily interaction data} = 100\text{M users} \times 10\text{ queries} \times 100\text{ KB} = 100\text{ TB/day}$$

$$\text{Monthly interaction data} = 100\text{ TB} \times 30 = 3\text{ PB/month}$$

> **Note:** Model weights, the retrieval corpus, and user profiles are mostly static. Interaction data scales with usage and drives storage requirements.

### [Inference Servers Estimation](#inference-servers-estimation)

For 100 million daily users submitting 10 queries each, the system must handle approximately 11,574 requests per second (RPS). This is calculated as:

$$\text{RPS} = \frac{100\text{M} \times 10}{86400\text{ seconds}} \approx 11{,}574\text{ RPS}$$

To serve this load with low latency, the system requires 112 high-performance GPU servers.

### [Bandwidth Estimation](#bandwidth-estimation)

To calculate the ingress and egress network bandwidth for 11,574 RPS, we assume the single request size is approximately 2 KB. Also, as a result, the response size for each request is approximately 10 KB. According to the BOTECs, these assumptions give us the following bandwidths:

$$\text{Ingress bandwidth} = 11{,}574 \times 2\text{ KB} = 23.15\text{ MBps} = 23.15\text{ MBps} \times 8 \approx 185\text{ Mbps}$$

$$\text{Egress bandwidth} = 11{,}574 \times 10\text{ KB} = 115.74\text{ MBps} = 115.74\text{ MBps} \times 8 \approx 926\text{ Mbps}$$

The table below consolidates all estimated resources required for the system:

| Resource | Estimate |
| --- | --- |
| Total Storage | 3 PB / month (interaction data) |
| Inference Servers | 112 high-performance GPU servers |
| Ingress Bandwidth | 185 Mbps |
| Egress Bandwidth | 926 Mbps |

## [High-level Design](#high-level-design)

A RAG system integrates two core capabilities: retrieval of relevant information from a knowledge store and generation of a coherent natural language response using that information. The architecture keeps these two responsibilities clearly separated, which makes the system easier to scale, debug, and improve independently.

The system is organized into three major subsystems:

- **Query understanding and embedding system:** Converts the user's raw query into a vector representation that captures its meaning, so that semantically similar documents can be found.

- **Knowledge management system:** Handles how documents are ingested, embedded, indexed, and retrieved in response to a query vector.

- **Response generation and quality control system:** Takes the retrieved documents and the original query, feeds them into an LLM, and ensures the final output is accurate, well-formatted, and appropriate.

### [Achieving Functional Requirements](#achieving-functional-requirements)

It is useful to connect each subsystem to a concrete functional requirement:

| Functional Requirement | Subsystem Responsible |
| --- | --- |
| Understanding user query intent | Query understanding and embedding system |
| Ingesting and indexing documents | Knowledge management system |
| Retrieving relevant knowledge | Knowledge management system |
| Generating grounded, accurate responses | Response generation and quality control system |

## [Detailed System Design](#detailed-system-design)

Now, let's go deeper into each subsystem and the individual services that make it up.

### [1. Query Understanding and Embedding System](#1-query-understanding-and-embedding-system)

This subsystem transforms a user query into a vector embedding that captures the query's intent. This vector is then used to search the knowledge base for semantically relevant documents.

- **Query preprocessing service:** Before encoding a query, it must be cleaned and normalized. This service handles tokenization, whitespace removal, spelling correction, and synonym expansion. Clean input leads to better embeddings. Query logs and input patterns are stored in a relational or document database for analysis and improvement over time.

- **Query encoder service:** This service passes the cleaned query through a dense language model, such as SBERT or MiniLM, to produce a fixed-size vector embedding. It supports batching for throughput and uses quantized model variants to reduce memory consumption and improve latency.

- **Embedding cache:** Frequently repeated queries produce the same embedding every time. Instead of re-encoding, the cache stores and retrieves precomputed embeddings, reducing latency and computational cost.

> **Flow:** Once the query embedding is ready, it is passed to the knowledge management system, which uses it to search the indexed document store.

### [2. Knowledge Management System](#2-knowledge-management-system)

This subsystem manages the document knowledge base, including file ingestion, vector encoding, and efficient retrieval of relevant passages at query time.

- **Document ingestion service:** This service accepts documents in various formats (PDF, DOCX, HTML, JSON), extracts and cleans the raw text, and splits it into semantically meaningful chunks. Chunking is typically based on paragraphs, sections, or token count, ensuring each chunk fits within the LLM's context window without losing meaning.

- **Document embedding service:** Each text chunk is passed through a transformer model to produce a vector embedding. GPU or quantized inference is used for speed. Each vector is stored alongside the original text and its metadata (source, date, author) to support accurate retrieval and attribution.

- **Document indexing and search service:** This service stores all document embeddings in a vector database such as Faiss or Pinecone and handles the retrieval search at query time. When a query embedding is received, the service retrieves the most similar document vectors using cosine similarity or inner product search. The system can also combine semantic vector search with keyword-based search to improve recall, followed by ranking and returning the top matching passages.

- **Metadata filter service:** Not every use case should search the entire corpus. This service applies pre-retrieval filters, by date, author, domain, or source type, to narrow the search space before vector similarity is computed. This is especially useful for applications that require recency (e.g., only articles from the past month) or source trust (e.g., only verified documents).

The table below summarizes the storage technologies used in this subsystem:

| Component | Storage Technology | Purpose |
| --- | --- | --- |
| Document Ingestion Service | Document DB | Stores raw documents and parsed text chunks with metadata |
| Document Embedding Service | Vector DB + Metadata DB | Stores embeddings linked to the source text and metadata |
| Document Indexing and Search Service | Vector DB + Metadata DB | Supports fast semantic and keyword search over embeddings |
| Metadata Filter Service | Document DB or Graph DB | Stores structured metadata for filtering search results |

### [3. Response Generation and Quality Control System](#3-response-generation-and-quality-control-system)

Once the relevant documents are retrieved, this subsystem assembles the final response. It combines the user query and the retrieved context, feeds it to the LLM, and then applies quality controls before returning the output to the user.

- **Context assembler:** This service builds the input prompt for the LLM by combining the user query with the top-retrieved passages. A typical prompt structure looks like this:
  ```
  Query: <user_question>\nContext: <passage_1>\n<passage_2>\n...
  ```
  This is usually stateless and operates in memory (e.g., Redis as a transient store), since there is no need to persist the assembled prompt.

- **Prompt customization service:** Different deployments, healthcare, legal, and customer service require different response styles. This service injects domain-specific instructions or user preferences into the prompt to control tone, structure, or content boundaries. Prompt templates and user settings are stored in a key-value or document store.

- **Response generator:** The LLM runs here, on GPU hardware, consuming the assembled and customized prompt to produce the final answer. The service exposes configurable parameters such as temperature (controls randomness vs. determinism), maximum token limits, and streaming output for low-latency responses. It logs outputs, usage statistics, and latency metrics to a time-series or analytics database.

- **Quality control service:** The generated response is reviewed before it reaches the user. This service performs three functions: it corrects grammar and applies formatting (headings, bullet points) using lightweight NLP tools; it checks for hallucinations, policy violations, or inappropriate content using moderation rules and optionally fact-verification APIs; and it collects user feedback (accept/reject, ratings) to support fine-tuning of the system over time. Moderation policies and feedback are stored in databases like PostgreSQL or MongoDB.

The table below summarizes the storage technologies used in this subsystem:

| Component | Storage Technology | Purpose |
| --- | --- | --- |
| Context Assembler | Transient in-memory store (e.g., Redis) | Holds prompt data temporarily; stateless between requests |
| Prompt Customization Service | Key-value store or Document DB | Stores prompt templates, domain rules, and user preferences |
| Response Generator | Logging DB + Time-series DB | Logs model outputs, latency, and usage metrics |
| Quality Control Service | PostgreSQL / MongoDB | Stores moderation policies, formatting rules, and user feedback |

### [Achieving Nonfunctional Requirements](#achieving-nonfunctional-requirements)

Let's see how the RAG system's architecture addresses each non-functional requirement:

| Nonfunctional Requirement | How the RAG System Achieves It |
| --- | --- |
| Low latency | The embedding cache avoids re-encoding repeated queries, the vector database enables fast similarity search, and the context assembler uses transient in-memory stores like Redis to build prompts on the fly without hitting disk. |
| Scalability | The modular architecture allows each subsystem (retrieval, embedding, and generation) to scale independently based on system load. GPU inference servers are scaled horizontally to handle increased request volume. |
| High availability | Each subsystem is a separate service, so no single component is a system-wide bottleneck. Redundant GPU servers and replicated vector databases ensure the system stays up even if individual nodes fail. |
| Data freshness | The document ingestion service can accept and process new documents at any time. Once embedded and indexed, new content is immediately available for retrieval without restarting or re-indexing existing data. |
| Content safety | The quality control service scans every generated response for hallucinations, inappropriate language, and policy violations before it reaches the user. Moderation rules and feedback are stored and continuously refined. |
| Fault tolerance | As each subsystem is decoupled, a failure in one component (such as the prompt customization service or metadata filter) does not cause a system-wide failure. The system can fall back to a degraded mode, such as skipping metadata filtering, while the failed component recovers. |

## [Conclusion](#conclusion)

A RAG system provides capabilities that neither retrieval nor generation can achieve independently. It grounds language model responses in an up-to-date, domain-specific knowledge base, which improves factual accuracy and relevance.

The architecture we explored separates concerns clearly across three subsystems: the query embedding pipeline, the knowledge management layer, and the response generation and quality control layer. Each subsystem uses purpose-appropriate storage technologies and can scale independently.

