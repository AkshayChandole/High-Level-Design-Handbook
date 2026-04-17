# Maintainability

## What is Maintainability?

After building a system, engineers must keep it operational. This involves fixing bugs, adding features, updating platforms, and ensuring smooth operations. Maintainability measures how easily a system supports these tasks. We can divide maintainability into three aspects:

- **Operability:** The ease of ensuring smooth operations and recovering from faults.
- **Lucidity:** The simplicity of the codebase. A simpler codebase is easier to understand and maintain.
- **Modifiability:** The ability to integrate changes and new features without difficulty.

## Measuring Maintainability

Maintainability (**M**) is the probability that a service will restore its functions within a specified time after a fault. It measures how quickly the service returns to normal operating conditions. For example, if a component has a maintainability value of 95% for 30 minutes, the probability of restoring the component within that time is 0.95.

> **Note:** Maintainability provides insight into a system's ability to undergo repairs and modifications while in operation.

We use **MTTR** to measure **M**.

```
MTTR = Total Maintenance Time / Total Number of Repairs
```

MTTR is the average time required to repair and restore a failed component. The goal is to keep MTTR as low as possible.

## Maintainability and Reliability

Maintainability is closely related to reliability. The primary difference is the variable of interest: maintainability refers to **time-to-repair**, whereas reliability refers to **time-to-failure**. Combining maintainability and reliability analysis helps us understand availability, downtime, and uptime.
