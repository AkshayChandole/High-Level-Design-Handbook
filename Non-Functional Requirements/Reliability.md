# What is Reliability?

Reliability is the probability that a service will perform its intended functions for a specified period. It measures performance consistency under varying operating conditions.

We typically use **mean time between failures (MTBF)** and **mean time to repair (MTTR)** to measure reliability.

```
MTBF = (Total Elapsed Time - Sum of Downtime) / Total Number of Failures
```

```
MTTR = Total Maintenance Time / Total Number of Repairs
```

System designers strive for a **high MTBF** and a **low MTTR**.

Reliability and availability are often confused, but they measure different aspects of system performance. Reliability focuses on how consistently a service operates without failure, while availability measures how often it is accessible when needed. Understanding both is crucial because even a highly reliable system can have low availability if downtime or repairs take too long.

## Reliability and Availability

Reliability and availability are key metrics for measuring compliance with agreed-upon **service level objectives (SLO)**.

Availability is driven by time loss, whereas reliability is driven by the frequency and impact of failures. These metrics enable stakeholders to assess the overall health of the service.

Reliability (**R**) and availability (**A**) are distinct but related. Mathematically, **A** depends on **R** as one of several factors. Since **R** can change independently, we can encounter four scenarios:

| Scenario | Description |
|---|---|
| Low A, Low R | Frequent failures, slow recovery |
| Low A, High R | Rare failures, slow recovery |
| High A, Low R | Frequent failures, fast recovery |
| High A, High R | Desirable |



> **Note:** There are variations of the MTBF metric, such as **mean time to failure (MTTF)**. We use MTTF for non-repairable components that require replacement, such as a failed hard disk or light bulb.

---

### Q. What is the difference between reliability and availability?

<details>
<summary>Show Answer</summary>

Reliability measures how well a system performs its intended operations (functional requirements). We use averages for that (Mean Time to Failure, Mean Time to Repair, etc.)

Availability measures the percentage of time a system accepts requests and responds to clients.

**Example 1:** A certain system may be 90% available but only reliable 80% of the time.

**Example 2:** Suppose we consider our "system" the stuff inside a data center (hardware + software). Let's assume this data center suffers a network failure, so no outside traffic is coming in, and no inside traffic is going out. In this case, instantaneous availability might be zero (because clients cannot reach the service) even though all systems within the data center are perfectly functioning (instantaneous reliability 100%).

We use both of them (reliability and availability) in different contexts. For example, storage vendors often quote MTTF for their disks. Most online services use uptime (as a measure of availability) in their SLAs. For example, the uptime of EC2 virtual machines is 99.95%.

</details>
