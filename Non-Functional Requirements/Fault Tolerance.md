# Fault Tolerance

## What is Fault Tolerance?

Large-scale applications utilize hundreds of servers and databases to serve billions of users. To ensure data safety and avoid redoing computationally intensive tasks, these systems must eliminate single points of failure.

Fault tolerance is a system's ability to continue operating even if one or more components (software or hardware) fail. While achieving 100% fault tolerance is practically impossible, systems aim to maximize persistence and minimize disruption.

<img width="933" height="329" alt="image" src="https://github.com/user-attachments/assets/9c51a381-f79a-4301-9809-c6857aba94a2" />


Fault tolerance relies on two key qualities:

- **Availability:** The system remains accessible and receives client requests at any time.
- **Reliability:** The system consistently processes requests and performs the correct actions.

To prevent disruptions from a single point of failure, systems use two main approaches:

- **Fault-removal:** Uses forward or backward error recovery.
- **Fault-masking:** Using redundancy to prevent a fault from affecting the system's output.

Systems also implement failover strategies to manage downtime:

- **Hot failover:** Instantly transfers workloads to a functioning backup (zero downtime).
- **Warm or cold failover:** Loads and starts the backup only when needed. This causes a delay but consumes fewer resources.

> **Note:** Fault tolerance offers limited protection against software failures, which remain a major cause of outages.

<img width="510" height="459" alt="image" src="https://github.com/user-attachments/assets/bcf696a3-90c2-41e8-9e13-9fd46f389ebc" />


---

## Advantages of Fault-Tolerant Systems

The primary purpose of fault tolerance is to prevent system unavailability. This is critical for safety-critical systems (like air traffic control) and platforms requiring high data integrity. However, these systems are expensive to implement because they require redundant hardware and complex synchronization logic.

---

## Fault Tolerance Techniques

Failures occur at hardware or software levels. Common techniques to address these include:

### Forward and Backward Error Recovery

- **Forward error recovery** identifies and corrects the error state (e.g., exception handling in Ada or PL/1).
- **Backward error recovery** restores the system to the stable state that existed before the fault.

### Replication

Replication-based fault tolerance involves duplicating services and data. If a node fails, the system transparently swaps it with a healthy replica.

Updating data across replicas presents a trade-off between consistency and availability:

- **Synchronous updates:** Ensure strong consistency but reduce availability.
- **Asynchronous updates:** Improve availability but result in eventual consistency (stale reads).

This trade-off is central to the **CAP theorem**.

<img width="871" height="696" alt="image" src="https://github.com/user-attachments/assets/3b4633ff-aa72-4244-9b8a-a0dd6f54e518" />



### Checkpointing

Checkpointing periodically saves the system's state to stable storage. If a failure occurs, the system recovers by reloading the last saved state.

Checkpoints are classified based on the consistency of the global state:

- **Consistent state:** All processes share a coherent view of events. Completed updates are saved, in-progress updates are rolled back, and no messages are in transit.
- **Inconsistent state:** Checkpoints across processes are uncoordinated, leading to discrepancies.

Consider three processes (`i`, `j`, `k`) exchanging messages (`m1`, `m2`). Each process saves a snapshot (`C1,i`, `C1,j`, `C1,k`).

<img width="1354" height="526" alt="image" src="https://github.com/user-attachments/assets/3c024d33-4ed0-4f1e-854f-292c27149301" />


- In the **consistent state**, message `m1` is sent and received after the checkpoints are taken. The timeline is coherent.
- In the **inconsistent state**, process `i` records receiving `m1`, but process `j` has not recorded sending it. This discrepancy creates an inconsistent state.

The consistent state occurs because no messages are exchanged between processes during checkpointing. In the inconsistent case, processes exchange messages during checkpointing, which can lead to an inconsistent global state.
