# [Design: Payment System](#design-payment-system)

## [Introduction to the Payment System](#introduction-to-the-payment-system)

### [What is a digital payment system?](#what-is-a-digital-payment-system)

A payment system handles the transfer of funds for purchases. It supports multiple methods, including credit cards, bank transfers, digital wallets, and cryptocurrencies. These systems ensure the secure transfer of funds between parties. This design focuses on payments made through credit and debit cards. Before defining the architecture, it is important to understand the payment flow.

### [The payment process](#the-payment-process)

The payment process involves multiple entities. When a customer pays with a card, the system must verify the cardholder, process the payment, and validate the transaction.

<img width="921" height="760" alt="image" src="https://github.com/user-attachments/assets/d3aaf09d-d5af-4935-9cc3-f98bdbac6c86" />


The table below details the roles of different entities in the system:

| Entities | Description |
| --- | --- |
| Customer | A cardholder who wants to purchase goods or services. |
| Merchant | A company or person who provides goods or services. |
| Issuer bank | The bank that holds the customer's account and provides credit or debit cards. |
| Acquiring bank | The bank holding the merchant's account. |
| Merchant's online store | The online website where the customer purchases goods or services by providing credit card details. |
| Payment gateway | A network that facilitates and authorizes transactions of funds to merchants' accounts. Requests and receives payment from the customer's account. |
| Cards network | Validates the credit or debit card information. Facilitates payments via credit and debit cards. Sets the terms for credit or debit card transactions. |

Credit card transactions typically involve two phases: authorization and settlement. These phases verify the transaction and transfer funds from the customer to the merchant. The following is an overview of each phase:

### [The authorization phase](#the-authorization-phase)

The authorization phase's primary purpose is to verify whether the customer's payment method (e.g., credit card) is valid, has sufficient funds or credit available, and can be used to make the intended purchase.

1. The customer provides payment details to the merchant via a payment gateway.

2. The gateway requests payment authorization from the customer's issuer bank.

3. The issuer bank validates the card and checks for fraud and the available credit limit.

4. If approved, the issuer sends an authorization code to the gateway.

5. The gateway relays the code to the merchant, authorizing the transaction.

6. Successful authorization reserves the amount on the customer's card but does not immediately transfer funds.

### [The settlement phase](#the-settlement-phase)

Settlement finalizes the transaction by moving funds to the merchant's account.

1. The merchant accumulates authorized transactions over a period (e.g., daily).

2. The merchant sends a batch of transactions to the payment processor or acquiring bank. (Note: Some gateways, like Stripe, also act as processors.)

3. The processor verifies transactions and transfers funds from issuing banks to the merchant's bank.

4. The processor reconciles the transactions to ensure accuracy.

5. Funds are deposited into the merchant's account, completing the process.

> **Note:** Authorization is typically immediate, while settlement often occurs in batches.

Once settled, the transaction appears on the customer's statement, and the payment is complete.

## [Requirements](#requirements)

We will define the functional and non-functional requirements.

### [Functional requirements](#functional-requirements)

The system will accept card payments via a PSP.

- **User registration and authentication:** Users should be able to create accounts, log in securely, and reset passwords.

- **Payment processing:** Users should be able to initiate payment processing upon purchasing a product or service. The system should accept payment via credit and debit cards.

- **Transaction history:** Users should be able to view a history of their transactions, including dates, amounts, and descriptions.

- **Balance management:** Users should be able to check their account balances, including available funds and pending transactions.

- **Mobile accessibility:** Users should be able to access and use the payment system through mobile apps or responsive web interfaces.

### [Non-functional requirements](#non-functional-requirements)

The system must meet the following operational standards:

- **Data integrity and security:** The system should comply with industry security standards. It should also have robust data encryption and secure storage mechanisms, maintain consistency, and prevent data corruption.

- **Availability:** The system should be highly available to ensure users can always access it.

- **Reliability:** Redundancy, failover mechanisms, and backup systems should be in place.

- **Scalability:** The system should scale seamlessly as the user base and transaction volume grow. It should also handle peak loads during events like promotions or sales.

- **Performance:** The system should handle high transaction loads without significant slowdowns. The response times for payment processing and account updates should also be within acceptable limits.

## [Resource Estimation](#resource-estimation)

We assume the system handles 50 million transactions per day. This averages ~2 million transactions per hour, rising to 5 million during peak times.

### [Storage Estimation](#storage-estimation)

We must store user details and transaction records. Assuming each transaction is 100 bytes and each user record is 200 bytes, we calculate the daily storage requirement:

$$\text{Storage required per day} = 50\text{M} \times 100\text{ bytes} + 200\text{ bytes} \approx 5\text{ GB}$$

> **Note:** User records are stored once; transactions reference the user ID.

### [Bandwidth Estimation](#bandwidth-estimation)

Bandwidth requirements depend on the amount of data moving in and out of the system during transactions.

**Incoming bandwidth**

With a peak load of 5 million transactions per hour, the system handles approximately 1,400 transactions per second (TPS). At 100 bytes per transaction, the incoming bandwidth is:

$$\text{Total incoming bandwidth} = \text{Total transaction/sec} \times \text{Total transaction size} = 1400 \times 100\text{ Bytes} \approx 1.12\text{ Mbps}$$

**Outgoing bandwidth**

Responses contain status codes, tokens, and metadata. We assume an average size of 130 bytes per response.

$$\text{Total outgoing bandwidth} = \text{Total response/sec} \times \text{Total response size} = 1400 \times 130\text{ Bytes} \approx 1.46\text{ Mbps}$$

### [Number of Servers Estimation](#number-of-servers-estimation)

To estimate the server count, we use the peak traffic load. Given our assumption that daily active users serve as a proxy for requests per second, we get 50 million requests per second. Then, we use the following formula to calculate the number of servers:

$$\text{Servers needed at peak load} = \frac{\text{Number of requests/second}}{\text{RPS of server}}$$

Assuming a single server can handle 64,000 RPS, the estimation is:

$$\text{Servers needed at peak load} = \frac{50\text{ million}}{64{,}000} \approx 782\text{ servers}$$

## [Building Blocks We Will Use](#building-blocks-we-will-use)

The payment system requires specific components, including essential services such as payment gateways, risk checks, and reconciliation systems.

<img width="438" height="236" alt="image" src="https://github.com/user-attachments/assets/3165346a-5590-471a-a8bc-64565eb36539" />


- **Databases:** Store user payment events and transaction data.

- **Load balancers:** Distribute incoming requests across available servers.

- **Pub-sub systems:** Decouple services like payment processing, wallets, and ledgers.

## [High-level Design](#high-level-design)

<img width="1053" height="436" alt="image" src="https://github.com/user-attachments/assets/20fcfa9a-64ba-4783-b4d4-c1b52395c380" />


1. A customer clicks the pay button on a merchant's website, which triggers the payment service.

2. The payment service processes the customer's information and sends the transaction details to the risk check system for fraud detection.

3. If the transaction passes the risk check, the payment service forwards the request to the payment gateway.

4. The payment gateway validates the payment details and forwards the request to the card issuer's bank.

5. The issuer's bank processes the request and sends the payment to the merchant's account via the payment service.

6. The merchant's account balance is updated to reflect the successful transaction.

## [API Design](#api-design)

The following APIs are essential to meet our functional requirements.

### [User Registration and Authentication](#user-registration-and-authentication)

**User registration:** This API handles new user registration.

> **`POST /v1/users/register`**

```
registerUser(username, email, password)
```

The `registerUser` API hashes the user's password before saving it to the database. The table below explains the API's parameters.

| Parameter | Description |
| --- | --- |
| `username` | A unique user name opted for by the customer. |
| `email` | Customer's email id to be used later in the authentication phase. |
| `password` | The customer's password is used for authentication later. |

**User authentication:** This API authenticates users. We will assume a basic authentication mechanism.

> **`POST /v1/users/authenticate`**

```
authenticateUser(email, password)
```

### [Payment Processing](#payment-processing)

The following APIs handle payment processing tasks.

**Payment authorization:** This API verifies that the customer has sufficient funds to complete the payment. If successful, it associates the transaction with the merchant and returns an authorization code.

> **`POST /v1/payments/authorize`**

```
authorizePayment(amount, card_number, expiration_date, CVV, merchant_id)
```

The table below describes the API's parameters.

| Parameter | Description |
| --- | --- |
| `amount` | The amount to be paid by the customer for a purchase. |
| `card_number` | 16-digit customer's payment card number. |
| `expiration_date` | The expiry date of the payment card. |
| `CVV` | A 3-digit card verification value of the payment card. |
| `merchant_id` | A unique identifier for the merchant receiving the payment. Used to route the transaction and reserve funds for the correct merchant account. |

**Payment capture:** This API captures previously authorized funds, completing the transaction and transferring the money to the merchant's account.

> **`POST /v1/payments/capture`**

```
capturePayment(authorization_code, amount)
```

| Parameter | Description |
| --- | --- |
| `authorization_code` | The authorization code provided by the `authorizePayment` API upon successful verification. |
| `amount` | The amount to be deducted from the customer's account and transferred to the merchant's account. |

**Verify payment status:** This API checks the status of a transaction to confirm if it was successful or declined.

> **`GET /v1/payments/{transaction_id}/status`**

```
verifyPaymentStatus(transaction_id)
```

**Transaction history:** This API returns a customer's transaction history during the specified date range.

> **`GET /v1/transactions`**

```
getTransactionHistory(customer_id, start_date, end_date)
```

| Parameter | Description |
| --- | --- |
| `customer_id` | The customer's ID is the one who has requested transaction history. |
| `start_date` | The start date of the transaction is to be displayed to a customer. |
| `end_date` | The end date of the transaction to be displayed to a customer. |

## [Storage Schema](#storage-schema)

Each of the above APIs requires database support. Let's understand how the payment system stores different types of information:

<img width="1252" height="938" alt="image" src="https://github.com/user-attachments/assets/40ffe52c-bb20-49b5-8abb-8578cf440d0e" />



## [Detailed Design](#detailed-design)

The process begins when a customer provides their payment details on a merchant's site. These details include the card number, cardholder name, CVV/CVC, and expiration date. Before payment, the user authentication service verifies the customer's identity. Once authenticated, clicking the pay button triggers the payment service.

<img width="1665" height="839" alt="image" src="https://github.com/user-attachments/assets/7bfd6394-3395-4743-b9d7-de8bf8bd7fe5" />


Let's examine each component in the System Design:

- **Payment service:** Receives the payment event from the merchant and stores it in the associated database, performs initial security checks, and forwards the details to the fraud detection service.

- **Fraud detection service:** Analyzes transaction patterns in real time to identify and flag suspicious activity. It works with the risk check system to block high-risk payments without flagging legitimate ones.

- **Risk check system:** Works with the fraud detection service to assess a transaction's risk level. It evaluates factors such as device fingerprinting, transaction history, and location to assign a risk score that determines whether the payment should proceed, be challenged, or be blocked.

> **Q. If a fraud detection system is already in place, what additional value does a dedicated risk check system provide?**
>
> A fraud detection system focuses on identifying and blocking malicious or unauthorized activity, whereas a risk assessment system evaluates the overall risk profile of a transaction or user, including cases where no fraud is detected. This layered approach mitigates fraud as well as financial, operational, and reputational risks, providing broader protection across the payment system.

- **Payment gateway (payment service provider):** The payment gateway securely transfers funds from the customer's account to the merchant's account. It performs extensive security checks, often handled by a third-party specialist. As a Payment Service Provider (PSP), it may also offer secondary services like session management, refunds, and invoice generation.

- **Card networks:** Networks such as Visa and Mastercard, verify credit and debit card information. The payment gateway interacts with these networks via their APIs.

- **Wallet:** When a payment is successful, the payment service updates the merchant's wallet in the database to reflect the new balance. For purchases involving multiple sellers, the service processes each order separately and updates each seller's wallet.

- **Ledger:** After updating the wallet, the ledger service appends a new, immutable entry to the ledger database. This entry includes details like the transaction ID, amount, and parties involved, creating an auditable trail of all financial operations.

> **Q. Where are the customer's payment details encrypted during a purchase?**
>
> Customer's payment details in the system are encrypted at several key points:
>
> - **Client-side encryption:** Payment details are encrypted on the customer's device (e.g., computer or smartphone) before being transmitted over the internet.
> - **During data transmission:** Encrypted connections (SSL/TLS) secure the transmission of payment details between the customer and the payment system's servers.
> - **Payment gateway:** The payment gateway also encrypts payment details to facilitate customer and merchant transactions.
> - **Storage encryption:** Payment details may be temporarily stored and should be encrypted at rest for security. For example, when the payment service receives a payment event, it stores the event details in the database after applying the necessary encryption.
>
> Encryption is a vital security measure, complemented by access controls and compliance with industry standards like PCI DSS, to safeguard customer payment information.

- **Reconciliation system:** Reconciliation is the process of matching financial records to ensure accuracy and identify discrepancies. The system compares the internal ledger against transaction records from the payment gateway (or PSP). This is critical for catching errors and fraud. At the end of each day, the PSP sends a settlement file summarizing the day's transactions. The reconciliation system compares this file with the internal ledger. For example, if a merchant's ledger shows three payments totaling $225, the system verifies that the PSP's settlement file matches those payments. Any mismatch is flagged for investigation.

> **Q. What happens if a mismatch is found during reconciliation?**
>
> If a mismatch is found, the reconciliation system alerts and logs the discrepancy for further investigation. The issue could be missing transactions, duplicate entries, timing delays, or data corruption. The system may:
>
> - Temporarily hold settlements until the mismatch is resolved.
> - Trigger a reprocessing of affected transactions.
> - Notify the operations team or initiate automated correction workflows, depending on the severity.

- **Dispute management service:** This service manages disputes between customers, merchants, and banks. It handles chargebacks, ensures compliance with financial regulations, and maintains dispute records to prevent fraud.

### [Transaction Completion](#transaction-completion)

Within the payment system, multiple interconnected services collaborate to execute a transaction. In a distributed system, requests can be lost over the network, or a service might be temporarily unavailable.

<img width="883" height="810" alt="image" src="https://github.com/user-attachments/assets/da0e2eb9-cf38-4b7c-a829-224d63955a96" />


To ensure transaction completion, we can use a pub-sub system such as Apache Kafka. When a payment is initiated, we publish an event to a Kafka topic. Consumer services process these events, and a message is only marked as consumed after the transaction is successfully processed and recorded in the wallet and ledger databases. This ensures that no payment events are lost.

### [Transient Failures](#transient-failures)

A transient failure is a temporary error caused by issues like network glitches or brief resource shortages. Our system must handle these gracefully. We can use several strategies:

- Retry strategy
- Timeout
- Fallback mechanism

Let's examine how each of these strategies helps improve resilience in the payment system.

**Retry strategy**

The retry strategy helps overcome temporary network issues. We can implement exponential backoff and set a maximum number of retries. If the transaction still fails after reaching the limit, it is marked as failed.

**Timeout**

Timeouts prevent a service from waiting indefinitely for a response. If a response takes too long, the operation is terminated. However, this can create ambiguity. For example, if a customer's payment request times out, it's unclear what happened on the backend:

- The payment was processed successfully, but the response was lost on its way back to the client.

<img width="1231" height="591" alt="image" src="https://github.com/user-attachments/assets/f5ee694b-47af-4fe6-9f5e-880845c2f0b9" />

  
- The request is still being processed, but the client-side timeout was reached.

<img width="1100" height="561" alt="image" src="https://github.com/user-attachments/assets/6f97dd3a-ad31-4dd5-8b46-0a96c788f937" />

  
- The request was lost in transit and never reached the payment service.

<img width="1002" height="558" alt="image" src="https://github.com/user-attachments/assets/559e7e8d-2b69-4d14-ac52-1134c9ba40c9" />


In these scenarios, the client cannot know the true status of the payment. If the customer retries the operation, they could be charged twice. This problem is solved using idempotency. An idempotent API ensures that repeating the same request multiple times produces the same result as the initial request, preventing duplicate charges. This is typically implemented by having the client generate a unique request ID (an idempotency key) that the server uses to recognize and de-duplicate retried requests.

> **Q. How to ensure that there are no duplicate payments when a customer hits the buy button many times for a single purchase?**
>
> We can create a distinct transaction identifier (like an order number or payment ID) for every transaction or purchase, which serves as our idempotence key. This key is stored in our database when the customer initiates the payment and the request's status (for example, succeeded, failed, or in progress). If a customer accidentally clicks the "buy" button twice or generates a duplicate request for the same purchase, they won't be charged, as the idempotence key in the database prevents multiple charges.

**Fallback**

A fallback is a backup process used when a primary method fails. For example, if the fraud check service returns an error, instead of failing the entire payment, we can use a fallback. For a very small transaction, the fallback might be to approve the payment and accept the minor risk by using a fallback value. This approach balances risk against customer experience.

## [Achieving Requirements](#achieving-requirements)

Let's see how the proposed payment System Design satisfies both functional and non-functional requirements.

### [Achieving Functional Requirements](#achieving-functional-requirements)

The following table summarizes how the payment system fulfills the functional requirements:

| Functional Requirements | Techniques |
| --- | --- |
| User registration and authentication | Handled by the merchant's online store. It provides secure login, session management, and user identity validation. Authenticated requests forwarded to the payment service. |
| Payment processing | Payment processing is handled by the payment service in conjunction with other coordinated services, such as PSP, fraud detection services, and reconciliation services, among others. |
| Transaction history | The ledger stores immutable transaction logs, including details such as ID, amount, currency, involved parties, status, and timestamp. |
| Balance management | The wallet maintains up-to-date merchant balances by reflecting successful payments and refunds. The payment service updates the wallet after each transaction. Ledger provides an auditable balance history for verification. |
| Mobile accessibility | Enabled by a responsive Merchant's online store UI. |

### [Achieving Non-functional Requirements](#achieving-non-functional-requirements)

Beyond core features, a payment system must be secure, scalable, and resilient. The table below shows how our proposed design meets these essential non-functional requirements:

| Non-Functional Requirements | Techniques |
| --- | --- |
| Data integrity and security | Adhere to industry security standards such as PCI DSS (Payment Card Industry Data Security Standard). Use robust encryption and secure storage mechanisms. |
| Reliability and availability | Used redundant services (payment, reconciliation, risk check) to prevent failures. Replicate data across multiple data centers for high availability. Utilize local load balancers to exclude failed servers. Deploy global load balancers to redirect traffic during regional failures. |
| Scalability | Scale databases efficiently and cache frequently accessed data to optimize performance. Implement load balancing to prevent bottlenecks. Enable monitoring and automated scaling for dynamic resource allocation. |
| Performance | Process transactions asynchronously to decouple services. Ensure sufficient bandwidth during peak times. Implement transient failure strategies (retry, timeout, and fallback) to ensure timely responses. |

## [Conclusion](#conclusion)

This lesson describes a payment system that covers the process from authorization to settlement. The design addresses key functional and non-functional requirements with a focus on credit and debit card transactions. While the design covers core components, entities such as banks and merchants are external systems outside the system boundary. A payment system must balance security, reliability, and efficiency to maintain financial integrity and user trust.

